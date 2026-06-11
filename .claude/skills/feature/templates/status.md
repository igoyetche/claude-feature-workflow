# Pipeline status: <feature name>

Current phase: SPEC | ARCHITECTURE | BUILD | TEST-FIX | REVIEW | RETRO | DONE | ESCALATED

## Gates
- Spec approved: NO | YES (<date>)
- Architecture + contract approved (contract frozen): NO | YES (<date>)

## Base commits (recorded at Phase 3 start; Phase 5 diffs against these)
- frontend: <SHA>
- backend: <SHA>

## Counters
- Fix rounds used (max 3): 0
- Review rounds used (max 3): 0

## Log
Append one line per event: phase transitions, worker completions, fix-round
results, review verdicts, escalations.

- <timestamp> — pipeline started
