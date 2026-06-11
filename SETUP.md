# Workspace setup

How to wire this workspace to the two product repos so `/feature` can run.
Nothing needs to change inside the product repos themselves — all pipeline
machinery lives here; agents reach into the repos through worktrees and
branches, which any git repo supports out of the box.

## 1. Put the repos inside the workspace

From the workspace root:

```
git clone <your-frontend-repo-url> frontend
git clone <your-backend-repo-url>  backend
```

`.gitignore` already excludes `frontend/`, `backend/`, and `wt/`, so this
workspace repo keeps versioning only the pipeline config, specs, and
lessons.

Alternative: add them as submodules if you want the workspace to pin exact
commits — then remove those two lines from `.gitignore`. Plain clones are
simpler and the recommended starting point.

## 2. Fill in the conventions skills

- `.claude/skills/frontend-conventions/SKILL.md`
- `.claude/skills/backend-conventions/SKILL.md`

This is the one genuinely load-bearing step: the pipeline runs each repo's
install/test/lint commands and the integration suite from what's written
there, so every TODO that's still a TODO is a phase that can't run.

## 3. Make sure each repo runs locally

From a fresh clone: install, tests, lint, and for the backend the local run
including its database. Phase 4 starts the real backend and drives the real
frontend against it, so whatever that needs (connection string, docker
compose, seed data) must work non-interactively and be documented in the
conventions skill.

## 4. (Recommended) Enable agent teams

For the parallel build, create `.claude/settings.json` in the workspace
root:

```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

Without it the pipeline still works, just via parallel background subagents
instead.

## 5. Start Claude Code from the workspace root

Not from inside either repo — agents and skills load from `.claude/` at
session start, so restart the session after config changes. Then
`/feature <name>` runs the whole pipeline; `docs/specs/<name>/` is created
on demand.

## Product-repo side: nothing required

The pipeline creates branches (`feature/<name>-impl`, `-tests`,
`-integration`, merged into `feature/<name>`) but never pushes — review
happens locally via diff ranges. No CI or branch-protection setup is needed
for the pipeline itself; push `feature/<name>` afterwards if you want your
normal PR flow on top.
