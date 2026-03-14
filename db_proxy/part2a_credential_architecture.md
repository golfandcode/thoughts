# Part 2A: Credential Architecture — Vault Dynamic Secrets

> **Decision:** Vault Database Secrets Engine + SPIFFE x509 Auth  
> **Status:** Canonical  
> **Referenced by:** [Part 2: ext_authz](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part2_ext_authz_service.md), [Part 3: PG Sidecar](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part3_pg_sidecar.md), [Part 4: CH Sidecar](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part4_ch_sidecar.md), [Part 5: MSSQL Sidecar](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part5_mssql_sidecar.md)

---

## 1. Options Evaluated

| # | Option | Security | Scalability | Enterprise-Ready | Verdict |
|---|---|---|---|---|---|
| A | **Vault KV (static secrets)** | 🟡 Passwords at rest, manual rotation | 🟡 Manual ops burden | 🟡 Requires custom rotation tooling | ❌ **Reject** — static secrets are the problem we're solving |
| B | **Vault Database Secrets Engine** | 🟢 Ephemeral creds, auto-revoked | 🟢 On-demand, zero manual ops | 🟢 Built-in lease/rotation/audit | ✅ **Winner** |
| C | **Vault Agent sidecar (file-based)** | 🟢 Same as B, but creds in files | 🟢 Vault Agent handles renewal | 🟡 Adds another sidecar per pod | ⚠️ Good but adds complexity — we already have a Python sidecar |
| D | **Cloud IAM DB auth** (GCP/AWS) | 🟢 No passwords at all, IAM tokens | 🟢 Fully managed | 🔴 Cloud-vendor-locked, no on-prem | ❌ **Reject** — must work across cloud + on-prem |
| E | **Kerberos / GSSAPI** | 🟢 No passwords, ticket-based | 🟡 Requires KDC infrastructure | 🟢 Enterprise standard for Windows/AD | ⚠️ SQL Server only; PG limited; ClickHouse: none |
| F | **DB-native cert auth** | 🟢 No passwords, cert = auth | 🟡 Per-DB cert management | 🔴 ClickHouse: no support; SQL Server: limited | ❌ **Reject** — not cross-database |

---

## 2. Winner: Vault Database Secrets Engine + SPIFFE x509 Auth

### Why This Wins

1. **Zero static credentials anywhere** — Vault dynamically creates DB users with TTL
2. **Sidecar authenticates to Vault using its SPIFFE SVID** — the same mTLS cert chain used for gateway auth
3. **Per-user DB connections** — Vault creates a unique DB user per identity, preserving RBAC/RLS/audit
4. **Automatic rotation** — credentials expire (1h default TTL), Vault revokes the DB user on expiry
5. **Full audit trail** — every credential issuance logged in Vault's audit log
6. **Works for all three databases** — PG, ClickHouse, SQL Server all have Vault database plugins

---

## 3. Credential Flow

```mermaid
sequenceDiagram
    participant Client
    participant Envoy
    participant ExtAuthz as ext_authz
    participant Sidecar
    participant Vault
    participant DB as Database

    Client->>Envoy: gRPC + mTLS client cert
    Envoy->>ExtAuthz: Check(cert SAN)
    ExtAuthz-->>Envoy: OK + x-client-identity, x-rate-tier
    Envoy->>Sidecar: Forward request + identity headers

    Note over Sidecar: Sidecar checks credential cache

    alt Cache HIT (valid lease)
        Sidecar->>DB: Connect with cached creds
    else Cache MISS or lease expiring
        Sidecar->>Vault: Auth via SPIFFE x509 SVID
        Vault-->>Sidecar: Vault token (bound to sidecar's SPIFFE ID)
        Sidecar->>Vault: GET /database/creds/{role}
        Note over Vault: Vault creates ephemeral DB user:<br/>v-sidecar-matt-pg-ro-abc123<br/>TTL: 1 hour
        Vault->>DB: CREATE ROLE ... WITH LOGIN PASSWORD ...
        Vault-->>Sidecar: {username, password, lease_id, ttl}
        Sidecar->>DB: Connect with dynamic creds
    end

    DB-->>Sidecar: Query results
    Sidecar-->>Envoy: Arrow IPC stream
    Envoy-->>Client: Arrow IPC stream

    Note over Vault: On lease expiry (1h):<br/>Vault → DB: DROP ROLE v-sidecar-matt-...
```

---

## 4. Vault Configuration

### 4.1 SPIFFE Auth Method

Vault authenticates the sidecar workload using its SPIFFE x509 SVID. The sidecar's mTLS cert (used for proxy→sidecar communication) doubles as its Vault authentication credential.

