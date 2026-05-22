# Coordination

## Scope

This document covers the coordination module surface, storage backends, transient primitives, dead letter handling, and placement rules for new coordination primitives.

## Purpose

The coordination module provides multi-process coordination primitives and higher-level patterns for replayable workflows. See [../operation-types.md](../operation-types.md) for the operation type system these primitives support.

## Module Surface

Coordination module primitives fall into three categories:

### Operation Primitives

These are the primary developer-facing API. Imported from `@codapt/core/shared/framework/coordination/operations` and `@codapt/core/shared/framework/coordination/*`.

- **Leaf primitives**: `SingleMutation.db`, `UnreplayableQuery.db`, `UnreplayableSingleMutation.db`, `UnreplayableSingleMutation.external`, `SingleMutation.rpc`, `Query.db`.
- **Operation constructors**: `SingleMutation.compose` / `.gen` / `.idempotentExternal`, `Query.compose` / `.gen`, `MultiMutation.compose` / `.gen`, `UnreplayableQuery.compose` / `.gen`, `UnreplayableQueryStream.compose` / `.gen`, `UnreplayableSingleMutation.compose` / `.gen`, `UnreplayableMultiMutation.compose` / `.gen`.
- **Leaf and query constructors**: `PureQuery.effect`, `UnreplayableQuery.effect`, `Query.effect`, `Query.from`, `UnreplayableQueryStream.effect`.
- **Raw-effect interop constructors**: `UnreplayableSingleMutation.effect`, `UnreplayableMultiMutation.effect`.
- **In-transaction helpers**: `createDbQueryHelper`, `createDbMutationHelper`.
- **Durable operations**: `declareDurableOperation`, `implementDurableOperation`, `createDurableOperation`, `CreateDurableDomain`.
- **Infrastructure combinators**: `uncertainMutation`, `unsafeMemoizeSingleMutation`, `unsafeMemoizeMultiMutation`, `memoizeMultiMutationResult`, `uncertainExecution`, `stabilizeMutation`, `linearizeSingleMutation` / `linearizeMultiMutation` (via `CoordinationLinearize`), `toctouQuery`, `toctouMutation`.
- **Utilities**: `replayableTokenQuery`.
- `Unknown` and duplicate-risk semantics belong inside coordination and operation recovery. They must not cross a product boundary unchanged.

### Transient Coordination Services

Low-level multi-process coordination backed by `CoordinationTransientDb`. Used internally by operation infrastructure; rarely referenced directly by domain code.

- **`CoordinationMemoized`**: run-or-return-cached keyed by coordination key. Used internally by mutation-frame memoization; domain code should rarely call this directly, but explicit-key maintenance or infrastructure code may.
- **`CoordinationQueue`**: ordered item collection with PG LISTEN/NOTIFY subscription. Used by the durable operation engine for queue-backed execution.
- **`CoordinationDeferred`**: write-once, read-many value with PG NOTIFY signaling. Used by durable operations for caller-side await.
- **`CoordinationHeartbeat`**: liveness detection built on `CoordinationPulse`. `start`, `pulse`, `isAlive`, `keepAlive`, `awaitExpiry`.

### Internal Primitives

Behind `internal/`. Not imported by domain code.

- **`CoordinationLease`**: time-bounded exclusivity row on a key. Provides raw live acquire or release for infra locking plus replayable acquire and release steps and deadline-based wait queries for the internal exclusivity protocol.
- **`CoordinationExclusive`**: replay-aware internal multi-step exclusivity protocol on top of `CoordinationLease`. Sticky in durable replayable mutation contexts and live in unreplayable mutation contexts. Used by `linearizeMultiMutation`; the protected body keeps the caller's single-vs-multi mutation semantics while acquire or release protocol steps run in internal multi-mutation regions.
- **`CoordinationLiveLock`**: attempt-scoped heartbeat lock on top of `CoordinationLease`. Used internally by replay-root serialization, the durable engine, `CoordinationMemoized`, and `linearizeSingleMutation`.
- **`CoordinationMemo`**: write-once key-value store. Backing store for `CoordinationMemoized`.
- **`CoordinationPulse`**: timestamped touch. Backing store for `CoordinationHeartbeat`.
- **Runtime compose builders**: builders under `src/shared/framework/coordination/internal/operations/runtime-compose.ts` for framework code that is implementing operation semantics rather than domain workflows.

## Wiring Convention

