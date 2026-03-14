# Part 3: PostgreSQL Sidecar — Design Document

> **Component:** Python gRPC sidecar for PostgreSQL query execution and Arrow serialization  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md), [Part 2A: Credential Architecture](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part2a_credential_architecture.md)  
> **Receives traffic from:** [Part 1: Envoy Gateway](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part1_envoy_gateway.md)

---

## 1. Responsibilities

- Receive `QueryRequest` from Envoy, extract DB credentials from metadata
- Manage an `asyncpg` connection pool per unique `(db_user, database)` pair
- Enforce read-only, statement timeout, and row limits
- Execute query with injected trace ID comment
- Convert PostgreSQL row results → Arrow IPC RecordBatches in streaming fashion
- Support Parquet output via sidecar-side Arrow → Parquet compression
- Kill orphaned queries on client disconnect

---

## 2. Connection Pool Architecture

```python
class PgPoolManager:
    """Manages asyncpg pools keyed by (user, database) tuple."""

    def __init__(self, pg_host: str, pg_port: int):
        self._host = pg_host
        self._port = pg_port
        self._pools: dict[tuple[str, str], asyncpg.Pool] = {}
        self._lock = asyncio.Lock()

    async def get_pool(
        self, identity: str, database: str,
    ) -> asyncpg.Pool:
        """Get or create pool for a user identity + database combination."""
        # Lookup per-user DB credentials from Vault
        db_user, db_pass = await self._resolve_credentials(identity, "pg")
        key = (db_user, database)
        if key not in self._pools:
            async with self._lock:
                if key not in self._pools:
                    self._pools[key] = await asyncpg.create_pool(
                        host=self._host, port=self._port,
                        user=db_user, password=db_pass, database=database,
                        min_size=2, max_size=20,
                        max_inactive_connection_lifetime=300,
                        server_settings={
                            "default_transaction_read_only": "on",
                            "application_name": "gateway-pg-sidecar",
                        },
                    )
        return self._pools[key]
```

**Pool sizing:** `max_size=20` per (user, database). With 3 sidecar replicas × 20 = 60 max connections to Postgres, well within typical `max_connections=200`.

---

## 3. Query Execution Flow

```python
class PgSidecar(DataGatewayServicer):
    async def QueryStream(
        self, request: QueryRequest, context: grpc.aio.ServicerContext,
    ) -> AsyncIterator[QueryChunk]:
        # 1. Extract identity (injected by ext_authz from cert SAN)
        metadata = dict(context.invocation_metadata())
        identity = metadata["x-client-identity"]
        trace_id = metadata.get("x-trace-id", str(uuid4()))
        tier_max_rows = int(metadata.get("x-max-rows-ceiling", "1000000"))
        tier_max_timeout = int(metadata.get("x-max-timeout-ceiling", "120"))

        # 2. Enforce limit ceilings
        max_rows = min(request.limits.max_rows or tier_max_rows, tier_max_rows)
        timeout_s = min(request.limits.timeout_seconds or tier_max_timeout, tier_max_timeout)

        # 3. Validate SQL (reject writes)
        if not self._is_read_only(request.sql):
            context.abort(grpc.StatusCode.FAILED_PRECONDITION,
                          "Only SELECT statements are permitted")

        # 4. Get pool (sidecar resolves per-user DB creds from Vault)
        pool = await self._pool_mgr.get_pool(identity, request.database)
        async with pool.acquire() as conn:
            # 5. Set statement timeout
            await conn.execute(f"SET statement_timeout = {timeout_s * 1000}")

            # 6. Execute with trace ID
            traced_sql = f"/* trace_id={trace_id} api_key_hash={metadata.get('x-client-identity','')} */ {request.sql}"

            # 7. Stream in batches
            rows_sent = 0
            batch_rows: list[asyncpg.Record] = []
            schema: pa.Schema | None = None

            async with conn.transaction():
                async for record in conn.cursor(traced_sql):
                    batch_rows.append(record)

                    if len(batch_rows) >= BATCH_SIZE:
                        batch, schema = self._records_to_arrow(batch_rows, schema)
                        chunk_data = self._serialize(batch, request.format)
                        yield QueryChunk(data=chunk_data, is_last=False)
                        rows_sent += len(batch_rows)
                        batch_rows.clear()

                        if rows_sent >= max_rows:
                            break

            # 8. Final partial batch
            if batch_rows:
                batch, schema = self._records_to_arrow(batch_rows, schema)
                chunk_data = self._serialize(batch, request.format)
                yield QueryChunk(data=chunk_data, is_last=False)
                rows_sent += len(batch_rows)

            # 9. Terminal chunk with metadata
            yield QueryChunk(
                data=b"", is_last=True,
                metadata=ResponseMetadata(
                    rows_returned=rows_sent,
                    execution_time_ms=elapsed_ms,
                    query_id=trace_id,
                    truncated=(rows_sent >= max_rows),
                ),
            )
```

