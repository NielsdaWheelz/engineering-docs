# Operation Types

## Scope

This document covers the coordination operation type system, entry point and composition patterns, replay identity, and durable operations.

## Purpose

Mutating operations can be interrupted by retries, timeouts, and crashes. A later attempt must finish the same operation without double-applying side effects or drifting to a different result.

Coordination operations make this resumable. Every effectful operation in the codebase is wrapped in an operation type that declares its replay characteristics, composes safely with other operations, and enforces correctness invariants through Effect's `R` parameter at compile time.

## Context Markers

Coordination operations use a small set of compile-time markers:

- `OperationScope` -- we are inside any coordination operation.
- `MutationScope` -- mutation `Flow.call(...)` composition is permitted.
- `DurableScope` -- durable multi-step composition is permitted.

Common public aliases:

- `MutationOperationContext = OperationScope | MutationScope`
- `DurableMutationContext = MutationOperationContext | DurableScope`

These are positive capability markers. Query composition gets `OperationScope` only; mutation composition gets `MutationOperationContext`; durable workflow bodies get `DurableMutationContext`.

Framework internals also use one stronger marker:

- `MultiMutationScope` -- internal marker used to distinguish multi-mutation bodies from single-mutation bodies. It remains part of the enforcement model, but ordinary domain code should not need to mention it directly.

There is also one important negative marker:

- `LeafBodyScope` -- blocks managed `Flow.call(...)` and `.run*()` inside leaf bodies such as `SingleMutation.db(...)`, `UnreplayableQuery.db(...)`, `UnreplayableSingleMutation.external(...)`, and `SingleMutation.rpc(...)`.

Replay paths, step-key counters, and single-mutation guards are internal runtime state, not public scope markers.

Coordination wrappers may spend protocol-owned internal mutation steps, but
they must not silently change the wrapped user body's mutation-cardinality
contract. A single-mutation body stays single-mutation; a multi-mutation body
stays multi-mutation.

## Operation Types

Operation types, ordered from lightest to heaviest:

| Type | Purpose | Replayable | Edge-runnable | Side effects |
|---|---|---|---|---|
| `PureQuery` | A deterministic, side-effect-free computation | not needed | yes | none |
| `UnreplayableQuery` | A simple read | no | yes | read-only |
| `UnreplayableQueryStream` | A simple read returning a stream | no | yes | read-only (stream) |
| `UnreplayableSingleMutation` | A one-shot state change that cannot be replayed | no | yes | at most 1 |
| `UnreplayableMultiMutation` | An opaque multi-step execution with no replay key | no | yes | multiple |
| `ReplayableQuery` | A read whose result can be cached for replay | yes | yes | read-only |
| `ReplayableSingleMutation` | A single atomic state change | yes | yes | at most 1 |
| `ReplayableMultiMutation` | A multi-step workflow (multiple state changes) | yes | no | multiple |
| `DurableOperation` | A multi-step workflow with autonomous completion | yes | yes | multiple (presented as 1) |

Column definitions:

- **Replayable**: retrying with the same replay key produces the same observable result without re-applying side effects.
- **Edge-runnable**: whether the operation can be run directly at entry points.
- **Side effects**: how many independent side effects (crash-separable commits or external calls) the operation may perform. Read-only operations perform none.

## Operation Details

### `PureQuery`

A deterministic, side-effect-free computation (`R = never`). Always produces the same result for the same input. Not memoized because determinism makes it unnecessary. Does not consume a step key.

- **`PureQuery.effect(body)`** -- wraps a pure effect so it can participate in the `Flow.call(...)` pattern inside composed flows.
  - Body contains: pure computation and `Flow.call(...)` on other pure queries. No services and no I/O.

### `UnreplayableQuery`

A read that may return different results each time. The simplest operation type.

- **`UnreplayableQuery.effect(body)`** -- a read with no transaction.
  - Body contains: read-only operations and control flow.
- **`UnreplayableQuery.db(body)`** -- a read inside a SERIALIZABLE read-only transaction.
  - Body contains: read-only SQL and control flow.
