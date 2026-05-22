# Main Semantic Index

## Scope

This document covers main-side semantic retrieval infrastructure: semantic-index generation contracts, hosted embeddings, reranking, committed skill semantic-index artifacts, and DB-backed workspace context search entries.

## Boundaries

- `src/main/server/internal/semantic-index.ts` owns shared semantic-index source truncation, generation prompt/output contract, corpus/content hashes, and in-memory vector ranking helpers.
- `src/main/server/internal/embedding/semantic-index.ts` owns hosted embedding calls for semantic-index texts and search queries. `semantic-index-contract.ts` owns the embedding model contract, dimensions, normalization, and score conversion.
- `src/main/server/internal/semantic-search.ts` owns cross-encoder reranking and the general candidate/result count policy for semantic search.
- `src/main/server/internal/agent/host-functions/library/helper/skill/` owns skill search/read behavior and committed skill semantic-index artifacts under `src/main/server/internal/agent/skills/`.
- `src/main/server/workspace-context.ts` owns DB-backed mutable workspace context entries and search. It is not the web workspace-context resolution module under `src/main/server/internal/web/workspace-context.ts`.
- `bun run semantic-index:generate` regenerates committed skill artifacts. `bun run semantic-index:check` and `bun run check` validate that generated artifacts are present, canonical, and fresh without regenerating them.
