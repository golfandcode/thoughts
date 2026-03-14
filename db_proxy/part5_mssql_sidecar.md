# Part 5: SQL Server Sidecar — Design Document

> **Component:** Python gRPC sidecar for SQL Server query execution  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md), [Part 2A: Credential Architecture](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part2a_credential_architecture.md)  
> **Key challenge:** No native Arrow/Parquet output — sidecar must serialize

---

## 1. The SQL Server Problem

SQL Server differs from PostgreSQL and ClickHouse in critical ways:
- **No native Arrow/Parquet output** — TDS protocol returns row-oriented data
- **SQLAlchemy ORM adds 5–10× CPU overhead** vs raw drivers (see benchmarks)
- **TDS protocol is opaque** — proxy-level inspection is not feasible
- The sidecar bears full responsibility for row → columnar serialization

---

## 2. Tiered Driver Strategy

```
┌─────────────────────────────────────────────┐
│ Tier 1 (Default): connectorx                │
│  Rust → ODBC → Arrow Tables, auto-parallel  │
│  Best: bulk SELECT, bounded results <2GB    │
├─────────────────────────────────────────────┤
│ Tier 2 (Streaming): turbodbc                │
│  ODBC columnar API → fetcharrowbatches()    │
│  Best: large results >2GB, parameterized    │
├─────────────────────────────────────────────┤
│ Tier 3 (Fallback): pyodbc                   │
│  cursor.fetchmany() → pyarrow.array()       │
│  Best: edge-case dtypes (NTEXT, XML, etc.)  │
└─────────────────────────────────────────────┘
```

### Driver Benchmark Comparison

| Driver | Throughput | Arrow Native | Memory Model | CPU Cost |
|---|---|---|---|---|
| **connectorx** | ~12K rows/ms | ✅ Direct | Full materialization | 🟢 Low (Rust) |
| **turbodbc** | ~8K rows/ms | ✅ `fetcharrowbatches()` | Streaming batches | 🟢 Low (C++) |
| **pyodbc** | ~2K rows/ms | ❌ Manual conversion | Streaming rows | 🟡 Medium |
| **SQLAlchemy ORM** | ~500 rows/ms | ❌ None | Full + ORM objects | 🔴 Very High |

> [!IMPORTANT]
> **SQLAlchemy is explicitly rejected.** The gateway does not construct SQL, map objects, or track changes. Every capability SQLAlchemy provides is unnecessary overhead. Use raw drivers only.

---

## 3. Implementation

```python
class MSSQLSidecar(DataGatewayServicer):
    async def QueryStream(self, request, context):
        md = dict(context.invocation_metadata())
        identity = md["x-client-identity"]
        trace_id = md.get("x-trace-id", str(uuid4()))
        max_rows = min(request.limits.max_rows or 1_000_000,
                       int(md.get("x-max-rows-ceiling", "1000000")))
        timeout_s = min(request.limits.timeout_seconds or 120,
                        int(md.get("x-max-timeout-ceiling", "120")))

        # Resolve per-user SQL Server credentials from Vault
        user, password = await self._resolve_credentials(identity, "mssql")

        traced_sql = f"/* trace_id={trace_id} */ {request.sql}"

        # Try Tier 1: connectorx
        try:
            async for chunk in self._connectorx_path(
                traced_sql, user, password, request.database, request.format
            ):
                yield chunk
            return
        except Exception as exc:
            logger.info(f"connectorx failed, falling back: {exc}")

        # Try Tier 2: turbodbc
        try:
            async for chunk in self._turbodbc_path(
                traced_sql, user, password, request.database,
                max_rows, timeout_s, request.format
            ):
                yield chunk
            return
        except Exception as exc:
            logger.warning(f"turbodbc failed, falling back to pyodbc: {exc}")

        # Tier 3: pyodbc fallback
        async for chunk in self._pyodbc_path(
            traced_sql, user, password, request.database,
            max_rows, timeout_s, request.format
        ):
            yield chunk
```

### Tier 1: connectorx

```python
async def _connectorx_path(self, sql, user, password, db, fmt):
    conn_str = (f"mssql://{user}:{password}@{MSSQL_HOST}:{MSSQL_PORT}"
                f"/{db}?ApplicationIntent=ReadOnly&TrustServerCertificate=yes")
    table: pa.Table = await asyncio.to_thread(
        cx.read_sql, conn_str, sql, return_type="arrow2"
    )
    for batch in table.to_batches(max_chunksize=50_000):
        yield QueryChunk(data=self._serialize(batch, fmt), is_last=False)
    yield QueryChunk(data=b"", is_last=True,
                     metadata=ResponseMetadata(rows_returned=table.num_rows))
```

### Tier 2: turbodbc (streaming)

```python
async def _turbodbc_path(self, sql, user, password, db, max_rows, timeout_s, fmt):
    odbc_str = (f"DRIVER={{ODBC Driver 18 for SQL Server}};"
                f"SERVER={MSSQL_HOST},{MSSQL_PORT};DATABASE={db};"
                f"UID={user};PWD={password};ApplicationIntent=ReadOnly")
    conn = await asyncio.to_thread(turbodbc.connect, connection_string=odbc_str)
    cursor = conn.cursor()
    await asyncio.to_thread(cursor.execute, sql)
    rows_sent = 0
    for batch in await asyncio.to_thread(cursor.fetcharrowbatches):
        yield QueryChunk(data=self._serialize(batch, fmt), is_last=False)
        rows_sent += batch.num_rows
        if rows_sent >= max_rows:
            break
    cursor.close(); conn.close()
    yield QueryChunk(data=b"", is_last=True,
                     metadata=ResponseMetadata(rows_returned=rows_sent))
```

---

## 4. Read-Only Enforcement

| Mechanism | Implementation |
|---|---|
| **Connection-level** | `ApplicationIntent=ReadOnly` in connection string |
| **SQL Server RBAC** | User mapped to `db_datareader` role only |
| **Resource Governor** | Workload group with `REQUEST_MAX_CPU_TIME_SEC=60`, `MAX_DOP=4` |
| **Sidecar validation** | Reject queries not starting with `SELECT`, `WITH`, or `EXPLAIN` |

---

## 5. Query Assassination

```python
async def _kill_query(self, trace_id: str):
    """Kill orphaned SQL Server query by SPID lookup."""
    async with self._admin_pool.acquire() as conn:
        spid = await asyncio.to_thread(conn.execute, f"""
            SELECT r.session_id FROM sys.dm_exec_requests r
            CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
            WHERE t.text LIKE '%trace_id={trace_id}%'
        """)
        if spid:
            await asyncio.to_thread(conn.execute, f"KILL {spid}")
```

---

## 6. Memory Budgeting

| Scenario | connectorx | turbodbc |
|---|---|---|
| 100K rows × 20 cols | ~200 MB peak | ~50 MB peak (batched) |
| 1M rows × 20 cols | ~2 GB peak | ~50 MB peak |
| 10M rows × 20 cols | ❌ OOM risk | ~50 MB peak |

> [!CAUTION]
> connectorx materializes the full result. For results >2 GB, the tier selection logic must route to turbodbc. Sidecar pod memory limit should be `max_concurrent_queries × 2 GB`.

---

## 7. Deployment

| Parameter | Value |
|---|---|
| Replicas | 3 |
| CPU | 4 vCPU (serialization-heavy) |
| Memory | 4 GB |
| gRPC port | 50053 |
| Dependencies | `connectorx`, `turbodbc`, `pyodbc`, `pyarrow`, `grpcio[aio]` |
| ODBC driver | `ODBC Driver 18 for SQL Server` |
