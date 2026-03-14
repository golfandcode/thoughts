# Part 4: ClickHouse Sidecar — Design Document

> **Component:** Python gRPC sidecar for ClickHouse query execution via HTTP  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md), [Part 2A: Credential Architecture](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part2a_credential_architecture.md)  
> **Key advantage:** ClickHouse natively serializes to Arrow — sidecar CPU is near-zero

---

## 1. Responsibilities

- Receive `QueryRequest`, extract DB credentials from metadata
- Forward query to ClickHouse HTTP interface with constraint headers
- Stream native Arrow/Parquet bytes directly from ClickHouse → gRPC
- Enforce `readonly=1`, `max_result_rows`, `max_execution_time`
- Kill orphaned queries via ClickHouse `KILL QUERY`

---

## 2. The Passthrough Model

Unlike PG and MSSQL sidecars, ClickHouse sidecar performs **no row-to-Arrow conversion**. ClickHouse supports `FORMAT ArrowStream` and `FORMAT Parquet` natively.

```
Client ─gRPC─▶ Envoy ─gRPC─▶ CH Sidecar ─HTTP─▶ ClickHouse
                                  │                    │
                                  │ Inject headers:    │ Returns:
                                  │ X-ClickHouse-User  │ Arrow IPC bytes
                                  │ readonly=1         │ (already serialized)
                                  │ max_result_rows    │
                                  │                    │
                                  ◀────────────────────┘
                                  │ Pass-through bytes
                              gRPC stream back to client
```

---

## 3. Format Mapping

| Client `OutputFormat` | ClickHouse `FORMAT` | Sidecar CPU |
|---|---|---|
| `ARROW_IPC` | `ArrowStream` | **Near-zero** — passthrough |
| `PARQUET` | `Parquet` | **Near-zero** — passthrough |
| `JSON` | `ArrowStream` + sidecar conversion | Medium |

---

## 4. Driver Choice: Raw HTTP vs `clickhouse-connect`

`clickhouse-connect` is the official ClickHouse Python driver. It also uses HTTP under the hood. Here's why we chose raw `httpx` instead:

| Factor | `clickhouse-connect` | Raw `httpx` |
|---|---|---|
| **Protocol** | HTTP (same) | HTTP (same) |
| **Arrow support** | `query_arrow_stream()` → `RecordBatchStreamReader` | `FORMAT ArrowStream` → raw bytes |
| **Deserialization** | Deserializes Arrow into Python objects, then re-serializes | **None** — bytes pass through untouched |
| **Streaming** | Processes block-by-block via internal Native format | `aiter_bytes()` — raw chunked HTTP |
| **Credential injection** | Via constructor (fixed per client instance) | Via headers (per-request, per-user) |
| **Setting headers** | `settings={}` dict on the client | `X-ClickHouse-Setting-*` headers (same mechanism) |
| **Overhead** | Cython deserialization + pyarrow RecordBatch construction | **Zero** — no deserialization |
| **Dependency weight** | `clickhouse-connect` + transitive deps | `httpx` only |

> [!IMPORTANT]
> **`clickhouse-connect` is the right choice when you need to process query results in Python** (DataFrames, transformations, aggregations). For our sidecar, the ClickHouse response bytes ARE the gRPC response bytes — there is nothing to process. Adding `clickhouse-connect` would mean deserializing Arrow into Python objects only to serialize them back to Arrow, which is pure waste.

**When to reconsider:** If the sidecar ever needs to inspect, transform, or filter results before forwarding (e.g., column-level redaction, PII masking), `clickhouse-connect`'s `query_arrow_stream()` returning `RecordBatchStreamReader` would be the right tool.

## 5. Query Execution

```python
class ClickHouseSidecar(DataGatewayServicer):
    def __init__(self, ch_http_url: str):
        self._ch_url = ch_http_url
        self._http = httpx.AsyncClient(
            timeout=httpx.Timeout(connect=5.0, read=600.0, write=30.0),
            limits=httpx.Limits(max_connections=64, max_keepalive_connections=32),
        )

    async def QueryStream(self, request, context):
        metadata = dict(context.invocation_metadata())
        identity = metadata["x-client-identity"]
        trace_id = metadata.get("x-trace-id", str(uuid4()))
        tier_max_rows = int(metadata.get("x-max-rows-ceiling", "1000000"))
        tier_max_timeout = int(metadata.get("x-max-timeout-ceiling", "120"))

        # Resolve per-user ClickHouse credentials from Vault
        db_user, db_pass = await self._resolve_credentials(identity, "clickhouse")

        max_rows = min(request.limits.max_rows or 1_000_000, tier_max_rows)
        timeout_s = min(request.limits.timeout_seconds or 120, tier_max_timeout)

        ch_format = "ArrowStream" if request.format != OutputFormat.PARQUET else "Parquet"
        sql = f"/* trace_id={trace_id} */ {request.sql} FORMAT {ch_format}"

        headers = {
            "X-ClickHouse-User": db_user,
            "X-ClickHouse-Key": db_pass,
            "X-ClickHouse-Setting-readonly": "1",
            "X-ClickHouse-Setting-max_result_rows": str(max_rows),
            "X-ClickHouse-Setting-max_execution_time": str(timeout_s),
            "X-ClickHouse-Setting-max_bytes_to_read": str(
                request.limits.max_bytes or 50 * 1024**3),
            "X-ClickHouse-Setting-log_comment": f"identity={identity} trace_id={trace_id}",
            "X-ClickHouse-Setting-result_overflow_mode": "break",
        }

        async with self._http.stream(
            "POST", f"{self._ch_url}/?database={request.database}",
            content=sql, headers=headers,
        ) as resp:
            resp.raise_for_status()
            async for chunk in resp.aiter_bytes(chunk_size=1_048_576):
                yield QueryChunk(data=chunk, is_last=False)

        yield QueryChunk(data=b"", is_last=True,
                         metadata=ResponseMetadata(query_id=trace_id))
```

---

## 6. Query Assassination

```python
async def _kill_query(self, trace_id: str, identity: str):
    # Resolve user's creds to issue KILL
    db_user, db_pass = await self._resolve_credentials(identity, "clickhouse")
    kill_sql = f"KILL QUERY WHERE query LIKE '%trace_id={trace_id}%' SYNC"
    await self._http.post(self._ch_url, content=kill_sql,
                          headers={"X-ClickHouse-User": db_user,
                                   "X-ClickHouse-Key": db_pass})
```

---

## 7. Optimizations

| Optimization | How |
|---|---|
| HTTP/2 multiplexing | `httpx.AsyncClient` persistent connections |
| `result_overflow_mode=break` | Stops at `max_result_rows` cleanly |
| LZ4 wire compression | `Accept-Encoding: lz4` on CH requests |
| ArrowStream vs Parquet | ArrowStream for interactive; Parquet for bulk download |

---

## 8. Deployment

| Parameter | Value |
|---|---|
| Replicas | 3 |
| CPU | 0.5 vCPU (passthrough) |
| Memory | 512 MB |
| gRPC port | 50052 |
| Dependencies | `httpx`, `pyarrow` (JSON path only), `grpcio[aio]` |