- **`UnreplayableQuery.compose(flow)` / `UnreplayableQuery.gen(...)`** -- an unreplayable read built by orchestrating other operations.
  - Flow contains: `Flow.call(...)` on queries and control flow. No direct side effects.

### `UnreplayableQueryStream`

A read that returns a stream of values rather than a single result.

- **`UnreplayableQueryStream.effect(body)`** -- a read that produces a `Stream` instead of a single value.
  - Body contains: read-only operations and control flow. The setup effect yields the stream; the scope covers both the setup effect and the resulting stream.
- **`UnreplayableQueryStream.compose(flow)` / `UnreplayableQueryStream.gen(...)`** -- a stream-producing read built by orchestrating other operations.
  - Flow contains: `Flow.call(...)` on queries and control flow. No direct side effects.
- `Flow.call(queryStream)` opens the stream inside an existing managed flow and returns it as an `Effect` result.
- `Operation.runQuery(queryStream)` flattens the setup effect for edge streaming handlers.

### `UnreplayableSingleMutation`

A one-shot state change with no guarantees if interrupted. If it crashes partway through, there is no automatic recovery or replay.

- **`UnreplayableSingleMutation.db(body)`** -- a DB write with no replay or memoization.
  - Runs in a SERIALIZABLE transaction.
  - Body contains: SQL reads, writes, and control flow.
- **`UnreplayableSingleMutation.effect(body)`** -- a raw-effect unreplayable mutation.
  - Use for callback or adapter boundaries that must stay in ordinary `Effect` code but still need managed-operation capabilities already present in `R`.
  - Edge execution still enforces the same at-most-one-mutation rule as `SingleMutation.compose(...)`.
- **`UnreplayableSingleMutation.external(body)`** -- a non-DB external call such as an API call to a third-party service.
  - No transaction and no retry.
  - Strict leaf form of `UnreplayableSingleMutation.effect(...)`: provides `LeafBodyScope`, so the body cannot nest managed operations.
  - Body contains: a single external call.
- **`UnreplayableSingleMutation.compose(flow)` / `UnreplayableSingleMutation.gen(...)`** -- an unreplayable mutation built by orchestrating other operations.
  - Flow contains: `Flow.call(...)` on queries and at most one mutation `Flow.call(...)`.

### `UnreplayableMultiMutation`

An opaque unreplayable execution region that may perform multiple managed mutation calls internally but has no replay key and no resumable replay semantics.

- **`UnreplayableMultiMutation.effect(body)`** -- a raw-effect opaque unreplayable execution.
  - Use for adapter or runtime regions that must stay in ordinary `Effect` code but may call multiple managed mutations internally.
- **`UnreplayableMultiMutation.compose(flow)` / `UnreplayableMultiMutation.gen(...)`** -- an opaque unreplayable execution built from `Flow`.
  - Flow contains: `Flow.call(...)` on queries and any number of mutations.

### `ReplayableQuery`

A read whose result is stable across replays. Asking the same question twice during a retry gives the same answer.

- **`Query.effect(schemas, body)`** -- a read with exit schemas so the result can be serialized for replay.
  - Body contains: read-only operations and control flow.
- **`Query.db(schemas, body)`** -- same, inside a SERIALIZABLE read-only transaction.
  - Body contains: read-only SQL and control flow.
- **`Query.from(query, schemas)`** -- upgrades an existing `UnreplayableQuery` by adding exit schemas.
- **`Query.compose(flow)` / `Query.gen(...)`** -- a read built by orchestrating other operations.
  - Flow contains: `Flow.call(...)` on queries and control flow. No direct side effects.
- **`toctouQuery({ check, operation })`** -- a TOCTOU-safe check-act-recheck read.
  - `check`: a `ReplayableQuery` that returns `Proceed(value)` or `Return(result)`. `operation`: a factory from the proceed value to a `ReplayableQuery` that returns `Return(result)` or `Recheck`. On `Recheck`, the check is re-run.

### `ReplayableSingleMutation`

A state change that completes exactly once, even across crashes and retries.