```bash
# Enable SPIFFE auth (Vault 1.21+)
vault auth enable -path=spiffe spiffe

# Configure trust bundle from SPIRE
vault write auth/spiffe/config \
  trust_bundle=@/path/to/spire-trust-bundle.pem

# Create roles mapping sidecar SPIFFE IDs to Vault policies
vault write auth/spiffe/role/pg-sidecar \
  spiffe_id="spiffe://corp/sidecar/pg" \
  token_policies="db-pg-dynamic" \
  token_ttl=15m \
  token_max_ttl=1h

vault write auth/spiffe/role/ch-sidecar \
  spiffe_id="spiffe://corp/sidecar/ch" \
  token_policies="db-ch-dynamic" \
  token_ttl=15m

vault write auth/spiffe/role/mssql-sidecar \
  spiffe_id="spiffe://corp/sidecar/mssql" \
  token_policies="db-mssql-dynamic" \
  token_ttl=15m
```

### 4.2 Database Secrets Engine — Per-DB Configuration

```bash
# Enable database secrets engine
vault secrets enable database

# --- PostgreSQL ---
vault write database/config/pg-primary \
  plugin_name=postgresql-database-plugin \
  allowed_roles="pg-readonly-*" \
  connection_url="postgresql://{{username}}:{{password}}@pg-primary:5432/postgres?sslmode=verify-full" \
  username="vault_admin" \
  password="vault_admin_password"

# --- ClickHouse ---
vault write database/config/ch-primary \
  plugin_name=clickhouse-database-plugin \
  allowed_roles="ch-readonly-*" \
  connection_url="clickhouse://{{username}}:{{password}}@ch-primary:8123" \
  username="vault_admin" \
  password="vault_admin_password"

# --- SQL Server ---
vault write database/config/mssql-primary \
  plugin_name=mssql-database-plugin \
  allowed_roles="mssql-readonly-*" \
  connection_url="sqlserver://{{username}}:{{password}}@mssql-primary:1433" \
  username="vault_admin" \
  password="vault_admin_password"
```

### 4.3 Dynamic Roles — Per-Identity Mapping

The key insight: **each user identity maps to a Vault role**, which generates a DB user with specific permissions.

```bash
# Role template: creates a read-only PG user for a specific identity
vault write database/roles/pg-readonly-standard \
  db_name=pg-primary \
  default_ttl=1h \
  max_ttl=4h \
  creation_statements="
    CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
    GRANT CONNECT ON DATABASE analytics TO \"{{name}}\";
    GRANT USAGE ON SCHEMA public TO \"{{name}}\";
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO \"{{name}}\";
    COMMENT ON ROLE \"{{name}}\" IS 'Vault-managed: identity={{identity}}, lease={{lease_id}}';
  " \
  revocation_statements="
    REASSIGN OWNED BY \"{{name}}\" TO postgres;
    DROP OWNED BY \"{{name}}\";
    DROP ROLE IF EXISTS \"{{name}}\";
  "
```

---

## 5. Sidecar Credential Manager

Each sidecar embeds a credential manager that handles Vault auth, credential retrieval, caching, and lease renewal.

```python
class VaultCredentialManager:
    """Manages per-identity DB credentials via Vault Database Secrets Engine."""

    def __init__(self, vault_addr: str, spiffe_cert: str, spiffe_key: str,
                 spiffe_ca: str, db_target: str):
        self._vault_addr = vault_addr
        self._db_target = db_target  # "pg", "ch", "mssql"
        self._spiffe_cert = spiffe_cert
        self._spiffe_key = spiffe_key
        self._spiffe_ca = spiffe_ca
        self._vault_token: str | None = None
        self._token_expiry: float = 0
        # Cache: identity → (username, password, lease_id, expiry)
        self._cache: dict[str, tuple[str, str, str, float]] = {}
        self._lock = asyncio.Lock()

    async def get_credentials(self, identity: str) -> tuple[str, str]:
        """Get DB credentials for an identity. Returns (username, password)."""
        # Check cache
        if identity in self._cache:
            user, password, lease_id, expiry = self._cache[identity]
            if time.time() < expiry - 300:  # 5 min buffer before expiry
                return user, password
            # Lease expiring soon — renew or get new
            try:
                await self._renew_lease(lease_id)
                self._cache[identity] = (user, password, lease_id,
                                         time.time() + 3600)
                return user, password
            except Exception:
                pass  # Fall through to new credential

        # Cache miss or expired — get new dynamic credentials
        async with self._lock:
            # Double-check after acquiring lock
            if identity in self._cache:
                cached = self._cache[identity]
                if time.time() < cached[3] - 300:
                    return cached[0], cached[1]

            return await self._issue_credentials(identity)

    async def _authenticate_to_vault(self) -> None:
        """Authenticate sidecar to Vault using SPIFFE x509 SVID."""
        if self._vault_token and time.time() < self._token_expiry:
            return

        async with httpx.AsyncClient(
            cert=(self._spiffe_cert, self._spiffe_key),
            verify=self._spiffe_ca,
        ) as client:
            resp = await client.post(
                f"{self._vault_addr}/v1/auth/spiffe/login",
                json={"name": f"{self._db_target}-sidecar"},
            )
            resp.raise_for_status()
            data = resp.json()["auth"]
            self._vault_token = data["client_token"]
            self._token_expiry = time.time() + data["lease_duration"]

    async def _issue_credentials(self, identity: str) -> tuple[str, str]:
        """Request new dynamic DB credentials from Vault."""
        await self._authenticate_to_vault()

        # Map identity to Vault role
        # e.g., spiffe://corp/user/matt → pg-readonly-standard
        tier = await self._resolve_tier(identity)
        vault_role = f"{self._db_target}-readonly-{tier}"

        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{self._vault_addr}/v1/database/creds/{vault_role}",
                headers={"X-Vault-Token": self._vault_token},
            )
            resp.raise_for_status()
            data = resp.json()
            username = data["data"]["username"]
            password = data["data"]["password"]
            lease_id = data["lease_id"]
            lease_ttl = data["lease_duration"]

            # Cache with TTL
            self._cache[identity] = (
                username, password, lease_id,
                time.time() + lease_ttl,
            )
            return username, password

    async def _renew_lease(self, lease_id: str) -> None:
        """Renew a Vault lease to extend credential TTL."""
        await self._authenticate_to_vault()
        async with httpx.AsyncClient() as client:
            resp = await client.put(
                f"{self._vault_addr}/v1/sys/leases/renew",
                headers={"X-Vault-Token": self._vault_token},
                json={"lease_id": lease_id, "increment": 3600},
            )
            resp.raise_for_status()
```

