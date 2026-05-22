# Transport

## Scope

This document covers transport lifecycle ownership.

## Lifecycle

- A transport layer should not control the lifecycle of application work unless that work genuinely depends on the transport being alive.
- If application work must survive transport reconnects, decouple it into a scope that outlives the transport.
- Treat the transport layer as a delivery pipe that can disconnect and reconnect without interrupting in-flight application work.

## Websocket RPC Clients

- Long-lived server RPC integrations should have one runtime service that owns the authenticated websocket protocol context.
- Domain services should construct narrow `RpcClient.make(...)` clients from that shared runtime context instead of exposing a broad merged client.
- If a call site needs a one-shot client, the client acquisition and full use should happen inside one fresh transport scope.
- Do not expose websocket protocol layers as reusable domain-service fields. Expose domain methods or a runtime-owned `makeClient(...)` function instead.
- `runRpcClient(...)`, `runRpcClientResult(...)`, and `subscribeRpcClient(...)` are only for one-shot browser calls where client acquisition and use are wrapped together.

Effect `4.0.0-beta.45` and newer route websocket RPC responses by client id
inside the protocol, so multiple narrow clients can safely share one protocol
context. Keep the shared lifetime at the runtime boundary and keep RPC call
surfaces domain-specific.
