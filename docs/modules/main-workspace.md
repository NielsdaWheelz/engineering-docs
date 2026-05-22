# Main Workspace

## Scope

This document covers the core main workspace web/server boundary: workspace context resolution, workspace creation, workspace profile fields, workspace summary projection, onboarding-note completion state, and pre-activation billing-plan selection.

## Boundaries

- `src/main/server/internal/web/workspace-context.ts` owns workspace context resolution, access checks, and `MainWorkspaceInfo` materialization through `MainWorkspaceAccess`.
- `src/main/server/internal/web/workspace-service.ts` owns core `main_workspace` mutation semantics for ensuring a workspace for a principal, setting the pre-activation billing plan, and completing the onboarding note.
- `src/main/server/internal/agent/workspace-summary.ts` owns the generated `main_workspace.summary` projection from workspace notes and agent summaries.
- `session-handlers.ts` and `workspace-handlers.ts` adapt current session/workspace context and RPC replay keys into `MainWorkspaceService`; they do not own core `main_workspace` table writes.
- Billing, grants, contact-number verification, navigation, surfaces, and workspace resources remain owned by their feature-local services.
