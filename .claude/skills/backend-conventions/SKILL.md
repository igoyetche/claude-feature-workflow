---
name: backend-conventions
description: The backend repository's stack, commands, and coding conventions. Background knowledge for the feature pipeline's agents; not invoked directly.
user-invocable: false
---

# Backend repository conventions

<!-- ⚠️ CUSTOMIZE: replace every TODO. Terse facts and commands, not essays. -->

Repository: `backend/` (relative to the workspace root)

## Stack
- TODO (e.g. FastAPI + Python 3.12 + PostgreSQL)

## Commands (run from inside the backend repo)
- Install: TODO
- Tests: TODO
- Lint / typecheck: TODO
- Run locally (incl. database): TODO

## Conventions
- TODO: module layout, error handling, migrations, auth patterns,
  test file placement, fixtures/factories, naming.

## Definition of done
- Compiles, lints, and typechecks clean.
- Follows the patterns above; no new dependencies without need.
- No secrets or debug output committed.