---

## 4. Row-to-Arrow Conversion

```python
BATCH_SIZE = 10_000  # Rows per Arrow RecordBatch

def _records_to_arrow(
    self, records: list[asyncpg.Record], schema: pa.Schema | None,
) -> tuple[pa.RecordBatch, pa.Schema]:
    """Convert asyncpg records to an Arrow RecordBatch."""
    if not records:
        raise ValueError("Empty batch")

    # Build schema from first batch if not yet known
    if schema is None:
        fields = []
        sample = records[0]
        for key in sample.keys():
            value = sample[key]
            arrow_type = PG_TYPE_MAP.get(type(value), pa.utf8())
            fields.append(pa.field(key, arrow_type))
        schema = pa.schema(fields)

    # Columnar conversion: transpose rows → columns
    columns: dict[str, list] = {f.name: [] for f in schema}
    for record in records:
        for key in columns:
            columns[key].append(record[key])

    arrays = [pa.array(columns[f.name], type=f.type) for f in schema]
    return pa.RecordBatch.from_arrays(arrays, schema=schema), schema
```

**Type mapping:**

| PostgreSQL Type | Arrow Type |
|---|---|
| `int2`, `int4` | `INT32` |
| `int8` | `INT64` |
| `float4` | `FLOAT32` |
| `float8` | `FLOAT64` |
| `numeric` | `DECIMAL128` |
| `bool` | `BOOL` |
| `text`, `varchar` | `UTF8` |
| `bytea` | `BINARY` |
| `date` | `DATE32` |
| `timestamp`, `timestamptz` | `TIMESTAMP_US` |
| `json`, `jsonb` | `UTF8` (serialized) |
| `uuid` | `UTF8` |
| `array` | `LIST` |

---

## 5. Alternative: pg_parquet Bulk Path

For large exports where server-side Parquet generation is preferred:

```python
async def _bulk_parquet_export(
    self, conn: asyncpg.Connection, sql: str, trace_id: str,
) -> AsyncIterator[bytes]:
    """Use pg_parquet extension for server-side Parquet generation."""
    # pg_parquet requires COPY TO with FORMAT 'parquet'
    parquet_sql = f"COPY (/* trace_id={trace_id} */ {sql}) TO STDOUT WITH (FORMAT 'parquet')"
    buffer = io.BytesIO()
    await conn.copy_from_query(parquet_sql, output=buffer)
    buffer.seek(0)
    # Stream in 1MB chunks
    while chunk := buffer.read(1_048_576):
        yield chunk
```

> [!WARNING]
> `pg_parquet` requires the extension to be installed on the PostgreSQL server. It also materializes the full result set before writing Parquet. Use only for known-bounded bulk exports, not for interactive queries.

---

## 6. Query Assassination on Disconnect

```python
async def _monitor_cancellation(
    self, context: grpc.aio.ServicerContext, conn: asyncpg.Connection,
    trace_id: str,
) -> None:
    """Background task: if client disconnects, kill the orphaned query."""
    try:
        await context.wait_for_disconnect()
    except asyncio.CancelledError:
        return  # Normal completion

    # Client disconnected — kill the running query
    admin_conn = await asyncpg.connect(
        host=self._pool_mgr._host, user=ADMIN_USER, password=ADMIN_PASS,
    )
    try:
        pid = await admin_conn.fetchval(
            "SELECT pid FROM pg_stat_activity WHERE query LIKE $1 AND state='active'",
            f"%trace_id={trace_id}%",
        )
        if pid:
            await admin_conn.execute("SELECT pg_terminate_backend($1)", pid)
            logger.warning(f"Killed orphaned PG query: pid={pid}, trace={trace_id}")
    finally:
        await admin_conn.close()
```

---

## 7. Deployment

| Parameter | Value |
|---|---|
| Replicas | 3 |
| CPU | 2 vCPU per pod |
| Memory | 2 GB per pod |
| gRPC port | 50051 |
| Dependencies | `asyncpg`, `pyarrow`, `grpcio[aio]` |
| Connection pool | 2–20 per (user, database) |
| Health check | gRPC health + `SELECT 1` to PG |
