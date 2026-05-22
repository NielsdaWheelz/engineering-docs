# Cloud Tunnel

## Scope

This document covers the cloud tunnel subsystem under `src/cloud/server/internal/tunnel/`: the shared authorize route, the HTTP endpoint feature, the private TCP tunnel feature, the public TCP tunnel feature, the relay feature, and the tunnel-server and relay-node setup entrypoints.

## Shared Module

- The shared tunnel module lives under `src/cloud/server/internal/tunnel/`.
- `routes.ts` exports `TunnelRoutesLayer`, which merges the shared authorize route with the HTTP endpoint, private TCP tunnel, public TCP tunnel, and relay route layers and is mounted by `src/cloud/server/routes.ts`.
- `paths.ts` defines `TUNNEL_FRP_AUTHORIZE_PATH` at `/tunnel/frp/authorize`.
- Feature-local `*/tenant-service.ts` services own tenant-facing tunnel read and mutation semantics for HTTP endpoints, private TCP tunnels, public TCP tunnels, and relays. `rpc-handlers.ts` adapts RPC current-tenant context into those services, executes operation handles with RPC replay namespaces, and does not own tunnel table joins or lifecycle transitions.
- `authorize-route.ts` validates FRP `Login`, `NewProxy`, `Ping`, and `NewWorkConn` requests on that shared authorize path.
- HTTP endpoint authorize requests parse the HTTP endpoint tunnel token, load the published `cloud_http_endpoint` state row joined through `resource_id` to `cloud_http_endpoint_resource`, require the claimed domain to match the stored HTTP endpoint domain, and require the client to be connected to the assigned tunnel server.
- Private TCP tunnel authorize requests parse the sealed tunnel handle, require the referenced `cloud_private_tcp_tunnel` row to exist on the connected tunnel server, branch by `tunnel_role`, forbid consumer `NewProxy`, and on provider `NewProxy` require `proxy_type = "stcp"`, `proxy_name = private_tcp_tunnel_handle`, and the presented shared secret to resolve to the same tunnel id.
- Public TCP tunnel authorize requests parse the sealed tunnel handle, require the referenced `cloud_public_tcp_tunnel` state row joined through `resource_id` to `cloud_public_tcp_tunnel_resource` to exist on the connected tunnel server, require the provider token to resolve to the same tunnel resource, and on provider `NewProxy` require `proxy_type = "tcp"`, `proxy_name = public_tcp_tunnel_handle`, and `remote_port` to match the stored public port.
- Relay authorize requests parse the sealed relay resource handle used by FRP and branch by `tunnel_role`. Provider authentication uses only the underlying `cloud_relay_resource` row so relay-node setup can connect before tenant publication; consumer authentication also requires the published tenant `cloud_relay` row joined through `resource_id` to that resource. Consumer `NewProxy` is forbidden, and provider `NewProxy` requires `proxy_type = "stcp"`, `proxy_name = relay_resource_handle`, and the presented shared secret to resolve to the same relay resource.
- The authorize route is tenant-agnostic because FRPS does not carry tenant session context. Any machine possessing a valid private TCP tunnel or relay resource handle plus the matching `providerToken` or STCP `sharedSecret` can participate in that tunnel, so those outward values must be treated as the authorization boundary.
- `internal/shell-script-route-layer.ts` and `internal/compile-setup-script-shared.ts` hold the shared script-serving and FRP install helpers.
- `internal/frp.ts` defines the shared FRP version and install paths.

## HTTP Endpoint

