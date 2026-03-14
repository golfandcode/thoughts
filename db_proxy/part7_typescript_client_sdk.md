# Part 7: TypeScript Client SDK — Design Document

> **Component:** TypeScript client library for the Data Gateway  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md)  
> **Output:** `npm install @gateway/client` → `import { GatewayClient } from "@gateway/client"`

---

## 1. Design Goals

- **Arrow IPC → DuckDB-WASM:** Results load into an in-browser analytical engine
- **Local-first analytics:** UI queries DuckDB locally — no round-trips for pagination/filtering
- **Web Worker isolation:** Heavy processing off main thread
- **gRPC transport:** Connect-ES (or gRPC-web) through Envoy
- **Type-safe:** Generated TypeScript types from protobuf

---

## 2. Architecture

### 2a. Node.js / CLI (Direct mTLS)

For server-side TypeScript (Node.js scripts, CLI tools, backend services), the SDK connects directly to the gateway using mTLS — same as the Python SDK.

```
┌─────────────────┐     gRPC + mTLS     ┌─────────────────┐
│ Node.js Script   │ ──────────────────▶ │ Envoy Gateway    │
│ (has cert + key) │ ◀────────────────── │                  │
└─────────────────┘   Arrow IPC stream  └─────────────────┘
```

### 2b. Browser (BFF + PingFederate OIDC)

Browsers **cannot** programmatically attach client certs to `fetch()` or gRPC-web calls. The browser's only cert mechanism is the OS/browser certificate store, which prompts the user the first time and has no JavaScript API. **This makes direct browser → gateway mTLS impractical for production.**

The solution is a **Backend-for-Frontend (BFF)** pattern:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                  │
│                                                                           │
│  ┌─────────────────────────┐                                             │
│  │ SPA (React/Vue/etc.)    │                                             │
│  │                         │  1. Redirect to PingFederate for login      │
│  │ OIDC Login ─────────────┼───────────────▶ PingFederate                │
│  │                         │  2. Auth code callback  ┌──────────────┐   │
│  │ ◀──────────────────────────────────────── │ PingFederate    │   │
│  │                         │                          │ (OIDC IdP)   │   │
│  │ GatewayClient           │                          └──────────────┘   │
│  │  .query()  ─────────────┼─────┐                                       │
│  │                         │     │ 3. REST/gRPC-web + HttpOnly cookie    │
│  │ DuckDB-WASM ◀───────────┼────Arrow IPC                                │
│  └─────────────────────────┘     │                                       │
└──────────────────────────────────┼───────────────────────────────────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │ BFF Service (Node.js)│
                        │                      │  4. gRPC + mTLS (cert + key)
                        │ - OIDC token verify  │──────────────────▶ Envoy Gateway
                        │ - Session management │◀──────────────────
                        │ - Holds mTLS cert    │  Arrow IPC stream
                        │ - Maps OIDC sub →    │
                        │   SPIFFE identity    │
                        └─────────────────────┘
