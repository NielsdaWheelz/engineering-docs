# Codapt Engineering Docs

Language-agnostic engineering documentation and project standards.

## Status

This repository currently contains a direct Markdown snapshot from `codapt2`.
Some docs still mention repository-specific paths, TypeScript, Bun, PostgreSQL,
Solid, and Effect. Treat that as migration debt: the content is being extracted
first, then generalized.

## Entry Point

Start with [`docs/index.md`](docs/index.md).

## Consumption

The docs are plain Markdown and intentionally do not require a language runtime.
Projects can consume this repository by:

- adding it as a private Git dependency or submodule,
- vendoring/copying `docs/` into a project-local docs directory,
- using a project-specific sync script that copies `docs/` from a checked-out
  copy of this repository.

Consumer repositories should keep local product, module, and architecture docs
separate from these shared standards.

## Boundary

Shared docs should define reusable engineering rules. Project-specific docs
should stay in the project that owns the product, module, deployment, or runtime
behavior.