- **`SingleMutation.db(schemas, body)`** -- a single atomic DB write.
  - Runs in a SERIALIZABLE transaction. Results are cached for crash recovery.
  - Body contains: SQL reads, writes, and control flow. No `Flow.call(...)` on managed operations.
- **`SingleMutation.idempotentExternal(unreplayableMutation, { success, error })`** -- memoizes an external or interop unreplayable mutation whose exact replay is already correct.
  - Use this when replay may issue the same external call again with identical input and that is an intended correctness property, not merely a tolerated duplicate.
  - Once the first exit is durably memoized, later replays no longer depend on provider-side idempotency retention.
  - If the process dies after the side effect happened but before the memoized exit is durably recorded, replay may execute the wrapped mutation again. That replay must still be correct.
- **`SingleMutation.rpc(fn)`** -- a mutation that crosses a service boundary via RPC.
  - Auto-generates a replay key that stays stable on replay and retries transport errors.
- **`SingleMutation.compose(flow)` / `SingleMutation.gen(...)`** -- a mutation built by orchestrating other operations.
  - Flow contains: `Flow.call(...)` on queries and at most one mutation `Flow.call(...)`.
- **`uncertainMutation(unreplayableMutation, { success, error })`** -- wraps one unreplayable transition with crash detection.
  - Keeps the wrapped mutation's normal single-mutation semantics. Returns `Confirmed<A>` on success and `Unknown` on ambiguous crash recovery.
  - Use `stabilizeMutation(...)` only when a replayable query can authoritatively prove the effect. Otherwise, inspect `Confirmed` versus `Unknown` explicitly inside a larger workflow and resolve that uncertainty before returning from the owning operation.
  - `Unknown` is an internal coordination state, not a normal product-facing outcome. Retry, reconcile, or classify a terminal modeled failure internally; if that process exhausts first, defect.
- **`unsafeMemoizeSingleMutation(unreplayableMutation, { success, error })`** -- memoizes an unreplayable single mutation's final exit value without crash-detection markers and returns a `ReplayableSingleMutation`.
  - Use this only when duplicate execution is an explicitly accepted tradeoff. If the process dies after the side effect happened but before the memoized result is durably recorded, replay may execute the wrapped body again.
- **`unsafeMemoizeMultiMutation(unreplayableMutation, { success, error })`** -- memoizes an opaque multi-mutation execution's final exit value without crash-detection markers and returns a `ReplayableMultiMutation`.
  - The helper only memoizes the final exit after the opaque region completes. A crash during the region may leave a prefix of its side effects applied. Durable recovery may run the whole region again. Use this only when partial prior execution and duplicate full-region execution are both acceptable.
  - This only changes crash semantics. It does not justify surfacing transient dependency failures directly. Retry transient failures internally when the owning operation still expects to succeed, and defect on retry exhaustion by default.
- **`uncertainExecution(execution, { success, error })`** -- wraps an `UnreplayableMultiMutation` with the same crash-detection protocol.
  - Use this for opaque adapter execution where the body may perform multiple managed mutation calls, the body cannot be made deterministic/replayable, and callers must handle `Completed<A>` versus `UnknownOnCrash`.
- **`stabilizeMutation({ query, mutation, reconcileSchedule, retrySchedule, consistency })`** -- stabilizes an uncertain mutation into a deterministic outcome.
  - `query`: a `ReplayableQuery` that checks whether the mutation's effect is already visible. `mutation`: an `UncertainMutation`. Checks whether the effect is already complete, executes if not, then retries or reconciles on `Unknown` until the outcome is proved or the retry budget is exhausted.
  - Retry or reconciliation exhaustion is a defect unless the owning operation intentionally models persistent dependency unavailability as a first-class outcome.