- The HTTP endpoint feature lives under `src/cloud/server/internal/tunnel/http-endpoint/`.
- It gives cloud-managed machines public HTTP endpoint access through their assigned tunnel server for managed direct subdomains under `HTTP_ENDPOINT_WILDCARD_BASE_DOMAIN`.
- `src/cloud/http-endpoint-base-domain.ts` reads `HTTP_ENDPOINT_WILDCARD_BASE_DOMAIN` and validates that the configured base domain can host managed direct subdomains.
- `src/cloud/web/http-endpoint-domain.ts` defines canonical lowercase HTTP endpoint domain parsing and direct-subdomain checks.
- `src/cloud/web/http-endpoint-domain-policy.ts` defines the shared managed-domain policy rules, including reserved labels and reserved brand substrings such as `codapt` and `solid`.
- `src/cloud/tunnel-tokens.ts` defines generated HTTP endpoint tunnel-token parsing. `cloud_http_endpoint_resource` stores the raw token for tenant-facing RPC responses and a verifier hash for shared tunnel authorization.
- Tenant HTTP endpoint RPCs check domain availability against `cloud_http_endpoint_resource`, derive `HttpEndpointInfo` from the published `cloud_http_endpoint` state row joined to its resource row, and reject registrations outside the managed direct-subdomain policy or under reserved labels. The Cloud web tenant RPCs also expose the base domain for the tenant UI.
- HTTP endpoint registration and deletion run as durable operations that pair DB visibility changes with a full Caddy projection deploy to all tunnel servers. Registration first writes `cloud_http_endpoint_resource` plus a `cloud_http_endpoint_registration` state row pointing to it so the reserved domain is included in uniqueness checks and the deploy projection while the tunnel token is durably tied to the resource; after deploy succeeds, the resource is published by inserting a `cloud_http_endpoint` state row pointing to the same resource and deleting the registration row. Deletion removes the published row before redeploying Caddy, then deletes the resource row so a failed undeploy leaves an allocation hold instead of allowing reuse.
- `paths.ts` defines the client setup and teardown script endpoints under `/http-endpoint/client/`.
- `routes.ts` exports `HttpEndpointTunnelRoutesLayer`, which merges those two script routes into the shared tunnel route surface.
- `client/setup-script-route.ts` compiles the generic script once during layer construction and serves it as `text/x-shellscript`.
- The setup script is compiled from `<tunnel_server_host> <http_endpoint_domain> <http_endpoint_tunnel_token> <bind_host> <bind_port>`, installs `frpc` into a codapt-owned shared binary path, writes `/etc/codapt/managed-services/<install-id>/frpc.toml`, writes `/etc/codapt/managed-services/<install-id>/exec.sh`, and registers one managed service through the shared manager layer.
- The managed service chooses `systemd` on Linux or `launchd` on macOS, records that choice in `/etc/codapt/managed-services/<install-id>/manifest.sh`, and points the backend at the codapt-owned `exec.sh` wrapper in that install dir.
- The teardown script derives the same install id from `http_endpoint_domain`, removes the managed service through the shared manager layer, and deletes the codapt-owned install directory.
- `maintenance.ts` deletes stale leaked `_acme-challenge` TXT records under `HTTP_ENDPOINT_WILDCARD_BASE_DOMAIN` after a one-hour grace period.

## Private TCP Tunnel

- The private TCP tunnel feature lives under `src/cloud/server/internal/tunnel/private-tcp-tunnel/`.
- It gives cloud-managed machines private TCP connectivity through their assigned tunnel server.
- `cloud_private_tcp_tunnel` persists tenant-owned private TCP tunnel rows.
- `CloudHandles` derives `privateTcpTunnelHandle` from the row id. Generated `providerToken` and `sharedSecret` credentials are stored on the tunnel row for setup/get/list responses, with hash verifier columns used by the shared authorize route.
- Tenant private TCP tunnel RPCs derive and return those outward values in `PrivateTcpTunnelInfo`.
- `paths.ts` defines four script endpoints under `/private-tcp-tunnel/`: provider setup and teardown plus consumer setup and teardown.
- `routes.ts` exports `PrivateTcpTunnelRoutesLayer`, which merges those four script routes into the shared tunnel route surface.
- The setup routes compile generic scripts once during layer construction and serve them as `text/x-shellscript`.
- The provider setup script is compiled from `<tunnel_server_host> <private_tcp_tunnel_handle> <provider_token> <shared_secret> <local_host> <local_port>`, installs `frpc`, writes `/etc/codapt/managed-services/<install-id>/frpc.toml`, writes `/etc/codapt/managed-services/<install-id>/exec.sh`, and registers one managed service through the shared manager layer.
- The consumer setup script is compiled from `<tunnel_server_host> <private_tcp_tunnel_handle> <shared_secret> <bind_addr> <bind_port>`, installs `frpc`, writes `/etc/codapt/managed-services/<install-id>/frpc.toml`, writes `/etc/codapt/managed-services/<install-id>/exec.sh`, and registers one managed service through the shared manager layer.
- Provider and consumer install ids are short stable hashes derived from the tunnel handle, with the consumer id additionally incorporating `bind_port`.
- Provider and consumer teardown scripts derive the same install ids, remove the managed service through the shared manager layer, and delete the codapt-owned install directory.
- Private TCP tunnels reuse the shared authorize route and have no separate tunnel-server provisioning entrypoint.

## Public TCP Tunnel

