# Part 1: Envoy Gateway — Design Document

> **Component:** Envoy Proxy (Data Plane)  
> **Depends on:** [Part 0: Interface Contract](file:///Users/matt/.gemini/antigravity/brain/e9320286-501d-4c6b-b88e-eee0f36d38cc/part0_interface_contract.md)  
> **Consumed by:** All sidecars, Python/TS SDK (indirectly)

---

## 1. Responsibilities

| What Envoy Does | What Envoy Does NOT Do |
|---|---|
| mTLS termination + client cert validation | Query execution |
| SPIFFE/SAN identity extraction | SQL parsing or inspection |
| ext_authz callout (cert SAN → identity validation) | Result serialization |
| gRPC routing by `x-target-db` metadata | Connection pooling to databases |
| Global rate limiting (RLS callout) | Format conversion |
| Circuit breaking + outlier detection | Direct database connections |
| Access log streaming (ALS) | — |
| Sensitive header stripping | — |

---

## 2. Listener Architecture

Single gRPC listener with filter-chain routing. All clients speak gRPC to a single port.

```yaml
static_resources:
  listeners:
    - name: gateway_listener
      address:
        socket_address: { address: 0.0.0.0, port_value: 8443 }
      per_connection_buffer_limit_bytes: 4194304  # 4 MiB
      listener_filters:
        - name: envoy.filters.listener.tls_inspector
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector
      filter_chains:
        - transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
              require_client_certificate: true
              common_tls_context:
                alpn_protocols: ["h2"]
                validation_context:
                  trusted_ca:
                    filename: "/certs/ca-chain.pem"
                  match_typed_subject_alt_names:
                    - san_type: URI
                      matcher:
                        prefix: "spiffe://gateway.example.com/"
                tls_certificates:
                  - certificate_chain: { filename: "/certs/server.crt" }
                    private_key: { filename: "/certs/server.key" }
          filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: gateway_grpc
                codec_type: HTTP2
                http2_protocol_options:
                  initial_stream_window_size: 1048576    # 1 MiB
                  initial_connection_window_size: 4194304 # 4 MiB
                  max_concurrent_streams: 128
                forward_client_cert_details: SANITIZE_SET
                set_current_client_cert_details:
                  uri: true
                  subject: true
                access_log:
                  - name: envoy.access_loggers.http_grpc
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.grpc.v3.HttpGrpcAccessLogConfig
                      common_config:
                        log_name: "gateway_access"
                        grpc_service:
                          envoy_grpc: { cluster_name: als_cluster }
                        transport_api_version: V3
                      additional_request_headers_to_log:
                        - "x-target-db"
                        - "x-request-id"
                        - "x-forwarded-client-cert"
                http_filters:
                  # 1. ext_authz — validate client cert identity, inject tier/limits
                  - name: envoy.filters.http.ext_authz
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
                      grpc_service:
                        envoy_grpc: { cluster_name: ext_authz_cluster }
                        timeout: 50ms
                      transport_api_version: V3
                      failure_mode_allow: false
                      clear_route_cache: true
                      status_on_error: { code: 403 }
                  # 2. RLS — multi-dimension rate limiting
                  - name: envoy.filters.http.ratelimit
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
                      domain: gateway
                      rate_limit_service:
                        grpc_service:
                          envoy_grpc: { cluster_name: rls_cluster }
                        transport_api_version: V3
                      failure_mode_deny: true
                  # 3. Router
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
                route_config:
                  name: gateway_routes
                  response_headers_to_remove:
                    - "x-client-identity"
                    - "x-rate-tier"
                    - "x-max-rows-ceiling"
                    - "x-max-timeout-ceiling"
                  virtual_hosts:
                    - name: gateway
                      domains: ["*"]
                      routes:
                        # Route by x-target-db header to sidecars
                        - match:
                            prefix: "/gateway.v1.DataGateway/"
                            headers:
                              - name: "x-target-db"
                                string_match: { exact: "pg" }
                          route:
                            cluster: sidecar_pg
                            timeout: 620s  # tier max + buffer
                          request_headers_to_add:
                            - header: { key: "x-request-id", value: "%REQ(x-request-id)%" }
                        - match:
                            prefix: "/gateway.v1.DataGateway/"
                            headers:
                              - name: "x-target-db"
                                string_match: { exact: "clickhouse" }
                          route:
                            cluster: sidecar_ch
                            timeout: 620s
                        - match:
                            prefix: "/gateway.v1.DataGateway/"
                            headers:
                              - name: "x-target-db"
                                string_match: { exact: "mssql" }
                          route:
                            cluster: sidecar_mssql
                            timeout: 620s
                        # Fallback: reject unknown targets
                        - match: { prefix: "/" }
                          direct_response:
                            status: 400
                            body: { inline_string: '{"error": "Missing or invalid x-target-db header"}' }
```

---

## 3. Cluster Definitions

```yaml
clusters:
  # --- Sidecar clusters ---
  - name: sidecar_pg
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options:
      initial_stream_window_size: 1048576
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 128
          max_pending_requests: 256
          max_requests: 512
    outlier_detection:
      consecutive_local_origin_failure: 3
      base_ejection_time: 15s
      max_ejection_percent: 33
      interval: 5s
    health_checks:
      - timeout: 2s
        interval: 10s
        unhealthy_threshold: 3
        healthy_threshold: 2
        grpc_health_check: {}
    load_assignment:
      cluster_name: sidecar_pg
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: sidecar-pg.internal, port_value: 50051 }

  - name: sidecar_ch
    # Same pattern as sidecar_pg, different address
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options:
      initial_stream_window_size: 1048576
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 128
          max_pending_requests: 512
          max_requests: 1024
    outlier_detection:
      consecutive_local_origin_failure: 3
      base_ejection_time: 15s
      max_ejection_percent: 33
    health_checks:
      - timeout: 2s
        interval: 10s
        grpc_health_check: {}
    load_assignment:
      cluster_name: sidecar_ch
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: sidecar-ch.internal, port_value: 50052 }

  - name: sidecar_mssql
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options:
      initial_stream_window_size: 1048576
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 64
          max_pending_requests: 128
          max_requests: 256
    outlier_detection:
      consecutive_local_origin_failure: 3
      base_ejection_time: 30s
      max_ejection_percent: 33
    health_checks:
      - timeout: 2s
        interval: 10s
        grpc_health_check: {}
    load_assignment:
      cluster_name: sidecar_mssql
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: sidecar-mssql.internal, port_value: 50053 }

  # --- Support service clusters ---
  - name: ext_authz_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    circuit_breakers:
      thresholds:
        - max_connections: 32
          max_requests: 256
    load_assignment:
      cluster_name: ext_authz_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: ext-authz.internal, port_value: 50060 }

  - name: rls_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: rls_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: rls-service.internal, port_value: 50070 }

  - name: als_cluster
    type: STRICT_DNS
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: als_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: als-sink.internal, port_value: 50080 }
```

---

## 4. RLS Rate Limit Actions

Applied per-route, three dimensions simultaneously:

```yaml
rate_limits:
  # Dim 1: Per-identity (cert SAN, injected by ext_authz)
  - actions:
      - request_headers:
          header_name: "x-client-identity"
          descriptor_key: "client_identity"
  # Dim 2: Per-target-database global capacity
  - actions:
      - request_headers:
          header_name: "x-target-db"
          descriptor_key: "target_db"
  # Dim 3: Global gateway throughput
  - actions:
      - generic_key:
          descriptor_value: "gateway_global"
```

---

## 5. xDS Bootstrap

For dynamic configuration via the Python control plane (Part 8):

```yaml
dynamic_resources:
  lds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
        - envoy_grpc: { cluster_name: xds_cluster }
  cds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
        - envoy_grpc: { cluster_name: xds_cluster }

node:
  id: "gateway-${POD_NAME}"
  cluster: "gateway-prod"
```

---

## 6. Deployment

| Parameter | Value |
|---|---|
| Replicas | 3 (minimum) |
| CPU | 2 vCPU per pod |
| Memory | 1 GB per pod |
| HPA trigger | CPU >70% or connections >80% of `max_connections` |
| Drain strategy | `GRADUAL`, 60s timeout |
| Admin port | 9901 (internal only, not exposed) |
