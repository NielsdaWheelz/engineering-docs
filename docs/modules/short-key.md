# Short Key

## Scope

This document covers the shared short key and short handle infrastructure under `src/shared/short-key/`.

## Purpose

- `ShortKey` is a globally unique short token for framework and product modules that need a compact allocation before they have another stable identifier.
- `ShortHandle` is an outward short alias that resolves server-side to a typed target.
- Use short handles only for compact human/agent-friendly references that are expected to be seen, copied, typed, displayed, embedded in URLs, included in prompts, or passed through agent tool calls.
- A short handle is a convenience alias, not authority.
- Agent-facing workspace functions may expose scoped short-handle inputs as `id` when the function name already names the target domain, but each such `id` must use the target's entity-specific short-handle schema.
- Resolve short handles server-side to the expected typed target, then enforce scope and ownership.
- Every short handle points at exactly one short key.
- A short key may exist without a short handle when the caller only needs a unique short token.

## Placement

- Short key and short handle code lives under `src/shared/short-key/`.
- Browser-safe value types and parsing live at the module root; DB-backed allocation and resolution services live under `src/shared/short-key/server/`.
- Persistent short key and short handle tables live in `SharedDb` under `src/shared/server/db/`.
- Main and cloud code use the shared services; they do not own service-local short key or short handle tables.
- `ShortHandles` depends on `ShortKeys`; `ShortKeys` does not depend on handles.

## Typed Boundaries

- Shared `ShortHandle` is infrastructure. Domain modules expose concrete entity-specific short-handle types and schemas for each target.
- Short-handle target specs carry the target key, target schema, entity-specific handle schema, and trusted handle constructor.
- Callers resolve a short handle through the matching target spec. A successful resolve only proves target type; scope and ownership checks still happen after resolution.

## Persistence

- `short_key` rows are permanent allocation records.
- Do not delete or recycle short keys when a referencing resource is deleted.
- `short_handle` rows are the server-side resolution index for outward short aliases.
- Do not add speculative purpose or category columns to short keys. If a short handle needs typed resolution, model that through its typed `targetKey` and `target`.