- The public TCP tunnel feature lives under `src/cloud/server/internal/tunnel/public-tcp-tunnel/`.
- It gives workspace agents a public raw TCP endpoint on the assigned tunnel server that forwards to a TCP service reachable from the provider machine.
- `cloud_public_tcp_tunnel_resource` persists the tenant-owned public TCP tunnel allocation with assigned tunnel server, port, and provider token. `cloud_public_tcp_tunnel_reservation`, `cloud_public_tcp_tunnel`, and `cloud_public_tcp_tunnel_release` are narrow state rows that point to the resource for Caddy-visible reservation, tenant-visible publication, and post-delete port drain. Publishing inserts the tenant-visible row and deletes the reservation row in the same transaction.
- `CloudHandles` derives the tenant-facing `publicTcpTunnelHandle` from the published tunnel row id. Generated `providerToken` credentials are stored on the resource row for setup/get/list responses, with a hash verifier column used by the shared authorize route.
- Tenant public TCP tunnel RPCs create, delete, get, and list public TCP tunnels. Creating a tunnel reserves the port in `cloud_public_tcp_tunnel_resource`, deploys the Caddy projection with a reservation marker that points to the resource, then publishes the tenant tunnel marker that points to the same resource. Deleting a tunnel removes the published marker before redeploying the projection so fresh tenant operations stop observing the tunnel immediately, then inserts a release row that keeps the resource and port allocated until FRPS/Caddy has had time to drain the old provider proxy. Future public TCP allocation lazily reclaims expired release rows before selecting an available port.
- `paths.ts` defines provider setup and teardown script endpoints under `/public-tcp-tunnel/`.
- `routes.ts` exports `PublicTcpTunnelRoutesLayer`, which merges those two script routes into the shared tunnel route surface.
- The provider setup script is compiled from `<tunnel_server_host> <public_tcp_tunnel_handle> <provider_token> <public_port> <local_host> <local_port>`, installs `frpc`, writes `/etc/codapt/managed-services/<install-id>/frpc.toml`, writes `/etc/codapt/managed-services/<install-id>/exec.sh`, and registers one managed service through the shared manager layer. Its FRPC config uses a `tcp` proxy with `remotePort` fixed to the cloud-assigned public port.
- The provider teardown script derives the same install id from the public TCP tunnel handle, removes the managed service through the shared manager layer, and deletes the codapt-owned install directory.
- Public TCP tunnels reuse the shared authorize route and have no consumer setup command; outside users connect directly to the returned `binding.host:binding.port` endpoint.

## Relay

- The relay feature lives under `src/cloud/server/internal/tunnel/relay/`.
- It gives tenant workspaces SOCKS5 egress through operator-configured relay nodes on business-network egress hosts.
- `cloud_relay_node` persists operator-owned relay nodes with JSON specs that tenant agents can discover and select. `cloud_relay_resource` persists the tenant-owned provider-facing relay resource, including tenant id, requested spec, assigned tunnel server, relay node, provider token, shared secret, and hash verifier columns. `cloud_relay` is the tenant-visible publication row that points to a relay resource with an explicit `resource_id`.
- `CloudHandles` derives `relayResourceHandle` from the relay resource identity for FRP provider and consumer proxy setup, and derives tenant-facing `relayHandle` from the published relay row identity for tenant get/delete/list paths. Generated `providerToken` is stored for provider setup, and generated `sharedSecret` is stored for setup/get/list responses, with hash verifier columns used by the shared authorize route.
- Tenant relay RPCs list available relay-node specs and create, delete, get, and list relays. Creating a relay chooses one tunnel server and one relay node matching the requested spec, creates the underlying relay resource, installs a managed FRPC STCP provider with the built-in `socks5` plugin on the relay node, then publishes the tenant-visible relay row. Tenant get/list only read published relays. Deleting a relay removes publication first, tears down the relay-node provider, then deletes the underlying relay resource.
- `paths.ts` defines four script endpoints under `/relay/`: provider setup and teardown plus consumer setup and teardown.
- `routes.ts` exports `RelayRoutesLayer`, which merges those four script routes into the shared tunnel route surface.
- The setup routes compile generic scripts once during layer construction and serve them as `text/x-shellscript`.
- The provider setup script is compiled from `<tunnel_server_host> <relay_resource_handle> <provider_token> <shared_secret>`, installs `frpc`, writes `/etc/codapt/managed-services/<install-id>/frpc.toml`, writes `/etc/codapt/managed-services/<install-id>/exec.sh`, and registers one managed service through the shared manager layer. Its FRPC config uses an STCP proxy whose local service is the FRP `socks5` client plugin.
- The consumer setup script is compiled from `<tunnel_server_host> <relay_resource_handle> <shared_secret> <bind_addr> <bind_port>`, installs `frpc`, writes `/etc/codapt/managed-services/<install-id>/frpc.toml`, writes `/etc/codapt/managed-services/<install-id>/exec.sh`, registers one managed service through the shared manager layer, and prints the local `socks5h://<bind_addr>:<bind_port>` URL.
- Provider and consumer install ids are short stable hashes derived from the relay resource handle, with the consumer id additionally incorporating `bind_port`.
- Provider and consumer teardown scripts derive the same install ids, remove the managed service through the shared manager layer, and delete the codapt-owned install directory.
- Relays reuse the shared authorize route and require separate relay-node provisioning before tenant relay creation can succeed.