```

---

## 3. Browser Authentication: PingFederate + BFF

### 3a. The Browser mTLS Problem

| Method | Mechanism | Why It Doesn't Work for Production |
|---|---|---|
| **OS cert store** | User imports `.p12` cert; browser presents it on TLS handshake | ❌ No JS control. Requires per-user cert provisioning to every laptop. Cert selection prompt is confusing. |
| **`AutoSelectCertificateForUrls` policy** | Chrome/Edge enterprise policy auto-selects cert by URL | ⚠️ Works on managed desktops but requires MDM (Intune/JAMF). No mobile. No contractor laptops. |
| **WebAuthn / FIDO2** | Hardware key or biometric | ❌ Not x509 — different protocol. Doesn't produce a SPIFFE URI. |
| **BFF + OIDC** | User logs in via PingFederate; BFF holds the mTLS cert | ✅ **Production-ready.** No cert on the browser. Works on any device. |

### 3b. BFF Authentication Flow

```
1. User opens dashboard → SPA redirects to PingFederate
2. PingFederate authenticates user (SSO, MFA, etc.)
3. PingFederate returns OIDC authorization code to SPA callback
4. SPA sends auth code to BFF → BFF exchanges for access_token + id_token
5. BFF validates id_token, extracts user identity (e.g., "sub": "matt@corp.com")
6. BFF maps OIDC subject to SPIFFE identity: matt@corp.com → spiffe://corp/user/matt
7. BFF establishes session with SPA via HttpOnly + Secure + SameSite cookie
8. SPA sends query request to BFF with session cookie
9. BFF calls gateway via gRPC + mTLS, injecting x-target-db header
10. Arrow IPC response streams back through BFF to browser
```

### 3c. PingFederate Configuration Requirements

| Requirement | Configuration |
|---|---|
| **OIDC Client** | Register BFF as a confidential client (`client_secret_basic` or `private_key_jwt`) |
| **Grant type** | `authorization_code` with PKCE |
| **Scopes** | `openid`, `profile`, custom `gateway:query` scope |
| **Claims mapping** | Ensure `sub` claim contains the user's identity (email, employee ID, or CN) |
| **Token lifetime** | Access token: 15 min. Refresh token: 8 hours (session length) |
| **MFA policy** | Recommended: Adaptive MFA via PingFederate authentication policies |
| **CORS** | Allow SPA origin(s) on PingFederate's OIDC endpoints |

### 3d. BFF Identity Mapping

The BFF is the **trust translation layer** between OIDC (PingFederate) and SPIFFE (gateway):

```typescript
// BFF: Map OIDC identity → SPIFFE identity for gateway calls
class IdentityMapper {
  /**
   * PingFederate's OIDC id_token 'sub' claim → SPIFFE URI
   * This mapping is the single source of truth for browser users.
   */
  mapToSpiffe(oidcSubject: string): string {
    // Option 1: Convention-based (if PingFederate 'sub' is email)
    // matt@corp.com → spiffe://corp/user/matt
    const [username, domain] = oidcSubject.split("@");
    return `spiffe://${domain.split(".")[0]}/user/${username}`;
  }
}
```

### 3e. BFF Security Requirements

| Requirement | Implementation |
|---|---|
| **Session storage** | Server-side sessions (Redis-backed). Session ID in HttpOnly cookie only. |
| **CSRF protection** | `SameSite=Strict` cookie + CSRF token for mutating operations |
| **Token storage** | Access/refresh tokens stored server-side (never sent to browser) |
| **mTLS cert** | BFF holds a **service-level** mTLS cert (`spiffe://corp/svc/bff`) |
| **OIDC → SPIFFE** | BFF maps OIDC `sub` to SPIFFE URI and passes as `x-client-identity` |
| **Rate limiting** | BFF enforces per-session rate limits in addition to gateway-level |
| **Audit logging** | Log every query with OIDC subject, mapped SPIFFE ID, and result metadata |

### 3f. Two SDK Modes

```typescript
// Mode 1: Node.js (direct mTLS) — for scripts, backend services, CLI
const gw = new GatewayClient({
  gatewayUrl: "gateway.example.com:8443",
  tls: {
    caCert: fs.readFileSync("ca.pem"),
    clientCert: fs.readFileSync("client.crt"),
    clientKey: fs.readFileSync("client.key"),
  },
});

// Mode 2: Browser (via BFF) — for dashboards, web apps
const gw = new GatewayClient({
  bffUrl: "https://bff.example.com/api/gateway",
  // Auth handled by BFF session cookie (set after PingFederate OIDC login)
});
```

---

## 4. Public API

```typescript
interface GatewayClientOptions {
  // Mode 1: Direct mTLS (Node.js only)
  gatewayUrl?: string;
  tls?: {
    caCert: Buffer;
    clientCert: Buffer;
    clientKey: Buffer;
  };

  // Mode 2: BFF proxy (Browser)
  bffUrl?: string;

  // Common
  defaultTarget?: "pg" | "clickhouse" | "mssql";
}

class GatewayClient {
  constructor(options: GatewayClientOptions);

  /**
   * Execute a query, load results into local DuckDB, return a QueryResult
   * handle for local querying.
   */
  async query(options: QueryOptions): Promise<QueryResult>;

  /** Destroy client and release DuckDB resources */
  async close(): void;
}

interface QueryOptions {
  sql: string;
  target?: string;
  database?: string;
  traceId?: string;  // OTel trace ID (auto-generated if omitted)
  maxRows?: number;
  timeout?: number;
}

class QueryResult {
  /** Unique result ID (used as DuckDB table name) */
  readonly id: string;

  /** Total rows returned from the gateway */
  readonly totalRows: number;

  /** Was the result truncated by server limits? */
  readonly truncated: boolean;

  /** Query the local DuckDB copy with SQL */
  async localQuery(sql: string): Promise<arrow.Table>;

  /** Get a page of results as JSON (for UI rendering) */
  async getPage(offset: number, limit: number): Promise<Record<string, unknown>[]>;

  /** Export the full result as a downloadable Parquet blob */
  async toParquetBlob(): Promise<Blob>;

  /** Drop the local DuckDB table to free memory */
  async dispose(): void;
}
```

---

