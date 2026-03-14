# Part 6: Python Client SDK — Design Document

> **Component:** Python client library for the Data Gateway  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md)  
> **Output:** `pip install gateway-client` → `from gateway import GatewayClient`

---

## 1. Design Goals

- **Zero-copy to Polars:** Arrow IPC → `polars.from_arrow()` — no intermediate pandas
- **Async-first:** `async def query()` with `asyncio` + `grpcio[aio]`
- **Sync wrapper:** `def query_sync()` for scripts and notebooks
- **Type-safe:** Full type annotations, IDE autocompletion
- **Transparent streaming:** Large results stream automatically; user gets a single DataFrame

---

## 2. Public API

```python
class GatewayClient:
    """Client for the Enterprise Data Gateway."""

    def __init__(
        self,
        gateway_url: str,                # "gateway.example.com:8443"
        ca_cert: str | Path,             # CA chain PEM
        client_cert: str | Path,         # Client cert PEM (identity)
        client_key: str | Path,          # Client private key PEM
        default_target: str = "pg",      # Default database target
        default_format: OutputFormat = OutputFormat.ARROW_IPC,
    ): ...

    async def query(
        self,
        sql: str,
        target: str | None = None,       # Override default_target
        database: str = "default",
        max_rows: int = 0,               # 0 = server default
        timeout: int = 60,
        format: OutputFormat | None = None,
        trace_id: str | None = None,     # OTel trace ID (auto-generated if None)
    ) -> pl.DataFrame: ...

    async def query_arrow(self, sql: str, **kwargs) -> pa.Table: ...

    async def query_parquet(self, sql: str, path: Path, **kwargs) -> Path: ...

    def query_sync(self, sql: str, **kwargs) -> pl.DataFrame: ...

    async def close(self) -> None: ...
    async def __aenter__(self) -> "GatewayClient": ...
    async def __aexit__(self, *args) -> None: ...
```

---

## 3. Core Implementation

```python
class GatewayClient:
    def __init__(self, gateway_url, ca_cert, client_cert, client_key,
                 default_target="pg", default_format=OutputFormat.ARROW_IPC):
        creds = grpc.ssl_channel_credentials(
            root_certificates=Path(ca_cert).read_bytes(),
            private_key=Path(client_key).read_bytes(),
            certificate_chain=Path(client_cert).read_bytes(),
        )
        self._channel = grpc.aio.secure_channel(gateway_url, creds, options=[
            ("grpc.max_receive_message_length", 256 * 1024 * 1024),
            ("grpc.keepalive_time_ms", 30_000),
        ])
        self._stub = DataGatewayStub(self._channel)
        self._default_target = default_target
        self._default_format = default_format

    async def query(self, sql, target=None, database="default",
                    max_rows=0, timeout=60, format=None, trace_id=None):
        target = target or self._default_target
        fmt = format or self._default_format
        tid = trace_id or str(uuid4())
        request = QueryRequest(
            target=target, database=database, sql=sql,
            format=fmt,
            limits=QueryLimits(max_rows=max_rows, timeout_seconds=timeout),
            trace_id=tid,
        )
        metadata = [("x-target-db", target)]

        # Stream Arrow IPC chunks
        batches: list[pa.RecordBatch] = []
        resp_meta: ResponseMetadata | None = None

        try:
            async for chunk in self._stub.QueryStream(request, metadata=metadata):
                if chunk.data:
                    reader = pa.ipc.open_stream(chunk.data)
                    for batch in reader:
                        batches.append(batch)
                if chunk.is_last:
                    resp_meta = chunk.metadata
        except grpc.aio.AioRpcError as exc:
            raise self._map_error(exc) from exc

        if not batches:
            return pl.DataFrame()

        table = pa.Table.from_batches(batches)
        return pl.from_arrow(table)  # Zero-copy

    def query_sync(self, sql, **kwargs):
        """Synchronous wrapper for notebooks and scripts."""
        loop = asyncio.get_event_loop()
        if loop.is_running():
            import nest_asyncio
            nest_asyncio.apply()
        return asyncio.run(self.query(sql, **kwargs))
```

---

## 4. Error Mapping

```python
class GatewayError(Exception): ...
class AuthenticationError(GatewayError): ...
class PermissionDeniedError(GatewayError): ...
class RateLimitError(GatewayError):
    retry_after: int
class QueryTimeoutError(GatewayError): ...
class InvalidQueryError(GatewayError): ...
class ServiceUnavailableError(GatewayError): ...

GRPC_ERROR_MAP = {
    grpc.StatusCode.UNAUTHENTICATED: AuthenticationError,
    grpc.StatusCode.PERMISSION_DENIED: PermissionDeniedError,
    grpc.StatusCode.RESOURCE_EXHAUSTED: RateLimitError,
    grpc.StatusCode.DEADLINE_EXCEEDED: QueryTimeoutError,
    grpc.StatusCode.INVALID_ARGUMENT: InvalidQueryError,
    grpc.StatusCode.FAILED_PRECONDITION: InvalidQueryError,
    grpc.StatusCode.UNAVAILABLE: ServiceUnavailableError,
}
```

---

## 5. Usage Examples

```python
# Async usage — identity comes from the client cert, no API key needed
async with GatewayClient("gw.example.com:8443",
                          ca_cert="ca.pem", client_cert="client.crt",
                          client_key="client.key") as gw:
    # PostgreSQL query → Polars DataFrame
    df = await gw.query("SELECT * FROM orders WHERE date > '2025-01-01'",
                        target="pg", database="analytics")
    print(df.shape)  # (1_000_000, 15)

    # ClickHouse query
    ch_df = await gw.query("SELECT * FROM events LIMIT 100000",
                           target="clickhouse")

    # SQL Server
    ms_df = await gw.query("SELECT TOP 1000 * FROM reports",
                           target="mssql", database="warehouse")

# Sync usage (notebooks)
gw = GatewayClient(...)
df = gw.query_sync("SELECT * FROM users LIMIT 100")
```

---

## 6. Package Structure

```
gateway-client/
├── pyproject.toml
├── src/gateway/
│   ├── __init__.py          # Re-export GatewayClient
│   ├── client.py            # GatewayClient implementation
│   ├── errors.py            # Exception hierarchy
│   ├── proto/               # Generated protobuf stubs
│   │   ├── gateway_pb2.py
│   │   └── gateway_pb2_grpc.py
│   └── _sync.py             # Sync wrapper utilities
└── tests/
    ├── test_client.py
    └── conftest.py
```

**Dependencies:** `grpcio[aio]`, `pyarrow`, `polars`, `protobuf`