- **Choosing an external-mutation wrapper**
  - Use plain `retryOrDie*` around the leaf when transient provider failure should just retry and no crash-ambiguity protocol is needed.
  - Use `SingleMutation.idempotentExternal(...)` when exact replay of the same external call is already correct and local memoization is there to stop depending on the provider retaining that capability forever.
  - Use `unsafeMemoizeSingleMutation(...)` when duplicate execution of one transition is an accepted domain tradeoff and there is no worthwhile authoritative recovery path.
  - Use `uncertainMutation(...)` plus `stabilizeMutation(...)` when a replayable query can authoritatively prove the effect after an ambiguous external transition.
  - If none of those fits cleanly, the operation likely needs more explicit domain recovery logic rather than a softer error surface.
- **Mutation naming rule** -- strict mutation semantics are the default.
  - Prefer names where success means this call actually performed a new state transition.
  - A pre-existing effect becomes `MutationAlreadyComplete`; callers should catch that only when "already done" is a legitimate domain outcome.
  - If a mutation instead converges on an outcome and treats an already-existing outcome as normal rather than error, name it `ensure...`.
  - If you intentionally want convergent semantics, implement them explicitly at the domain boundary. Do not hide them behind a generic helper.
- **`linearizeSingleMutation(conflictKey, mutation)`** -- serializes concurrent single-step mutations on the same conflict key through a short-lived live lock on the shared exclusivity keyspace.
- **`toctouMutation({ check, operation })`** -- a TOCTOU-safe check-act-recheck mutation for replayable mutation contexts.
  - Same shape as `toctouQuery`, but `operation` returns a `ReplayableSingleMutation`. The first check is memoized for stable input across replays.

### `ReplayableMultiMutation`

Multiple independently committed state changes that must be driven to completion in order, but span more than one transaction or system.

- **`MultiMutation.compose(flow)` / `MultiMutation.gen(...)`** -- a multi-step workflow built by orchestrating other operations.
  - Flow contains: `Flow.call(...)` on any number of queries and mutations.
  - Use this when the work cannot honestly be one DB commit. If one SERIALIZABLE transaction is enough, keep it as `SingleMutation.db(...)`.
  - Not an edge entrypoint by itself. Use it as a reusable subworkflow inside another durable flow, or wrap it in a durable operation when you want a named recoverable edge workflow.
  - Opaque unreplayable regions cannot call `ReplayableMultiMutation`s directly, even if a `DurableScope` service leaks through the Effect context. Start a durable operation instead.
  - Deterministic interpreters or script runtimes may be modeled as replayable multi-mutations only when every durable effect crosses a managed `Flow.call(...)` host boundary and ambient runtime nondeterminism is removed or exposed through replayable operations. Resource budgets that affect interpreter control flow must also be replay-stable; do not let unmanaged wall-clock time decide whether later managed calls are reachable.
- **`memoizeMultiMutationResult(mutation, { success, error })`** -- memoizes a replayable multi-mutation's final result while leaving its child effects as ordinary replayable steps.
  - Use this when later durable steps need a stable final value from an otherwise replay-safe multi-step workflow.
  - If the process dies before the final result is memoized, replay re-runs the orchestration; already-completed child effects are recovered through their own replay keys.
- **`linearizeMultiMutation(conflictKey, mutation)`** -- serializes a durable multi-step workflow on a conflict key through the internal replay-aware exclusivity protocol.
  - Intended for correctness boundaries that must survive crash or replay, such as "read current state, perform external side effect, then persist resulting state".
  - Successful completion and typed failures release the conflict-key lease. Defects and interruptions intentionally do not release it; for durable dead-lettered work, the stuck lease is operator-visible containment rather than a state to auto-clear.

### `DurableOperation`

A multi-step workflow that will run to completion on its own, even if the original caller crashes.

