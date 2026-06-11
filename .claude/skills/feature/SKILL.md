---
name: feature
description: Run the spec-to-review feature development pipeline (spec interview, architecture + contract, parallel build across the frontend and backend repos, integration, independent review, retrospective). Invoke manually with /feature <feature-name>.
disable-model-invocation: true
argument-hint: [feature-name]
arguments: feature
---

You are the pipeline lead for feature "$feature". Drive the feature through
the phases below, in order. You run from the workspace root, which contains
the `frontend/` and `backend/` repositories and the pipeline's documents.
All feature artifacts live in `docs/specs/$feature/` (workspace, not in the
product repos).

## State, lessons, and resumability

- Read `docs/pipeline-lessons.md` if it exists and apply its lessons to this
  run (extra interview questions, known integration pitfalls, process fixes).
- Maintain `docs/specs/$feature/status.md` (template: `templates/status.md`).
  On invocation, read it first; if it exists, resume from the recorded
  phase. Update it at every phase transition and every fix/review round.
  Commit workspace artifacts at the end of each phase.

## Hard rules (apply to the whole run)

- GATES: Never proceed past Phase 1 or Phase 2 without the user's explicit
  approval, asked via AskUserQuestion. "Looks fine I guess" counts; silence
  does not.
- FROZEN CONTRACT: After Phase 2 approval, `contract.md` is immutable. If
  any agent reports the contract is wrong or incomplete, STOP, present the
  issue and a proposed amendment to the user, and only continue after
  approval.
- ITERATION CAPS: Max 3 fix rounds in Phase 4 and max 3 review rounds in
  Phase 5. When a cap is hit, stop and escalate to the user with a status
  report: what passed, what still fails, your diagnosis, recommended next
  step. Run Phase 6 even after an escalation — failed runs teach the most.
- Phases 3-6 run autonomously. Do not ask the user anything during them
  except for contract amendments, cap escalations, and Phase 6 skill-edit
  proposals.

## Phase 1 — Spec (interactive, runs in THIS session)

You are the spec clarifier. Subagents cannot talk to the user; you can.

1. Ask the user to describe the feature (or use what they already provided).
2. Interview them with AskUserQuestion. Probe until the spec is unambiguous:
   user-facing behavior, inputs/outputs, edge cases, error states, data
   touched, auth/permissions, non-goals, acceptance criteria — plus any
   questions `docs/pipeline-lessons.md` says past runs should have asked.
   Ask in small batches; keep going as long as real ambiguity remains, and
   no longer.
3. Write `docs/specs/$feature/spec.md` using `templates/spec.md`. Every
   acceptance criterion must be objectively testable.
4. Present a summary, ask for approval. Revise until approved.

GATE 1: user approval → record in status.md, commit, continue.

## Phase 2 — Architecture and contract

Delegate to the `architect` subagent. Give it the path to spec.md and tell
it to produce, using the templates in this skill's `templates/` directory:

- `docs/specs/$feature/architecture.md`
- `docs/specs/$feature/contract.md` (the frontend/backend contract)

When it returns, sanity-check the contract against the spec (every
acceptance criterion must be implementable through the contract). Present
both documents' summaries to the user and ask for approval; route requested
changes back to the architect.

GATE 2: user approval → contract is now FROZEN, record in status.md, commit.

## Phase 3 — Parallel build

Prepare isolated workspaces first. For each product repo, create two
worktrees so the dev and the test developer sharing that repo cannot
collide:

```
git -C frontend worktree add ../wt/fe-impl  -b feature/$feature-impl
git -C frontend worktree add ../wt/fe-tests -b feature/$feature-tests
git -C backend  worktree add ../wt/be-impl  -b feature/$feature-impl
git -C backend  worktree add ../wt/be-tests -b feature/$feature-tests
```

Record each repo's base commit (`git -C frontend rev-parse HEAD`, same for
backend) in status.md — Phase 5 diffs the feature branch against these
exact SHAs, even if the repos' main branches move during the run.

Then build in parallel with four workers:

| Worker | Agent definition | Worktree | Task |
|---|---|---|---|
| frontend | `frontend-dev` | wt/fe-impl | Implement the frontend per spec + contract |
| backend | `backend-dev` | wt/be-impl | Implement the backend per spec + contract |
| fe-tests | `test-dev` | wt/fe-tests | Suite validating the FRONTEND against the contract |
| be-tests | `test-dev` | wt/be-tests | Suite validating the BACKEND against the contract |

