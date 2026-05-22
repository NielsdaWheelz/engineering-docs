# Keep-Alive

## Scope

This document covers keep-alive policies.

## Policies

- Define keep-alive policies in `src/shared/framework/keep-alive-policies.ts` as categorical policies.
- Creating a keep-alive policy anywhere else requires `justify-keep-alive-policy`.
- A `KeepAlivePolicy` pairs a TTL with a derived keep-alive interval of one third of that TTL.
- `SERVER_KEEP_ALIVE` is the standard policy for server-side resources. TTL: 30s. Keep-alive: 10s.
- Keep-alive TTLs reflect shared infrastructure characteristics.
- Do not use keep-alive policies for data-retention or expiry durations.
- Data-retention and expiry durations are separate domain rules.