## Tunnel Server

- `tunnel-server/` holds the shared server-side tunnel implementation details.
- `tunnel-server/internal/setup-command.ts` renders one bootstrap script that installs FRPS, builds Caddy with the DigitalOcean DNS and layer-4 plugins, writes the managed FRPS and Caddy files, validates Caddy, and restarts Caddy and FRPS.
- Generated Caddy config treats `:443` as the external TLS router and `:8443` as the peer HTTPS terminator. External TLS routes send every known endpoint and tunnel-server hostname to the owner `private_ip_address:8443` with PROXY protocol v2. The peer HTTPS terminator binds only to the local `private_ip_address` and requires PROXY before TLS so HTTP handling sees the original client IP.
- Generated Caddy config also renders public TCP listeners for the local tunnel server's assigned public TCP tunnel ports. Each listener binds on the tunnel server public hostname and proxies raw TCP to the matching loopback FRPS proxy port.
- The FRPS config uses the shared authorize path plus the local tunnel server `public_hostname` as an HTTP plugin path for `Login`, `NewProxy`, `Ping`, and `NewWorkConn`.
- The FRPS config keeps proxy listeners on loopback and restricts client-requested public TCP proxy ports to the configured public TCP tunnel port range.
- The setup flow reads stable cloud-wide config from `DIGITALOCEAN_API_TOKEN`, `HTTP_ENDPOINT_WILDCARD_BASE_DOMAIN`, and `CLOUD_API_ORIGIN`. The specific tunnel server being provisioned is supplied with `--public-hostname`, `--private-ip-address`, and `--executor-worker-token`.
- `tunnel-server/setup.ts` exposes the setup effect and the command-execution durable operation catalog used by setup and Caddy deploys.

## Operations

- `bun run cloud:server:maintenance` runs the HTTP endpoint stale `_acme-challenge` cleanup task.
- `src/cloud/server/bin/tunnel-server-setup.ts` is the operator entrypoint for provisioning the tunnel server.
- It delegates to `tunnel-server/setup.ts`, generates a fresh replay key, executes one durable setup attempt under the replay-aware tunnel config lock, prepares a replay-stable tunnel-server id when the row does not exist yet, defects if an existing row has a different private IP or remote setup command fails, runs setup through the target executor worker, then inserts or refreshes the tunnel-server row after setup succeeds.
- Run it from a control-plane host with the target tunnel server values:
  `bun run cloud:server:tunnel-server:setup -- --public-hostname tunnel-1.dc.tk.co --private-ip-address 10.20.0.1 --executor-worker-token ...`.
- `bun run cloud:server:background` must already be running for tunnel-server setup.
- Each tunnel-server setup invocation starts a fresh durable setup attempt.
- If the tunnel-server setup CLI process dies after the durable operation is persisted, the background worker can finish or dead-letter that attempt. Rerunning the command later starts a new attempt for the same `public_hostname` row.
- `src/cloud/server/bin/relay-node-setup.ts` is the operator entrypoint for registering a relay node.
- It delegates to `relay/setup.ts`, waits for executor worker readiness, generates a fresh replay key, then inserts a `cloud_relay_node` row for a new worker token or refreshes the spec for the same worker token.
- Run it from a control-plane host with the relay node values:
  `bun run cloud:server:relay-node:setup -- --spec-json '{"network_class":"business","country":"US"}' --executor-worker-token ...`.
- `src/cloud/server/bin/dev-relay-node-setup.ts` is the development wrapper around the same setup effect. It reads `DEV_RELAY_NODE_EXECUTOR_WORKER_TOKEN` and `DEV_RELAY_NODE_SPEC_JSON`; the default development spec is `{"location":"new-york"}`.
- `dev:e2e:provision-infra` waits for a running dev stack and reruns both dev tunnel-server setup and dev relay-node setup without resetting the database.
- `dev:e2e:reset-and-serve` resets the database, seeds app state, starts the dev stack, waits for readiness, then runs `dev:e2e:provision-infra` behavior so database resets recreate the relay catalog entry for the persistent development relay node. It stays alive as the dev stack supervisor.