- **`declareDurableOperation(options)`** -- a named, recoverable workflow declaration.
- **`implementDurableOperation(declaration, implementation)`** -- binds a declaration to the workflow implementation used by foreground callers and worker catalogs.
- **`createDurableOperation(options, implementation)`** -- convenience form for declaring and binding in one step.
  - If the process crashes mid-execution, orphaned work is picked up and replayed to completion.
  - Declaration handles support `DurableOperation.submit(payload)(declaration)` for fire-and-forget execution.
  - Implemented handles provide all three execution modes: `DurableOperation.execute(payload)(handle)` for foreground execution, `DurableOperation.submit(payload)(handle)` for fire-and-forget execution, and `Flow.call(handle, payload)` for inline composition within a parent durable operation.
  - `DurableOperation.execute(payload)(handle)` cannot be called inside a normal durable operation body. Opaque unreplayable interop regions may use it to create a child recovery boundary. `DurableOperation.submit(payload)(handle)` and `Flow.call(handle, payload)` can be used from normal durable contexts.
  - Child durable operation bodies clear any inherited opaque unreplayable marker before running, so their internal `ReplayableMultiMutation` calls execute as normal durable work.

### In-Transaction Helpers

For composing reads and writes within a `SingleMutation.db(...)`, `UnreplayableQuery.db(...)`, or `UnreplayableSingleMutation.db(...)` body:

- **`createDbQueryHelper(body)`** -> `DbQueryHelper` -- composable read-only helper. Can only be called inside an `UnreplayableQuery.db(...)`, `UnreplayableSingleMutation.db(...)`, or `SingleMutation.db(...)` body.
- **`createDbMutationHelper(body)`** -> `DbMutationHelper` -- composable mutation helper. Can only be called inside a `SingleMutation.db(...)` or `UnreplayableSingleMutation.db(...)` body.
- `requirePrimaryTransaction(effect)`, `requireTransientTransaction(effect)`, and the family-specific DB transaction scopes are raw transaction-scope tools for infrastructure and adapters. Prefer `*.db(...)` bodies and DB helper constructors for domain code.
- Keep transaction-scoped helpers narrow. They should usually be private
  sequence/allocation or row-shape primitives used inside a domain
  `*.db(...)` boundary. Public domain APIs should own the transaction boundary
  and expose domain decisions rather than raw transaction plumbing.
- Do not introduce `*InTxn` helper APIs just to make related follow-up work
  happen in the same commit. If the follow-up is not part of the committed-state
  invariant, model it as an explicit replayable step.

## Running At Entry Points

Every RPC handler or entrypoint terminates domain operation work with an
`Operation.run*` call.

### Mutation Handlers

```typescript
pipe(
  SingleMutation.db(
    { success: ResultSchema, error: ErrorSchema },
    Effect.gen(function* () { /* ... */ }),
  ),
  Operation.runReplayable({
    replayKey: ["Namespace.RpcName", replayKey],
  }),
)
```

- `replayKey` comes from the RPC request payload. Namespace it with the RPC name.
- The caller keeps `replayKey` stable across retries of the same logical mutation.

### Unreplayable Mutation Handlers

```typescript
pipe(
  UnreplayableSingleMutation.db(
    Effect.gen(function* () { /* ... */ }),
  ),
  Operation.runUnreplayable,
)
```

- Use when replay is not needed such as maintenance, timing, or coordination internals.

### Read-Only Handlers

```typescript
pipe(
  UnreplayableQuery.effect(
    Effect.gen(function* () { /* ... */ }),
  ),
  Operation.runQuery,
)
```

### Streaming Handlers

```typescript
pipe(
  UnreplayableQueryStream.effect(
    Effect.gen(function* () { /* ... returns Stream ... */ }),
  ),
  Operation.runQuery,
)
```

For invalidation-backed snapshot streams, the handler may return a raw
`Stream` adapter only when the stream setup performs listener wiring and each
snapshot load terminates its own domain work with `Operation.runQuery(...)` or
`Operation.runUnreplayable(...)`. Use this for read-like streams whose snapshots
may allocate local handles or otherwise need mutation semantics. Primary
PostgreSQL invalidation streams should use
`streamFromPrimaryPgInvalidations(...)`.

### Durable Operation Handlers

```typescript
pipe(
  myDurableOperation,
  DurableOperation.execute({ payload }),
  Operation.runReplayable({ replayKey: ["Namespace.RpcName", replayKey] }),
)
```

## Composing With `Flow.call(...)`

Inside a domain operation flow, use `Flow.call(...)` to invoke inner operations. Never call `.run*()` inside an operation flow.