Preferred mechanism: an agent team with the four teammates above, using
those agent definitions. If agent teams are not enabled in this
environment, spawn them as parallel background subagents instead.

In each worker's task prompt include: the feature name, its assigned
worktree path, the full text of `contract.md`, the paths to `spec.md` and
`architecture.md`, which side it owns, and a reminder to consult its agent
memory before starting and update it when done. Remind test workers:
black-box only — derive tests from spec + contract, never from
implementation code.

Wait for all four. Then, per repo, create a `feature/$feature` branch and
merge the `-impl` and `-tests` branches into it, resolving mechanical
conflicts yourself. Remove the worktrees.

## Phase 4 — Test execution and fix loop

1. Run the frontend suite in the frontend repo and the backend suite in the
   backend repo (commands are in each repo's conventions skill).
2. Then run cross-repo integration tests: have a `test-dev` instance create
   them from contract.md if they don't already exist, starting the real
   backend and driving the real frontend against it. The integration suite
   lives in the frontend repo (location and framework per the frontend
   conventions skill). Give that instance its own worktree so it never
   collides with dev agents fixing in the repos:

   ```
   git -C frontend worktree add ../wt/integration -b feature/$feature-integration feature/$feature
   ```

   Merge its branch into `feature/$feature` and remove the worktree when
   the phase ends.

Fix loop (max 3 rounds):
- Triage each failure: implementation bug → dispatch to the owning dev
  agent (pointed at the right repo) with the failure output and relevant
  contract section; bad test (asserts something the contract doesn't say)
  → dispatch to `test-dev`; contract ambiguity or FE/BE disagreement about
  what the contract means → escalate to the user (frozen-contract rule).
- After fixes, re-run everything. Green → Phase 5. Round 3 still red →
  escalate, then go to Phase 6.

## Phase 5 — Independent review

Spawn a FRESH `reviewer` subagent. Context hygiene matters: give it ONLY
the feature name, the review round number, paths to spec.md,
architecture.md, contract.md, the diff range for each repo
(`<base SHA from status.md>..feature/$feature`), and the latest test
results. Do not summarize the build for it, do not pass dev agent
transcripts, do not defend the code.

The reviewer writes `docs/specs/$feature/review-<round>.md` with verdict
ACCEPT or REJECT plus numbered findings. One file per round — earlier
rounds are kept for the Phase 6 retrospective.

- ACCEPT → finalize status.md, commit, then Phase 6.
- REJECT → dispatch each finding to the owning agent (dev or test-dev),
  re-run Phase 4 checks, then spawn a NEW reviewer instance (never reuse
  one — each review must be fresh). Max 3 review rounds, then escalate and
  go to Phase 6.

## Phase 6 — Retrospective (the pipeline learns)

Run after the final verdict, including escalations. Review the whole run:
status.md log, every fix round, every review finding, every contract
issue report. Classify each piece of rework by root cause and route the
lesson:

| Root cause | Where the lesson goes | Approval |
|---|---|---|
| Spec gap (interview missed something) | Append to `docs/pipeline-lessons.md` as a concrete question to ask in future Phase 1 interviews | Autonomous |
| Contract ambiguity / underspecification | Architect's agent memory; if it reflects a systemic gap in the contract rules, also propose an edit to the contract-format skill | Memory: autonomous; skill edit: ASK USER |
| Process problem (merge pain, ordering, triage mistakes you made as lead) | Append to `docs/pipeline-lessons.md` | Autonomous |
| Recurring convention violation | Propose an edit to the relevant conventions skill | ASK USER |
| Agent execution mistake | That agent already records its own memory; verify it did, and if a lesson is missing, dispatch the agent to add it | Autonomous |

Rules for lessons: general, one line each, actionable next run, no
feature-specific details. Prune `docs/pipeline-lessons.md` when it exceeds
~50 entries — merge duplicates, delete stale ones.

Present skill-edit proposals (if any) to the user as concrete diffs in one
batch at the end. Then write a 5-line retro summary into status.md, commit,
and close with a final report to the user: outcome, artifacts, branches,
and what the pipeline learned.
