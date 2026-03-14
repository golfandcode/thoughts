# OTel Semantic Conventions Reference — Data Gateway

> **Source:** [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)  
> **Covers:** PostgreSQL, SQL Server, ClickHouse, Envoy, Nginx, mTLS, x509, JWT, gRPC  
> **Applies to:** All gateway components (Parts 0–8)

---

## Databases — PostgreSQL, SQL Server, ClickHouse

### Span & Metric Attributes

| Semantic Name | Type | Req. Level | Description & When to Use |
|---|---|---|---|
| `db.system.name` | Attribute | **Required** | Database system identifier. Use `postgresql`, `mssql`, or `clickhouse`. Set at span creation. If the actual DBMS differs (e.g., CockroachDB via PG driver), use the client library's system. |
| `db.namespace` | Attribute | **Cond. Required** | Database + schema identifier. PG: `{database}\|{schema}` (e.g., `analytics\|public`). MSSQL: `{instance}\|{database}` (e.g., `MSSQLSERVER\|analytics`). CH: database name (e.g., `default`). Concatenate from most general to most specific with `\|`. |
| `db.collection.name` | Attribute | **Cond. Required** | Table or collection being accessed. Set when a single collection is targeted. Do NOT extract from `db.query.text` for multi-table queries. Use as provided by the application (no case normalization). |
| `db.operation.name` | Attribute | **Cond. Required** | High-level SQL operation: `SELECT`, `INSERT`, `UPDATE`, `DELETE`. For batch ops, prepend with `BATCH` (e.g., `BATCH INSERT`). Do NOT extract from `db.query.text` when multiple operations are present. |
| `db.query.text` | Attribute | **Opt-In** | The parameterized query text. Use with `?` placeholders, never raw user values. **⚠️ High PII risk** — must be explicitly opted-in; sanitize or truncate before export. |
| `db.query.summary` | Attribute | **Recommended** | A short categorization of the query for grouping: `SELECT users_table`, `INSERT orders SELECT products`. Useful for dashboard aggregation without the full query text. |
| `db.query.parameter.<key>` | Attribute | **Opt-In** | Individual query parameter values. Use sparingly — **high cardinality and PII risk**. Only enable for debugging scenarios. |
| `db.response.status_code` | Attribute | **Cond. Required** | Native DB error code. PG: SQLSTATE (e.g., `08P01`, `23505`). MSSQL: severity level or error number (e.g., `2627`). CH: exception code. Set when the operation fails. |
| `db.response.returned_rows` | Attribute | **Opt-In** | Number of rows returned by the query. Use for resource attribution and anomaly detection (unexpected row counts). |
| `db.stored_procedure.name` | Attribute | **Recommended** | Stored procedure name when the operation targets one. Highly relevant for MSSQL workloads. |
| `db.operation.batch.size` | Attribute | **Recommended** | Number of operations in a batch. Use for bulk INSERT/UPDATE monitoring. |
| `server.address` | Attribute | **Recommended** | Database server hostname or IP (e.g., `pg-primary.corp.com`, `10.1.2.80`). |
| `server.port` | Attribute | **Cond. Required** | Database server port. Required when non-default (PG: not 5432, CH: not 8123/9000, MSSQL: not 1433). |
| `network.peer.address` | Attribute | **Recommended** | Resolved IP address of the DB peer (differs from `server.address` when DNS is involved). |
| `network.peer.port` | Attribute | **Recommended** | Actual port used (may differ from `server.port` through a proxy). |
| `error.type` | Attribute | **Cond. Required** | Should match `db.response.status_code` or exception class name. Use for span status and error rate metrics. |

### `db.system.name` Well-Known Identifiers

| Value | Database |
|---|---|
| `postgresql` | PostgreSQL (also CockroachDB via PG driver) |
| `mssql` | Microsoft SQL Server |
| `clickhouse` | ClickHouse |
| `other_sql` | Fallback for unknown SQL-compliant DBs |

---

## Database Client Metrics

| Metric Name | Type | Unit | Req. Level | Description & When to Use |
|---|---|---|---|---|
| `db.client.operation.duration` | Histogram | `s` | **Required** | Duration of a single DB operation. Bucket boundaries: `[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10]`. Should equal the span duration. Labels: `db.system.name`, `db.namespace`, `db.operation.name`, `db.collection.name`, `server.address`. |

### Connection Pool Metrics