```typescript
SingleMutation.gen(function* () {
  const data = yield* Flow.call(someQuery);
  return yield* Flow.call(someMutation(data));
})
```

Public authoring can use `Flow.gen(...)`, `Flow.call(...)`, and the other `Flow` combinators such as `flatMap`, `tap`, `if`, `matchTag`, and `forEach`. `FlowUnsafe` is reserved for raw `Effect` interop such as `FlowUnsafe.unsafeFromEffect(...)`. Use `Flow.toEffect()` only at plain `Effect` or `Stream` interop boundaries. Otherwise keep composing in `Flow`, and use `Operation.run*` only at entrypoints.

The current context determines what may be called:

- Query flows have `OperationScope` only, so `Flow.call(...)` on a mutation is a compile error.
- Mutation flows have `MutationOperationContext`, so they can call queries and mutations.
- Durable flows add `DurableScope`, enabling `Flow.call(...)` on `ReplayableMultiMutation`s.
- Opaque unreplayable interop regions deliberately reject `Flow.call(...)` on `ReplayableMultiMutation`s. If that region needs multi-step durable work, expose the work as a `DurableOperation` and call `DurableOperation.execute(...)`.
- The child durable body entered by `DurableOperation.execute(...)` clears the opaque marker before it runs; only the outer interop region remains opaque.

Each `Flow.call(...)` gets a structural step key. Auto-generated step keys and structured combinators such as `Flow.forEach(...)` together form the replay path for memoization.

Coordination runtime and framework code that is implementing the operation framework itself should use the internal runtime compose builders in `src/shared/framework/coordination/internal/operations/runtime-compose.ts` rather than the public domain operation constructors in `src/shared/framework/coordination/operations.ts`.

### Explicit Step Keys

Public `Flow.forEach(...)` owns iteration replay identity. Each array element gets its own deterministic child scope, so concurrent loop bodies can safely use plain `Flow.call(...)` inside the iteration body without manual keys.

Explicit step keys are internal-only. Coordination kernel code uses helpers such as `withExplicitChildOperationScope(...)` and `callWithStepKey(...)` when it needs named child scopes outside the public `Flow` combinators.

Duplicate explicit step keys within the same scope are a defect.

## Replay Identity And Memoization

### Replay Path

In replayable mutation contexts, each `Flow.call(...)` gets a stable structural address. On retry, the same path must produce the same observable result.

Replayable mutation roots are also serialized by replay key, so duplicate `Operation.runReplayable({ replayKey })(mutation)` executions cannot overlap and race the same logical path.

### What Gets Memoized

- `Flow.call(Query.effect(...))` or `Flow.call(Query.db(...))` in a replayable mutation context: the result is memoized so the same query returns the same answer on retry.
- `Flow.call(singleMutation)` in a replayable mutation context: the mutation's result is memoized so replay reaches the same observable result without re-executing the mutation.
- `Flow.call(UnreplayableQuery.effect(...))`: never memoized. It cannot be called directly from replayable mutation or durable contexts. Upgrade it to a `ReplayableQuery` with `Query.from(...)`, `Query.effect(...)`, or `Query.db(...)` instead.

Replay memoization is not an optimization or a best-effort cache. It is the durability mechanism for in-flight replayable work. Domain tables should not duplicate memoized values solely so a workflow can recover without the coordination replay state. Persist a value in a domain table when it is part of that domain's storage or observation model; otherwise let the replay path own it.

### Stable IDs

`replayableTokenQuery()` returns a `ReplayableQuery<string>` that generates a random token, memoized on replay. Use this for any ID that must be stable across retries, including provider idempotency keys, reconciliation tags, and external names that exist only to make an in-flight operation replayable.

Replay-stable UUIDv7 ids may be generated before insert when a workflow needs a
future primary key to derive handles, subjects, commands, or provider metadata.
Use `replayableUuidV7Query()` for those ids. If a workflow might reuse an
existing row, read the row first and generate a UUIDv7 candidate only on the
new-row path. The replay boundary should memoize a generic branded UUIDv7
value; convert it to the specific domain id at the domain boundary. Do not
insert a domain row solely to obtain a generated id.

