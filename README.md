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
    ├── contract-format/       # contract rules (preloaded into agents)
    ├── frontend-conventions/  # YOUR frontend repo — customize
    └── backend-conventions/   # YOUR backend repo — customize
```

## Tuning

- Caps, gates, retro routing: `.claude/skills/feature/SKILL.md`.
- Review scope: `.claude/agents/reviewer.md`.
- Contract rigor: `.claude/skills/contract-format/SKILL.md`.
- Models: add `model: opus` (architect/reviewer) or `model: haiku` to any
  agent's frontmatter; default is `inherit`.
- Reset learning: delete `.claude/agent-memory/` and
  `docs/pipeline-lessons.md`.