| Metric Name | Type | Unit | Req. Level | Description & When to Use |
|---|---|---|---|---|
| `db.client.connection.count` | UpDownCounter | `{connection}` | **Required** | Current connection count by state. Labels: `db.client.connection.pool.name`, `db.client.connection.state` (`idle`/`used`). **Critical for sidecar pool monitoring.** |
| `db.client.connection.max` | Gauge | `{connection}` | **Recommended** | Maximum connections allowed. Use to calculate pool saturation: `count / max`. |
| `db.client.connection.idle.max` | Gauge | `{connection}` | **Recommended** | Maximum idle connections allowed in the pool. |
| `db.client.connection.idle.min` | Gauge | `{connection}` | **Recommended** | Minimum idle connections maintained by the pool. |
| `db.client.connection.pending_requests` | UpDownCounter | `{request}` | **Recommended** | Requests waiting for a connection from the pool. Non-zero = pool saturation. |

**Pool name convention:** `{server.address}:{server.port}/{db.namespace}` (e.g., `pg-primary:5432/analytics`).

---

## gRPC / RPC

### Span Attributes

| Semantic Name | Type | Req. Level | Description & When to Use |
|---|---|---|---|
| `rpc.system.name` | Attribute | **Required** | MUST be `grpc` for all gateway gRPC calls. Set at span creation time. |
| `rpc.method` | Attribute | **Required** (gRPC) | Fully-qualified method name: `gateway.v1.DataGateway/QueryStream`. Set to `_OTHER` if method is unrecognized (must also set `rpc.method_original`). |
| `rpc.method_original` | Attribute | **Cond. Required** | Original method name when `rpc.method` = `_OTHER`. Use for unknown methods received by sidecar. |
| `rpc.response.status_code` | Attribute | **Required** (gRPC) | gRPC status code string. All codes except `OK` are errors: `OK`, `CANCELLED`, `UNKNOWN`, `INVALID_ARGUMENT`, `DEADLINE_EXCEEDED`, `NOT_FOUND`, `PERMISSION_DENIED`, `RESOURCE_EXHAUSTED`, `FAILED_PRECONDITION`, `ABORTED`, `UNIMPLEMENTED`, `INTERNAL`, `UNAVAILABLE`, `DATA_LOSS`, `UNAUTHENTICATED`. |
| `rpc.request.metadata.<key>` | Attribute | **Opt-In** | Specific gRPC request metadata values. Use for `x-target-db`, `x-client-identity`, `x-rate-tier`. **⚠️ Never log credentials.** |
| `rpc.response.metadata.<key>` | Attribute | **Opt-In** | Specific gRPC response metadata values. Use for `x-query-id`, `x-rows-returned`. |
| `server.address` | Attribute | **Required** (client) | Gateway or sidecar hostname (e.g., `gateway.example.com`, `sidecar-pg-1`). |
| `server.port` | Attribute | **Cond. Required** | Port number (e.g., `8443` for gateway, `50051` for sidecar). Required when non-default. |
| `error.type` | Attribute | **Cond. Required** | Set to `rpc.response.status_code` value on error. For pre-status errors, use exception class name. |

### RPC Metrics

| Metric Name | Type | Unit | Req. Level | Description & When to Use |
|---|---|---|---|---|
| `rpc.server.call.duration` | Histogram | `s` | **Required** | Server-side gRPC call duration. Buckets: `[0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1, 2.5, 5, 7.5, 10]`. Use on sidecar and ext_authz. Labels: `rpc.system.name`, `rpc.method`, `rpc.response.status_code`, `error.type`. |
| `rpc.client.call.duration` | Histogram | `s` | **Required** | Client-side gRPC call duration. Use on SDK (Python/TS) and BFF for end-to-end latency. Same labels and buckets as server. |

> [!IMPORTANT]
> **Duration unit is seconds** (`s`), not milliseconds. This changed in recent semconv versions. Metric name changed from `rpc.server.duration` → `rpc.server.call.duration`.

---

## HTTP — Envoy, Nginx, Sidecar-to-ClickHouse

### Client Span Attributes (Sidecar → ClickHouse HTTP API)

