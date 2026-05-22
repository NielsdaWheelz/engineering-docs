# Main-Agent Host Functions

## Scope

This document covers production host functions reachable from the main-agent `run` tool and workspace-function HTTP route.

## Host Function Families

- Host-function infrastructure lives under `src/main/server/internal/agent/host-functions/`.
- Run-tool host-function boundary types and loading live under `host-functions/run-tool.ts` and `host-functions/run-tool-loader.ts`.
- Lua exposure adapters live under `src/main/server/internal/agent/host-functions/lua/`.
- Host-function implementations live under `src/main/server/internal/agent/host-functions/library/`.
- Workspace functions live under `src/main/server/internal/agent/host-functions/library/workspace/`.
- Helper functions live under `src/main/server/internal/agent/host-functions/library/helper/`.
- Agent-state functions live under `src/main/server/internal/agent/host-functions/library/agent-state/`.
- Agent-run functions live under `src/main/server/internal/agent/host-functions/library/agent-run/`.
- Workspace functions are workspace-scoped host functions with explicit function names and optional HTTP exposure through the workspace-function route. Current workspace functions use `workspace.*` names by convention, but the host-function machinery never infers or enforces a prefix.
- Helper functions are top-level host functions such as `skill.*`, `web.*`, `bytes.*`, `text.*`, and `random.*`.
- Agent-state functions are ordinary JSON-returning `agent.*` host calls for durable agent-scoped state, such as `agent.note.*`.
- Agent-run functions are `agent.observe_value(...)`, `agent.sleep(...)`, `agent.show_message(...)`, and `agent.idle_until_input_event({ message = { blocks = ... } })`. They are host functions with a run boundary adapter because their success path returns `RunToolHostFunctionCallResult`; today these functions return no value to Lua, and `agent.idle_until_input_event(...)` additionally records an idle request.

## Boundary Contract

- `defineHostFunction` is the single production host-function primitive. Families supply context and boundary adapters; they do not own input decoding or registry identity.
- The shared host-function call method is `callWithJsonInput(...)`; family adapters decide whether the success path is ordinary JSON, run-tool control state, or another boundary result.
- Workspace functions, helper functions, and agent-state functions use the shared JSON boundary through their family-specific declaration helpers.
- Agent-run functions use the same host-function primitive with a run boundary that maps input-decode failures and intentional implementation failures to `LuaGuestError`.
- Helper and workspace function registries stay separate; `host-functions/registry.ts` is only the aggregation point for consumers that need the combined workspace/helper catalog.
- Every production host function declaration must provide its complete function name explicitly as `functionName`. Runtime lookup and error reporting use only the declared `functionName`.
- Every production JSON host-function declaration must expose explicit model-visible input, success, and declared error schemas at the declaration site. Use the shared empty-input schema for no-argument functions, `Schema.Void` for JSON `null` success, and `Schema.Never` for no declared model-visible failures.
- Workspace, helper, and agent-state function implementations fail with the internal declared-failure wrapper via family helpers such as `failWorkspaceHostFunction`, `failHelperHostFunction`, and `failAgentStateHostFunction`. Workspace and helper families also expose matching `declare*Failures` adapters for adapting an inner operation whose error channel is already a declared model-visible host-function payload; agent-state functions currently expose only `failAgentStateHostFunction`.
- Omitted raw input at the Lua or HTTP boundary is decoded as `{}` against the declared input schema. Explicit `null` and every other non-object input is rejected at the schema boundary.
- Host-function success and declared error payloads must encode to JSON without embedded NUL bytes before crossing the Lua or HTTP boundary.

## Error Shape

- Host-call wrapper errors are owned boundary envelopes and use model-visible `type` fields.
- Declared host-function errors are already model-visible before they cross the host-call boundary.
- The declared-failure wrapper is internal host-function metadata; the boundary unwraps its payload and encodes only that payload through the host function's declared error schema.
- Never recursively rewrite arbitrary JSON to replace `_tag` with `type`.
- When a production error needs both Effect `catchTag` behavior and model-visible JSON fields, declare it with `defineModelVisibleTaggedError`.
- Keep sandbox/runtime Lua errors separate from host-call wrapper errors. Lua snippet runtime details can contain arbitrary JSON and must pass through unchanged.

## Lua Exposure

- Production Lua exposure flows through:

```text
HostFunction -> RunToolHostFunction -> LuaHostFunctionBinding
```

- Raw `LuaHostFunctionBinding` definitions are for sandbox internals, tests, and examples, not production workspace/helper/agent host-function declarations.
- Sandbox utilities that validate Lua function names may keep literal `LuaFunction` terminology because they refer to VM-level Lua functions.

## Durable Semantics

- Pure Lua may replay after a crash; durable effects must cross a managed host-call boundary as `Flow.call(...)` operations.
- Replay-safe nondeterminism exposed to the model, such as `random.*`, is a production host function because it crosses the durable host boundary.
- Unexpected host defects must defect. Only intentional guest-level failures should become catchable Lua errors.
- Idle requests are evaluated after `run` returns. Successful runs apply the idle request only when no unread waking context events arrived first; failed runs append `run_failed` and, when an idle request had already been accepted, `idle_request_not_applied`.