## 5. Core Implementation

```typescript
class GatewayClient {
  private db: duckdb.AsyncDuckDB;
  private conn: duckdb.AsyncDuckDBConnection;
  private transport: ConnectTransport;

  async query(options: QueryOptions): Promise<QueryResult> {
    const target = options.target ?? this.defaultTarget;
    const resultId = `result_${crypto.randomUUID().slice(0, 8)}`;

    // 1. Stream Arrow IPC chunks from gateway
    const chunks: Uint8Array[] = [];
    let metadata: ResponseMetadata | undefined;

    const stream = this.client.queryStream({
      target,
      database: options.database ?? "default",
      sql: options.sql,
      format: OutputFormat.ARROW_IPC,
      limits: {
        maxRows: options.maxRows ?? 0,
        timeoutSeconds: options.timeout ?? 60,
      },
    }, { headers: { "x-target-db": target } });

    for await (const chunk of stream) {
      if (chunk.data.byteLength > 0) chunks.push(chunk.data);
      if (chunk.isLast) metadata = chunk.metadata;
    }

    // 2. Concatenate and register in DuckDB-WASM
    const arrowBuffer = concatUint8Arrays(chunks);
    const fileName = `${resultId}.arrow`;
    await this.db.registerFileBuffer(fileName, arrowBuffer);

    // 3. Create a DuckDB table from the Arrow IPC file
    await this.conn.query(`
      CREATE TABLE "${resultId}" AS
      SELECT * FROM read_ipc('${fileName}')
    `);

    // 4. Return handle for local querying
    return new QueryResult(resultId, metadata, this.conn);
  }
}
```

### QueryResult Local Querying

```typescript
class QueryResult {
  async getPage(offset: number, limit: number): Promise<Record<string, unknown>[]> {
    const result = await this.conn.query(`
      SELECT * FROM "${this.id}" LIMIT ${limit} OFFSET ${offset}
    `);
    return result.toArray().map(row => row.toJSON());
  }

  async localQuery(sql: string): Promise<arrow.Table> {
    // User can write arbitrary SQL against the local table
    // e.g., "SELECT category, SUM(amount) FROM result_abc GROUP BY 1"
    return await this.conn.query(sql);
  }

  async toParquetBlob(): Promise<Blob> {
    await this.conn.query(`
      COPY "${this.id}" TO '${this.id}.parquet' (FORMAT PARQUET)
    `);
    const buffer = await this.db.copyFileToBuffer(`${this.id}.parquet`);
    return new Blob([buffer], { type: "application/octet-stream" });
  }

  async dispose(): void {
    await this.conn.query(`DROP TABLE IF EXISTS "${this.id}"`);
  }
}
```

---

## 6. DuckDB-WASM Considerations

| Factor | Value |
|---|---|
| WASM bundle size | ~4 MB (compressed) |
| Init time | ~500ms (first load) |
| Memory limit | Browser-dependent: ~1–2 GB practical |
| Threading | Single-threaded in WASM; use Web Worker for isolation |
| Arrow IPC ingest | `read_ipc()` — native, 10–100× faster than JSON parse |
| Local SQL | Full DuckDB SQL support (joins, aggregations, window functions) |

> [!WARNING]
> **Browser memory ceiling:** A 2 GB result set will consume ~2 GB of browser memory. Set sensible `maxRows` defaults in the SDK (e.g., 100K for interactive, 1M for dashboard) and warn users when results exceed 500 MB.

---

## 7. Package Structure

```
@gateway/client/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              # Re-export GatewayClient
│   ├── client.ts             # GatewayClient implementation
│   ├── client-node.ts        # Node.js mTLS transport
│   ├── client-bff.ts         # Browser BFF transport
│   ├── bff/                  # BFF server (deploy separately)
│   │   ├── server.ts         # Express/Fastify BFF entry point
│   │   ├── oidc.ts           # PingFederate OIDC integration
│   │   ├── identity-mapper.ts # OIDC sub → SPIFFE mapping
│   │   └── session.ts        # Redis-backed session management
│   ├── result.ts             # QueryResult class
│   ├── errors.ts             # Error hierarchy
│   ├── worker.ts             # Web Worker entry point (DuckDB init)
│   ├── proto/                # Generated Connect-ES stubs
│   │   ├── gateway_pb.ts
│   │   └── gateway_connect.ts
│   └── utils.ts              # Buffer concatenation, etc.
├── tests/
│   └── client.test.ts
└── README.md
```

**Dependencies:** `@connectrpc/connect-web`, `@duckdb/duckdb-wasm`, `apache-arrow`, `openid-client` (BFF)
