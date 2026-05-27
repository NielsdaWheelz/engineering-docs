# Engineering Docs

Language-agnostic engineering documentation and project standards.

## Status

The core docs are written as reusable engineering standards. Some module-owned
docs may still capture repository-local systems; treat those as extraction
source material until their portable rules have been pulled into the shared
standards.

## Entry Point

Start with [`docs/index.md`](docs/index.md).

## Consumption

The docs are plain Markdown and intentionally do not require a language runtime.
Projects can consume this repository by:

- adding it as a private Git dependency or submodule,
- vendoring/copying `docs/` into a project-local docs directory,
- using a repository-local sync script that copies `docs/` from a checked-out
  copy of this repository.

Consumer repositories should keep local product, module, and architecture docs
separate from these shared standards.

## Boundary

Shared docs should define reusable engineering rules. Repository-local docs
should stay in the repository that owns the product, module, deployment, or
runtime behavior.
