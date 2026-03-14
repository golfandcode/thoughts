# Part 2: ext_authz & Identity Service — Design Document

> **Component:** Python gRPC External Authorization Service  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md)  
> **Consumed by:** [Part 1: Envoy Gateway](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part1_envoy_gateway.md) (as `ext_authz` filter backend)

---

## 1. Responsibilities

- Extract client identity from mTLS certificate SAN (SPIFFE URI / CN)
- Validate the identity is registered and not revoked
- Determine the identity's entitlements: allowed targets, tier, limits
- Inject `x-client-identity`, `x-rate-tier`, and limit ceiling headers
- Cache identity→entitlement lookups to minimize latency
- Return `DENY` for unknown, revoked, or unauthorized identities

> [!IMPORTANT]
> **No API key.** The client cert SAN is the sole identity credential. **No DB credentials injected.** Sidecars look up per-user DB credentials from Vault using the identity.

---

## 2. Architecture

```
┌──────────┐     gRPC      ┌─────────────────────────────┐
│  Envoy   │──────────────▶│  ext_authz Service          │
│          │◀──────────────│  (Python asyncio + grpcio)   │
│          │  CheckResponse│                              │
│ Extracts │               │  ┌─────────────────────┐    │
│ XFCC hdr │               │  │ Layer 1: LRU Cache   │    │
│          │               │  │ TTL: 30s, 10K entries│    │
│          │               │  └─────────┬───────────┘    │
│          │               │            │ miss           │
│          │               │  ┌─────────▼───────────┐    │
│          │               │  │ Layer 2: Redis       │    │
│          │               │  │ TTL: 5min            │    │
│          │               │  └─────────┬───────────┘    │
│          │               │            │ miss           │
│          │               │  ┌─────────▼───────────┐    │
│          │               │  │ Layer 3: Entitlement │    │
│          │               │  │ Registry (Vault/DB)  │    │
│          │               │  └─────────────────────┘    │
│          │               └─────────────────────────────┘
└──────────┘
```

---

## 3. Check Request Flow

```python
async def Check(self, request: CheckRequest, context) -> CheckResponse:
    # 1. Extract identity from Envoy's XFCC header (set by mTLS)
    xfcc = self._get_header(request, "x-forwarded-client-cert")
    target_db = self._get_header(request, "x-target-db")

    # 2. Parse SPIFFE URI from cert SAN
    identity = self._parse_spiffe_from_xfcc(xfcc)
    if not identity:
        return self._deny(Code.UNAUTHENTICATED, "NO_CLIENT_CERT",
            "No valid client certificate presented")
    if not target_db:
        return self._deny(Code.INVALID_ARGUMENT, "MISSING_TARGET_DB")

    # 3. Lookup entitlements: LRU → Redis → Registry
    entitlement = await self._resolve(identity)
    if not entitlement:
        return self._deny(Code.UNAUTHENTICATED, "UNKNOWN_IDENTITY",
            f"Identity '{identity}' is not registered")
    if entitlement.revoked:
        return self._deny(Code.PERMISSION_DENIED, "IDENTITY_REVOKED")

    # 4. Verify target_db is allowed for this identity
    if target_db not in entitlement.allowed_targets:
        return self._deny(Code.PERMISSION_DENIED, "TARGET_NOT_ALLOWED",
            f"'{identity}' cannot access target '{target_db}'")

    # 5. Build OK response — inject identity + tier, NOT db creds
    return self._allow(
        headers={
            "x-client-identity": identity,
            "x-rate-tier": entitlement.tier,
            "x-max-rows-ceiling": str(TIER_LIMITS[entitlement.tier].max_rows),
            "x-max-timeout-ceiling": str(TIER_LIMITS[entitlement.tier].timeout),
        }
    )
```

---

## 4. Data Model

```python
@dataclass(frozen=True)
class IdentityEntitlement:
    identity: str                              # SPIFFE URI (e.g., "spiffe://corp/user/matt")
    tier: str                                  # "standard" | "premium"
    allowed_targets: frozenset[str]            # {"pg", "clickhouse", "mssql"}
    revoked: bool                              # True if identity has been revoked
    created_at: datetime
    expires_at: datetime | None
```

> [!IMPORTANT]
> **DB credentials are NOT in the entitlement record.** Each sidecar independently looks up the user's per-DB credentials from Vault using the `x-client-identity`. This ensures:
> - The DB connection is made AS the user (preserving RBAC, RLS, audit)
> - ext_authz never handles or transmits DB passwords
> - Credential rotation is per-user, per-DB, independent of the gateway

---

## 5. Entitlement Registry

```python
async def _registry_lookup(self, identity: str) -> IdentityEntitlement | None:
    """Lookup identity entitlements from Vault or entitlement DB."""
    identity_hash = hashlib.sha256(identity.encode()).hexdigest()
    record = await self._vault_client.read(
        f"secret/data/gateway/identities/{identity_hash}"
    )
    if not record:
        return None

    data = record["data"]["data"]
    return IdentityEntitlement(
        identity=identity,
        tier=data["tier"],
        allowed_targets=frozenset(data["allowed_targets"]),
        revoked=data.get("revoked", False),
        created_at=datetime.fromisoformat(data["created_at"]),
        expires_at=data.get("expires_at"),
    )
```

---

## 6. Caching Strategy

| Layer | Store | TTL | Capacity | Invalidation |
|---|---|---|---|---|
| **L1** | In-process `cachetools.TTLCache` | 30 sec | 10,000 entries | TTL expiry only |
| **L2** | Redis cluster | 5 min | Unbounded | TTL + explicit `DEL` on revocation |
| **L3** | Vault / Entitlement DB | N/A | Source of truth | Identity rotation or revocation |

**Cache key format:** `SHA-256(spiffe_uri)` — identity hash, not raw URI.

**Why L1 exists:** Eliminates Redis RTT (~1ms) for frequently seen identities. 30s TTL means a revoked identity is blocked within 30s.

---

## 7. Failure Modes

| Scenario | Behavior | Envoy config |
|---|---|---|
| ext_authz unreachable | **Deny all requests** | `failure_mode_allow: false` |
| Redis unreachable | Fall through to Vault (slower, ~50ms) | — |
| Vault unreachable | Serve from L1/L2 cache + log alert | — |
| L1 + L2 miss + Vault down | **Deny** — cannot verify identity | — |
| ext_authz latency >50ms | Envoy times out, returns 403 | `timeout: 50ms` |

> [!CAUTION]
> `failure_mode_allow: false` means **if this service goes down, the entire gateway stops accepting new requests**. This is correct zero-trust behavior. Mitigate with 3+ replicas and independent L1 caches.

---

## 8. Deployment

| Parameter | Value |
|---|---|
| Replicas | 3+ (behind Envoy cluster with outlier detection) |
| CPU | 1 vCPU per pod |
| Memory | 512 MB per pod |
| Language | Python 3.12+ with `grpcio[aio]` |
| Dependencies | `grpcio`, `cachetools`, `aioredis`, `hvac` (Vault client) |
| gRPC port | 50060 |
| Health check | gRPC health protocol (standard) |
