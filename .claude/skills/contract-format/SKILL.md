---
name: contract-format
description: Conventions for reading and writing the frontend/backend contract document used by the feature pipeline. Background knowledge for agents; not invoked directly.
user-invocable: false
---

# The frontend/backend contract

`docs/specs/<feature>/contract.md` is the single source of truth that lets
frontend, backend, and test workers build in parallel without talking to each
other.

## Structure (when writing one)

1. **Overview** — one paragraph: what the two sides exchange and why.
2. **Endpoints / interfaces** — one subsection per endpoint or interface:
   method + path (or function signature / channel name), purpose, auth
   requirements.
3. **Request schema** — every field: name, type, required/optional, constraints
   (length, range, format), and a valid example. For map/record fields, define
   the full cross of key-status × value-type: if unknown keys are ignored but
   values must be a certain type, state which rule wins for an invalid value
   under an unknown key.
4. **Response schema** — same rigor, for success responses. Include an example.
5. **Error behavior** — every error case: trigger condition, status code /
   error shape, message format. Unspecified errors are contract violations.
6. **Data types glossary** — shared types (IDs, timestamps, enums) defined
   once, referenced everywhere. State formats explicitly (e.g. timestamps are
   ISO 8601 UTC strings).
7. **Non-functional notes** — only what both sides must honor: pagination
   rules, idempotency, rate limits, ordering guarantees.

Be exhaustive on shapes and exact on names. A reader implementing one side
must never need to guess what the other side does. Prefer concrete examples
over prose. If the spec leaves a wire-level detail open, decide it here and
record the decision — the contract exists to eliminate ambiguity.

## Rules (when working from one)

- The contract is LAW. Implement exactly what it says: exact field names,
  exact status codes, exact error shapes.
- Never edit `contract.md`. If you believe it is wrong, incomplete, or
  ambiguous: stop work on the affected part, report the issue precisely
  (section, problem, suggested fix) to the lead / in your result, and continue
  with unaffected work. The lead escalates contract changes to the user.
- When the contract and the spec appear to conflict, the contract wins for
  wire-level details; report the conflict anyway.
- Test writers: every endpoint, every field constraint, and every documented
  error case in the contract should have at least one test.