| Semantic Name | Type | Req. Level | Description & When to Use |
|---|---|---|---|
| `http.request.method` | Attribute | **Required** | `POST` for ClickHouse HTTP API queries. `GET` for health checks. |
| `http.response.status_code` | Attribute | **Cond. Required** | HTTP status code from ClickHouse (200, 400, 500, etc.). Set when response is received. |
| `http.request.method_original` | Attribute | **Cond. Required** | Only when `http.request.method` was normalized (rare for our case). |
| `url.full` | Attribute | **Required** | Full URL including query params: `http://ch-primary:8123/?database=analytics`. **⚠️ Redact password params.** |
| `url.scheme` | Attribute | **Recommended** | `http` or `https`. Use when TLS status matters for debugging. |
| `http.request.resend_count` | Attribute | **Recommended** | Number of retry attempts. Use for retry-storm detection. |
| `network.protocol.name` | Attribute | **Cond. Required** | `http` for CH sidecar HTTP calls. `h2` for HTTP/2 calls via Envoy. |
| `network.protocol.version` | Attribute | **Recommended** | `1.1`, `2`, `3`. Use to detect protocol downgrade issues. |

### Server Span Attributes (Envoy/Nginx receiving client requests)

| Semantic Name | Type | Req. Level | Description & When to Use |
|---|---|---|---|
| `http.route` | Attribute | **Recommended** | Route template that matched: `/gateway.v1.DataGateway/QueryStream`. Use for cardinality-safe request grouping. |
| `url.path` | Attribute | **Required** | The URI path component. |
| `url.query` | Attribute | **Opt-In** | Query string component. **⚠️ May contain sensitive parameters.** |
| `client.address` | Attribute | **Recommended** | Downstream client IP connecting to Envoy/Nginx (from X-Forwarded-For or socket). |
| `client.port` | Attribute | **Opt-In** | Downstream client port. |
| `user_agent.original` | Attribute | **Recommended** | Client User-Agent string for SDK version tracking. |
| `http.request.body.size` | Attribute | **Recommended** | Request body size in bytes (SQL query payload). |
| `http.response.body.size` | Attribute | **Recommended** | Response body size in bytes (Arrow IPC payload). |

---

## mTLS, TLS & x509 Certificates

| Semantic Name | Type | Description & When to Use |
|---|---|---|
| `tls.cipher` | Attribute | Negotiated TLS cipher suite (e.g., `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`). Use to enforce cipher policy compliance. |
| `tls.curve` | Attribute | Elliptic curve used (e.g., `secp256r1`, `X25519`). Use for cryptographic strength auditing. |
| `tls.protocol.name` | Attribute | `tls` (normalized lowercase). Use to distinguish SSL vs TLS. |
| `tls.protocol.version` | Attribute | `1.2`, `1.3`. **Alert on anything below 1.2.** |
| `tls.established` | Attribute | Boolean — `true` if TLS handshake succeeded. Use to detect failed mTLS negotiations. |
| `tls.resumed` | Attribute | Boolean — `true` if TLS session was resumed. Use for session cache hit rate. |
| `tls.next_protocol` | Attribute | ALPN protocol negotiated (e.g., `h2`, `http/1.1`). Use to verify HTTP/2 for gRPC. |
| `tls.client.certificate` | Attribute | PEM-encoded client cert. **⚠️ High cardinality, large payload.** Only log for auth failure debugging. |
| `tls.client.certificate_chain` | Attribute | Array of PEM-encoded certs in the client chain. Same caution as above. |
| `tls.client.subject` | Attribute | Client cert DN: `CN=matt,OU=Engineering,DC=corp,DC=com`. **Use for identity attribution on ext_authz spans.** |
| `tls.client.issuer` | Attribute | Issuer DN: `CN=Corp Root CA`. Use to verify cert was issued by the expected CA. |
| `tls.client.not_before` / `tls.client.not_after` | Attribute | Certificate validity window. Use to detect expired or not-yet-valid certs. |
| `tls.client.hash.sha256` | Attribute | SHA256 fingerprint of client cert. Use as a stable, low-cardinality identity correlator (instead of full PEM). |
| `tls.client.ja3` | Attribute | JA3 fingerprint of client TLS handshake. Use for client fingerprinting and bot detection. |
| `tls.client.supported_ciphers` | Attribute | Array of ciphers offered by client. Use for compliance and capability auditing. |
| `tls.server.certificate` | Attribute | PEM-encoded server cert. Log only for debugging server cert issues. |
| `tls.server.subject` | Attribute | Server cert DN. Use to verify Envoy/sidecar is presenting the correct cert. |
| `tls.server.issuer` | Attribute | Server cert issuer DN. |
| `tls.server.not_before` / `tls.server.not_after` | Attribute | Server cert validity. **Alert when `not_after` is within 30 days.** |
| `tls.server.hash.sha256` | Attribute | SHA256 fingerprint of server cert. Use for cert rotation tracking. |
| `tls.server.ja3s` | Attribute | JA3S fingerprint of server TLS handshake. Use to detect server config drift. |