---

## 6. Identity → Vault Role Mapping

| Client Identity (SPIFFE URI) | ext_authz Tier | Vault Role | DB Permissions |
|---|---|---|---|
| `spiffe://corp/user/matt` | `premium` | `pg-readonly-premium` | `SELECT` on all tables, 4h TTL |
| `spiffe://corp/user/matt` | `premium` | `ch-readonly-premium` | `readonly=1`, 4h TTL |
| `spiffe://corp/svc/etl-pipeline` | `standard` | `mssql-readonly-standard` | `db_datareader`, 1h TTL |
| `spiffe://corp/user/intern-bob` | `standard` | `pg-readonly-standard` | `SELECT` on whitelisted tables, 1h TTL |

The tier lookup is done by the sidecar calling the same entitlements registry that ext_authz uses (Redis-cached). This keeps the sidecar lightweight — it just needs the tier to pick the right Vault role.

---

## 7. Credential Lifecycle

```
  t=0     Sidecar requests creds from Vault for identity "matt"
  t=0     Vault → DB: CREATE ROLE "v-sidecar-matt-abc123" WITH PASSWORD '...' VALID UNTIL t+1h
  t=0     Vault → Sidecar: {username, password, lease_id, ttl=3600s}
  t=0     Sidecar caches creds, opens DB connection pool

  t=55m   Sidecar detects lease expiring in <5min
  t=55m   Sidecar → Vault: renew lease (increment=3600s)
  t=55m   Vault extends lease to t+1h55m (if within max_ttl)

  t=4h    max_ttl reached — Vault refuses renewal
  t=4h    Vault → DB: DROP ROLE "v-sidecar-matt-abc123"
  t=4h    Sidecar evicts from cache, requests new creds on next query
  t=4h    Vault → DB: CREATE ROLE "v-sidecar-matt-def456" ...
```

---

## 8. Failure Modes

| Scenario | Behavior | Impact |
|---|---|---|
| **Vault unreachable** | Serve from credential cache until lease expires | ⚠️ Degraded — new users can't get creds, existing ones work until TTL |
| **Vault lease expires + Vault down** | ❌ Cannot get new creds → new queries fail | Existing connections remain open until DB closes them |
| **DB rejects Vault-created user** | Sidecar returns `UNAVAILABLE` | Likely misconfigured Vault creation_statements |
| **Credential cache evicted (sidecar restart)** | Re-authenticate to Vault, request new creds | ~100ms latency on first query post-restart |
| **SPIFFE cert rotation** | SPIRE rotates sidecar SVIDs automatically | Vault re-auth on next request (transparent) |

---

## 9. Security Properties

| Property | How It's Achieved |
|---|---|
| **No static DB passwords** | Vault dynamically generates credentials with TTL |
| **Per-user DB connections** | Each identity gets its own Vault-generated DB user |
| **Credential rotation** | Automatic — TTL expiry + Vault lease renewal |
| **Zero-knowledge sidecars** | Sidecars never see long-lived DB admin credentials |
| **Audit trail** | Vault audit log: who requested creds, when, for which DB |
| **Blast radius** | Compromised credential expires in ≤1h; Vault can force-revoke |
| **Sidecar auth** | SPIFFE x509 SVID — same PKI as gateway mTLS |
| **Cross-DB consistency** | Same pattern for PG, ClickHouse, SQL Server |
