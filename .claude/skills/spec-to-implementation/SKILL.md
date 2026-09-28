---
name: spec-to-implementation
description: Create a code-grounded implementation plan and execution strategy from a supplied spec. Use when asked to plan a spec or feature before implementation.
---

# Plan a spec

Turn the user's spec into a concise, reviewable plan that another session can execute. This skill produces the plan; it does not authorize implementation or spawning implementation agents.

## Establish the work

- Use the spec supplied by file path, attachment, or conversation. Include later user corrections. If the intended spec is missing or ambiguous, ask for that input rather than inventing requirements.
- Read applicable repository instructions and inspect the relevant existing code and coverage. Identify reusable code, actual interfaces, dependencies, and consequential failure cases before proposing changes. Scale exploration to the task.
- Separate confirmed requirements from assumptions and unresolved decisions. Investigate technical unknowns where possible; ask about product choices only when they materially change the solution.

## Plan implementation and execution together

Choose the simplest effective approach considering quality, total cost including rework, elapsed time, uncertainty, dependencies, shared files, and shared runtime resources. Choose one implementer or parallel implementers; independent review in a fresh context is a separate step in either arrangement. One implementer is a valid recommendation; do not manufacture parallel work or fixed team sizes.

Define shared interfaces, domain behavior, and failure contracts before dividing work. Keep coupled changes with one owner. Delegate only bounded responsibilities that can proceed independently. Assign an integration owner and sequence dependent tasks. Account for repository restrictions on concurrent tests or shared databases even when coding is parallel.

Apply the user's current model preferences from their global instructions, check which models are available in the execution environment, and record the selected models and reasoning levels in the plan. Record any needed substitution explicitly. Recommend a stronger implementer when ambiguity or demonstrated requirement misses justify it. Do not claim prices or model availability without evidence.

## Deliver a proportional plan

Include the following information, combining sections when the task is small:

1. Durable spec reference, accepted corrections, intended outcome, non-goals, constraints, and observable acceptance criteria.
2. Relevant current behavior, code to reuse, affected files, and shared contracts.
3. Ordered implementation steps, dependencies, and unresolved questions or assumptions.
4. Recommended execution arrangement and a brief reason it is worth its coordination cost. For each proposed agent, give its model and reasoning level, bounded responsibility, owned files, relevant context, acceptance checks, and handoff to the integration owner. Include useful work for the coordinator while delegates run.
5. Verification using existing coverage where sufficient, new coverage only for missing consequential behavior, and repository-required review and handoff checks. State the base branch for review and the handoff endpoint, such as local changes, a local commit, a pushed branch, or a PR targeting that base with its required review environment. Derive the endpoint from the user's request and applicable instructions; record its source and any genuinely unresolved scope. A plan field records existing authorization and does not grant new authority. Give known commands or mark commands that still need discovery; distinguish planned checks from checks actually run.

For independent review, plan a fresh context with the saved spec and accepted corrections, applicable instructions, review base, and final diff so the reviewer can assess the implementation without inheriting its author's conclusions.

Save the plan at the user's requested path; otherwise follow the repository's current documentation policy. Historical plan directories do not establish permission to store new point-in-time plans there. In a Conductor workspace, use the gitignored `.context/plans/<feature-slug>.md` when consistent with that policy. Elsewhere, use a policy-compliant local or external location; verify that a local scratch location is gitignored when plans must stay outside the main tree. Preserve existing files and report the path. Do not commit the plan unless requested or required by the authorized workflow.

Reference the spec by a durable path or accessible document link. If it exists only in chat or a transient attachment, save its supplied content in a companion spec file or embed it in the plan; a summary alone is not the original spec. Record accepted corrections in the plan or referenced spec. When the user requests plan revisions, update the saved artifacts before handing off. The plan and its referenced artifacts must carry the requirements into a fresh session without relying on the planning chat; later user instructions still take precedence.

Finish with the plan link, execution recommendation, and any decisions that need the user. Stop before implementation. Do not add a new mandatory approval stage to an implementation task that was already authorized; use this planning-only workflow when the user is requesting a plan.