---

## JWT / OIDC Authentication (BFF Layer)

> [!NOTE]
> OTel does not define JWT-specific semantic conventions. These are the closest conventions plus recommended custom attributes for the BFF layer.

| Semantic Name | Type | Description & When to Use |
|---|---|---|
| `http.request.header.authorization` | Attribute | **⚠️ Opt-In only.** Captures the Authorization header containing the JWT. **MUST be sanitized** — log only token type (e.g., `Bearer ***`), never the full token. |
| `enduser.id` | Attribute | Authenticated user identity extracted from the JWT `sub` claim. Use on BFF spans after OIDC validation (e.g., `matt@corp.com`). |
| `enduser.role` | Attribute | User's role from JWT claims. Use for RBAC attribution (e.g., `premium`, `standard`). |
| `enduser.scope` | Attribute | OAuth scopes from the token (e.g., `openid profile gateway:query`). Use to verify scope enforcement. |

**Custom attributes for the gateway (not in OTel spec):**

| Custom Name | Type | Description & When to Use |
|---|---|---|
| `gateway.identity` | Attribute | SPIFFE URI mapped from cert SAN or OIDC subject: `spiffe://corp/user/matt`. Use on every gateway span for identity correlation. |
| `gateway.rate_tier` | Attribute | Rate limit tier for the identity: `standard`, `premium`. |
| `gateway.target` | Attribute | Target database: `pg`, `clickhouse`, `mssql`. |
| `gateway.format` | Attribute | Output format: `ARROW_IPC`, `PARQUET`, `JSON`. |
| `gateway.trace_id` | Attribute | Application-level trace ID from `QueryRequest.trace_id`. |
| `gateway.rows_returned` | Attribute | Rows returned to the client. |
| `gateway.bytes_sent` | Attribute | Bytes sent to the client (Arrow IPC payload). |
| `gateway.truncated` | Attribute | Boolean — was the result truncated by max_rows or max_bytes. |
| `gateway.driver` | Attribute | Sidecar driver used: `asyncpg`, `connectorx`, `turbodbc`, `httpx`. |

---

## Network & General

| Semantic Name | Type | Description & When to Use |
|---|---|---|
| `server.address` | Attribute | Logical server hostname or IP. For proxied connections, the ultimate backend address. |
| `server.port` | Attribute | Logical server port. |
| `client.address` | Attribute | Client IP (from Envoy/Nginx perspective). Use for geo-location and access control logging. |
| `client.port` | Attribute | Client ephemeral port. |
| `network.transport` | Attribute | Transport protocol: `tcp`, `udp`, `quic`. Use `tcp` for all gateway gRPC and HTTP traffic. |
| `network.type` | Attribute | Network layer: `ipv4`, `ipv6`. |
| `network.peer.address` | Attribute | Resolved IP of the peer (may differ from `server.address` after DNS). |
| `network.peer.port` | Attribute | Actual peer port used. |
| `network.protocol.name` | Attribute | Application protocol: `http`, `grpc`. |
| `network.protocol.version` | Attribute | Protocol version: `1.1`, `2`, `3`. |

---

## Implementation Notes

> [!WARNING]
> **Redaction Risk**: `db.query.text`, `db.query.parameter.<key>`, `http.request.header.authorization`, `rpc.request.metadata.<key>`, and `tls.client.certificate` can leak PII or credentials. Use strict allow-listing and truncation before export. Configure your OTel Collector's `attributes` processor to redact or hash sensitive values.

> [!IMPORTANT]
> **Duration Metric Units**: Recent semconv updates changed duration metrics from milliseconds (`ms`) to **seconds (`s`)** and renamed them:
> - `rpc.server.duration` → `rpc.server.call.duration`
> - `rpc.client.duration` → `rpc.client.call.duration`
> - `db.client.operation.duration` is in **seconds**, not milliseconds
>
> Ensure all histogram buckets are specified in seconds.

> [!TIP]
> **Cardinality Management**: Attributes like `db.query.text`, `rpc.method`, and `tls.client.hash.sha256` can have unbounded cardinality. Use `db.query.summary` instead of `db.query.text` on metrics. Hash or truncate high-cardinality values. Configure your Collector's `transform` processor to control label cardinality.
