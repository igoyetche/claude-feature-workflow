# Spec-to-Review Feature Pipeline for Claude Code (two-repo edition)

A multi-agent workflow: interactive spec interview → architecture + frozen
frontend/backend contract → parallel build across your two repositories →
test + integration fix loop → independent fresh review → retrospective that
makes every agent and the pipeline itself smarter next run.

## Workspace layout

The pipeline runs from a small workspace repo that wraps your two product
repos. Create it once:

```
dev-workspace/                  # git repo — holds pipeline config + docs
├── .claude/                    # ← this scaffold
├── docs/
│   ├── specs/<feature>/        # spec, architecture, contract, review, status
│   └── pipeline-lessons.md     # grows automatically (Phase 6)
├── frontend/                   # your frontend repo (clone or submodule)
├── backend/                    # your backend repo (clone or submodule)
└── wt/                         # transient worktrees during builds (gitignore)
```

Add `frontend/`, `backend/`, and `wt/` to the workspace's `.gitignore`
(or use submodules for the repos). The workspace repo versions the pipeline,
the feature documents, the lessons file, and the agent memories — your team
shares all of it by pulling.

## Install

1. Copy `.claude/` into the workspace root and commit it.
2. Customize `.claude/skills/frontend-conventions/SKILL.md` and
   `.claude/skills/backend-conventions/SKILL.md` — replace every TODO with
   each repo's stack, commands, and conventions.
3. (Recommended) Enable agent teams for the parallel build:

   ```json
   // settings.json
   { "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
   ```

   Without it, the pipeline falls back to parallel background subagents.
4. Start Claude Code from the workspace root. Restart it after first
   copying the scaffold in (subagent files load at session start).

## Run

```
/feature my-feature-name
```

You'll be interviewed, then asked to approve twice (spec, then architecture
+ contract). After the second approval the run is autonomous until it
finishes, hits an iteration cap, or needs a contract change — all of which
come back to you. Re-invoking `/feature <name>` resumes from the phase
recorded in `docs/specs/<name>/status.md`.

## Standalone planning and execution skills

For an existing spec, use these two skills independently of `/feature`.
They work with a single repository or multiple repositories, follow the
target project's instructions, and choose one implementer or parallel
implementers based on the work. Independent review is a separate step.

- [`spec-to-implementation`](.claude/skills/spec-to-implementation/SKILL.md)
  inspects the spec and existing code, saves the implementation plan and
  execution strategy, and stops before implementation.
- [`execute-implementation`](.claude/skills/execute-implementation/SKILL.md)
  executes the agreed plan through integration, verification, independent
  review, and the authorized handoff.

Both are included when you copy `.claude/` as described above. To install
only these skills personally, copy their directories from `.claude/skills/`
into `~/.claude/skills/` for Claude Code or `~/.codex/skills/` for Codex.
The same `SKILL.md` files work in both tools. Keep model preferences in
your global agent instructions; these skills do not prescribe model names
or a product stack.

In Claude Code:

```text
/spec-to-implementation path/to/spec.md
```

Review the saved plan, request any changes, then run:

```text
/execute-implementation path/to/plan.md
```

In Codex, invoke the corresponding skills with
`$spec-to-implementation path/to/spec.md` and
`$execute-implementation path/to/plan.md`.

Pass the saved plan's path when moving to a fresh session. The plan and its
referenced spec carry accepted corrections and progress between sessions.
Plan storage follows the target repository's documentation policy; in
Conductor, the default is `.context/plans/` when that policy permits it.
The handoff endpoint comes from the user's request and applicable
instructions, so these skills do not automatically authorize a push,
deployment, or merge.

## How the pipeline learns (Phase 6 + agent memory)

Three layers, separated by how risky autonomous self-modification is:

1. **Agent memory (autonomous).** Every agent has `memory: project`: a
   persistent directory at `.claude/agent-memory/<agent>/` whose MEMORY.md
   is injected into its context each run. Agents consult it before working
   and append general one-line lessons after. Version-controlled, so the
   whole team's runs feed the same memory.
2. **Pipeline lessons (autonomous).** The lead session has no agent memory,
   so the orchestrator reads `docs/pipeline-lessons.md` at the start of
   every run and appends to it during the Phase 6 retrospective: interview
   questions that were missed, integration pitfalls, process fixes.
3. **Skill and prompt edits (you approve).** When the retro finds a
   systemic lesson that belongs in a conventions skill or agent prompt, it
   proposes a concrete diff. Self-modifying instructions stay human-gated
   so the pipeline can't drift unattended.

The retro classifies every fix round and review finding by root cause and
routes it to the right layer — see Phase 6 in
`.claude/skills/feature/SKILL.md`. It runs even when a run escalates;
failed runs teach the most.

## Design decisions encoded here

- **The spec interview runs in the main session** — subagents cannot ask
  the user questions, and the approval gates need the main session too.
- **The contract freezes at gate 2** and is the only thing connecting the
  two repos. Workers report contract problems; they never improvise around
  them. Amendments require your approval.
- **Test developers are black-box**: tests derive from spec + contract,
  never from reading the implementation.
- **Per-repo worktree pairs** (`wt/fe-impl` + `wt/fe-tests`, same for
  backend) let the dev and test-dev sharing a repo build simultaneously
  without collisions; the lead merges per repo afterwards.
- **Iteration caps**: 3 fix rounds, 3 review rounds, then escalation.
- **The reviewer is always a fresh instance** given only spec,
  architecture, contract, diffs, and test results. Its agent memory is the
  one deliberate exception: it carries the reviewer's own cross-run defect
  patterns, not context from the current build.

## Layout

```
.claude/
├── agents/
│   ├── architect.md       # Phase 2 · memory: project
│   ├── frontend-dev.md    # Phase 3-5 · frontend repo only · memory
│   ├── backend-dev.md     # Phase 3-5 · backend repo only · memory
│   ├── test-dev.md        # Black-box FE/BE/integration suites · memory
│   └── reviewer.md        # Phase 5 · fresh, independent · pattern memory
└── skills/
    ├── feature/               # /feature — orchestrator + templates
    ├── spec-to-implementation/ # Standalone spec → plan + execution strategy
    ├── execute-implementation/ # Standalone plan → implementation + review
    ├── contract-format/       # contract rules (preloaded into agents)
    ├── frontend-conventions/  # YOUR frontend repo — customize
    ├── backend-conventions/   # YOUR backend repo — customize
    ├── modular-monolith-feature-design/  # C# backend boundary rules (architect, backend-dev)
    ├── modular-monolith-review/          # C# backend modularity audit (reviewer)
    └── efcore-conventions/               # EF Core rules (backend-dev, reviewer)
```

## Tuning

- Caps, gates, retro routing: `.claude/skills/feature/SKILL.md`.
- Review scope: `.claude/agents/reviewer.md`.
- Contract rigor: `.claude/skills/contract-format/SKILL.md`.
- Models: add `model: opus` (architect/reviewer) or `model: haiku` to any
  agent's frontmatter; default is `inherit`.
- Reset learning: delete `.claude/agent-memory/` and
  `docs/pipeline-lessons.md`.
