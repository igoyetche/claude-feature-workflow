---
name: frontend-conventions
description: The frontend repository's stack, commands, and coding conventions. Background knowledge for the feature pipeline's agents; not invoked directly.
user-invocable: false
---

# Frontend repository conventions

<!-- ⚠️ CUSTOMIZE: replace every TODO. Terse facts and commands, not essays. -->

Repository: `frontend/` (relative to the workspace root)

## Stack
- TODO (e.g. React 18 + TypeScript + Vite)

## Commands (run from inside the frontend repo)
- Install: TODO
- Tests: TODO
- Lint / typecheck: TODO
- Run locally: TODO

## Integration (e2e) tests
This repo hosts the pipeline's cross-repo integration suite.
- Location: TODO (e.g. e2e/)
- Framework: TODO (e.g. Playwright)
- Run command (expects the real backend running): TODO

## Conventions
- TODO: component structure, state management, API client location,
  test file placement, fixtures to reuse, naming.

## Definition of done
- Compiles, lints, and typechecks clean.
- Follows the patterns above; no new dependencies without need.
- No secrets or debug output committed.