Durable workflow replay state may carry candidate ids, tokens, selected
configuration, generated commands, and other setup values across steps. Persist
those values in domain tables only when they are part of the domain's storage or
observation model.

## Durable Operations

Use durable operations for workflows that span multiple independent side effects such as separate transactions or DB plus external API sequences.

### Defining

Use `createDurableOperation(...)` by default:

```typescript
export const myOperation = createDurableOperation(
  { key: "Domain.OperationName", domain: MyDurableDomain, payload: PayloadSchema },
  (payload) => Flow.gen(function* () {
    yield* Flow.call(stepOne(payload));
    yield* Flow.call(stepTwo(payload));
  }),
);
```

Split only when app code needs to enqueue work without importing the workflow
implementation:

```typescript
export const myOperation = declareDurableOperation({
  key: "Domain.OperationName",
  domain: MyDurableDomain,
  payload: PayloadSchema,
});

const myOperationBinding = implementDurableOperation(
  myOperation,
  myOperationImplementation,
);

export const MyDurableOperationCatalog = makeDurableOperationCatalog({
  bindings: [myOperationBinding.binding],
});
```

There is no required filename or file count. Keep the operation in one module
unless the import boundary is useful.

Durable operation implementations return `OperationFlow`. If you already have a
reusable `ReplayableMultiMutation`, call it inside the implementation with
`Flow.call(existingWorkflow(...))`.

### Running

- `DurableOperation.submit(payload)(declarationOrHandle)` -- fire-and-forget; returns after durable enqueue.
- `DurableOperation.execute(payload)(handle)` -- foreground execution with crash recovery. Requires an implemented handle.
- `Flow.call(handle, payload)` -- inline execution inside a parent durable flow. Requires an implemented handle.

### Nesting

- `DurableOperation.execute(payload)(handle)` cannot be called inside a normal durable operation body.
- Opaque unreplayable interop regions are the exception: they may use `execute(...)` to create an explicit child recovery boundary.
- `DurableOperation.submit(payload)(handle)` and `Flow.call(handle, payload)` both work within durable contexts for composing follow-up work.

### Durable Intermediate State

Durable workflows may persist intermediate state between replayable steps. That
state is part of replay recovery and operator inspection.

- Keep separate durable workflow steps in separate transactions by default.
  Do not merge a later step into an earlier `SingleMutation.db(...)` because it
  is expected to run immediately, because replay will eventually run it, or
  because the committed prefix looks incomplete without it.
- Combining writes into one DB transaction requires a concrete observation-time
  domain invariant: after either write alone commits, a fresh reader would see a
  state the domain is not allowed to expose.
- Advance support, dedupe, checkpoint, and projection state in explicit
  replayable steps unless that state participates in such an invariant.
- Durable workflow correctness is judged by the state reached after successful
  replay, not by requiring every committed prefix to look like the final product
  state.
- A committed prefix may include real side effects, reservations, charges,
  ownership markers, or derived intermediate state whose matching domain
  artifact is produced by a later replayable step.
- Such prefixes are correct when replay can resume from them and complete the
  workflow without duplicating earlier side effects.
- Do not introduce generic cleanup scopes, finalizers, or transaction-callback
  APIs solely to make a multi-step durable workflow appear atomic.
- Do not create placeholder artifacts, fallback projections, or rollback
  machinery solely to make a dead-lettered prefix look complete.
- Cleanup after typed failures belongs in explicit domain compensation steps.
- Cleanup after defects belongs in explicit operator or admin repair paths; normal durable replay should continue from the persisted intermediate state.
- When reviewing durable code, do not flag "step A committed before step B" as
  a bug by itself. First check whether step A is replay-safe, whether step B
  will be retried from the memoized prefix, and whether successful replay
  reaches the correct final state. Flag it only when replay can duplicate side
  effects, lose the ability to complete, or expose an invalid committed
  invariant that the domain requires at every observation point.