Public reusable coordination APIs expose a semantic service or handle boundary rather than exporting helper functions that inline-close another module's self-wired internal service dependencies with `withSelfService(...)`.

- Use `withSelfService(...)` in exported APIs only at true execution or adapter boundaries, or in handle factories such as durable and scheduled-task wrappers where choosing the canonical runtime internally is the point.
- If other services may reasonably depend on a coordination or admin API, prefer exposing that API as a service and keep lower-level inspection or backend helpers behind its layer.
- For durable operations, split declaration from implementation only when app
  code needs an enqueue-only import. Bind implementations at catalog, worker,
  or adapter composition boundaries; there is no filename convention.

## Storage Backends

### `SharedDb` (Primary)

Coordination tables that must stay in primary PostgreSQL live in `SharedDb` under `src/shared/server/db/` because `coordination_primary_txn_memo` rows commit atomically with business mutations:

- `coordination_primary_txn_memo` -- used by `SingleMutation.db(...)` for crash-recovery memoization.
- `coordination_durable_operation_dead_letter` -- dead-lettered durable operation items.
- `coordination_scheduled_task` -- scheduled task definitions.

### `CoordinationTransientDb` (Transient)

Transient coordination state with a 72h TTL horizon and maintenance pruning. This database owns transient replay caching and hot-path coordination data:

- `coordination_transient_txn_memo` -- transient replayable mutation memo for coordination backends.
- `coordination_transient_memo` -- `CoordinationMemo` entries.
- `coordination_transient_lease` -- `CoordinationLease` entries.
- `coordination_transient_queue_generation` / `coordination_transient_queue_item` -- `CoordinationQueue` state.
- `coordination_transient_pulse` -- `CoordinationPulse` entries.

`CoordinationTransientDb` is the PostgreSQL implementation detail. Runtime access to transient tables is confined to the internal transient backend layer under `src/shared/framework/coordination/internal/transient/backend/*`, and public and domain code should depend on coordination primitive APIs rather than on PostgreSQL directly.

## Waiting In Replayable Workflows

- Durable exclusivity acquisition uses replayable lease-acquire attempts plus a replayable "wait until notified or recorded expiry" step.
- The wait step records completion, not a cached duration, so replay never sleeps based on stale wall-clock values.
- Sticky durable exclusivity does not heartbeat. It acquires once with `TRANSIENT_STATE_TTL`, runs the protected body, and replays its explicit release step like any other child mutation step.
- Sticky exclusivity releases on success and typed failure. It does not release on defects or interruptions; defected durable work should remain visibly stuck until an operator inspects the dead letter or the transient TTL expires.
- This pattern is currently kept local to lease and the internal exclusivity protocol rather than exposed as a generic operation primitive. The correctness story is specific: a PG-notified coordination key and a DB-backed deadline.
- `stabilizeMutation` does not need the same abstraction today. Its retry and reconciliation sleeps are advisory polling backoff, not correctness-bearing deadlines.

## Dead Letter Handling

- Durable dead-letter handling is for item-local poison and execution defects.
- Unregistered operations, payload decode failures, and execution defects are dead-lettered.
- Infrastructure defects before a valid queue item exists, and post-execution defects such as deferred completion or queue cleanup still crash the runner.
- Dead letters are operational containment and debugging, not application control flow.
- Do not persist synthetic "success" artifacts or placeholder derived state merely to avoid dead-lettering a defective durable-operation item.
- Dead-lettering does not complete the caller deferred.
- The original caller waits for retry success or coordination TTL expiry.
- TTL expiry of coordination state is a defect, not graceful degradation.
- Dead letters can be retried through the admin UI, which re-enqueues the item into the transient queue.

## Placing New Primitives

- If a coordination primitive needs persistent primary PostgreSQL tables, put those tables in `SharedDb` and keep the semantic API under `src/shared/framework/coordination/`.
- If a primitive only uses `CoordinationTransientDb`, place it in `transient/` or `internal/transient/` when it is internal.
- If a public API is keyed by replay identity or by an explicit coordination key, place it outside `internal/`.
- If a primitive only provides low-level non-replay-keyed coordination mechanics such as write-once values, lease gates, or mutual exclusion, place it behind `internal/`.
- If code is a shared implementation detail of multiple primitives, place it in nested `internal/` within the parent primitive.
- Lower-level coordination mechanics that require callers to manage their own consistency guarantees belong in `internal/`.
