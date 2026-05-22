# Services

## Scope

This document covers the backend services produced by the repo and the shared infrastructure they compose.

## Main Service

- The main service lives in `src/main/`.
- It owns user-facing functionality: executor, credits, and the main app and admin web UIs.
- Entrypoints live in `src/main/server/bin/`.
- API env vars are `MAIN_API_BIND_PORT`, `MAIN_API_BIND_HOST`, and `MAIN_API_ORIGIN`.
- Main web origin env vars are `MAIN_APP_WEB_ORIGIN` and `MAIN_ADMIN_WEB_ORIGIN`.
- Main web dev env vars are `MAIN_APP_WEB_BIND_PORT`, `MAIN_APP_WEB_BIND_HOST`, `MAIN_ADMIN_WEB_BIND_PORT`, and `MAIN_ADMIN_WEB_BIND_HOST`.

## Cloud Service

- The cloud service lives in `src/cloud/`.
- It owns cloud infrastructure management: tenants, VM provisioning, HTTP endpoint/domain management, private TCP tunnels, relays, and the cloud admin and tenant web UIs.
- See [tunnel.md](tunnel.md) for the cloud tunnel subsystem, including the shared authorize route, HTTP endpoint, private TCP tunnel, relay features, and the tunnel-server and relay-node setup entrypoints.
- Entrypoints live in `src/cloud/server/bin/`.
- API env vars are `CLOUD_API_BIND_PORT`, `CLOUD_API_BIND_HOST`, and `CLOUD_API_ORIGIN`.
- Tunnel server setup env vars are documented in [tunnel.md](tunnel.md).
- Cloud web origin env vars are `CLOUD_ADMIN_WEB_ORIGIN` and `CLOUD_TENANT_WEB_ORIGIN`.
- Cloud web dev env vars are `CLOUD_ADMIN_WEB_BIND_PORT`, `CLOUD_ADMIN_WEB_BIND_HOST`, `CLOUD_TENANT_WEB_BIND_PORT`, `CLOUD_TENANT_WEB_BIND_HOST`.

## Shared Infrastructure

- Shared reusable modules live under `src/shared/`.
- `src/shared/framework/` is the lowest-level reusable substrate: coordination, DB families, retry, RPC, serialization, setup, process helpers, and web substrate.
- `src/shared/server/db/` is the shared primary DB owner for persistent reusable-module tables such as auth, primary coordination, short key, and short handle tables.
- Shared semantic modules such as `src/shared/auth/`, `src/shared/executor/`, `src/shared/executor-process-specs/`, and `src/shared/short-key/` remain outside `framework/` and are composed where a service needs them.
- `CoordinationTransientDb` stays with coordination under `src/shared/framework/coordination/transient/db/` because it is a backend detail for the transient coordination runtime, not a shared primary schema owner.