### Dead Letters And Ownership State

Durable-operation dead letters are loud terminal states. A dead-lettered item
means the autonomous worker stopped at an unexpected defect and must not be
silently bypassed by another worker path.

- A dead letter is a suspended durable prefix, not the intended final business
  state.
- The default operator action is to fix the defect, dependency, or data issue
  and replay the same durable operation until it reaches normal completion.
- Compensation, deletion, and manual state edits are explicit repair choices,
  not the default correctness model.
- Do not add automatic "unstick" logic that clears domain ownership state after a durable operation dead-letters.
- If a durable workflow claims a persistent `Running`/owned state before doing multi-step work, leave that state intact when the workflow defects. The stuck state is part of the operator-visible failure signal.
- Future submits for the same work may legitimately no-op or block on that ownership state until the dead letter is manually resolved.
- Retrying, deleting, or compensating a dead letter belongs in an explicit admin/operator repair path. That repair path may include deliberate domain-state changes, but normal background workers should not perform them implicitly.
- Do not persist synthetic fallback results, provisional summaries, or placeholder derived data just to make downstream views look healthy after a defect.
- Dead-letter retry relies on the retained durable item and replay memo state for that attempt. If the coordination retention window expires before repair, treat that as an operational defect requiring explicit investigation rather than requiring every domain workflow to persist shadow copies of replay-only data.

## Choosing A Constructor

- One DB transaction owns the mutation -> `SingleMutation.db(...)`.
- One DB transaction establishes the invariant, and later work merely needs to happen eventually -> keep the DB step as `SingleMutation.db(...)`, then enqueue durable follow-up work.
- Read-only -> `UnreplayableQuery.db(...)`, `UnreplayableQuery.effect(...)`, `Query.db(...)`, or `Query.effect(...)`.
- One external transition where exact replay of the same call is already correct -> `SingleMutation.idempotentExternal(...)`.
- One external transition where duplicate execution is explicitly acceptable -> `UnreplayableSingleMutation.external(...)` or `UnreplayableSingleMutation.effect(...)` plus `unsafeMemoizeSingleMutation(...)`.
- One ambiguous external or single-transition interop mutation -> `UnreplayableSingleMutation.external(...)` or `UnreplayableSingleMutation.effect(...)` plus `uncertainMutation(...)`, then either `stabilizeMutation(...)` or explicit domain recovery from `Confirmed` versus `Unknown`.
- One deterministic interpreter or script runtime whose side effects all cross managed host-call boundaries -> `MultiMutation.compose(...)` / `.gen(...)`, make ambient nondeterminism unavailable or route it through replayable operations, and use `memoizeMultiMutationResult(...)` if later durable steps consume the final interpreter result.
- One opaque callback or interop execution that may do multiple managed mutation calls and needs crash ambiguity modeled -> `uncertainExecution(...)`, then inspect `Completed` versus `UnknownOnCrash`. If that opaque region needs autonomous multi-step durable side effects, make those side effects child `DurableOperation.execute(...)` calls rather than direct `ReplayableMultiMutation` calls.
- One opaque callback or interop execution that may do multiple managed mutation calls where partial and duplicate execution are explicitly acceptable -> `UnreplayableMultiMutation.effect(...)` or `.gen(...)` plus `unsafeMemoizeMultiMutation(...)`.
- Single-step mutation needing serialization -> add `linearizeSingleMutation(...)`.
- Multi-step durable workflow needing serialization -> build `MultiMutation.gen(...)` or `.compose(...)`, then add `linearizeMultiMutation(...)`.
- Multiple independent side effects -> `MultiMutation.gen(...)` or `.compose(...)` plus a durable operation. Split declaration from binding only when an import boundary needs it.
- Cross-service RPC -> `SingleMutation.rpc(...)`.
- Non-replayable mutation such as maintenance, coordination internals, or raw interop at the edge -> `UnreplayableSingleMutation.db(...)`, `UnreplayableSingleMutation.effect(...)`, or `UnreplayableSingleMutation.external(...)`.
