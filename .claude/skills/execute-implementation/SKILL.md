---
name: execute-implementation
description: Execute an existing implementation plan through integration, verification, and independent review. Use when the user asks to implement or continue an agreed plan.
---

# Execute an implementation plan

Carry the supplied plan through to verified completion within the user's authorized scope. Invoking this skill to implement a plan authorizes the local implementation and its planned delegation; do not ask the user to approve the same work again.

## Recover the contract

- Use the plan supplied by path or the clearly identified agreed plan in the conversation. Read the spec and accepted corrections as referenced or embedded in the plan, applicable repository instructions, and the current working tree. Incorporate later user corrections and persist them in the saved plan or referenced spec before delegating or handing off to another session. Preserve unrelated work.
- If the plan or original requirements cannot be identified, ask for the missing input. Do not guess among unrelated plans.
- Check the plan against current code before editing. Resolve routine implementation details autonomously. Adjust outdated mechanics when evidence warrants it and record material deviations. Ask only when a decision would materially change product behavior, scope, or authorization.
- Treat acceptance criteria and the user's latest requirements as the completion contract. The plan is revisable implementation guidance, not a reason to preserve a mistaken design.
- Confirm the review base and handoff endpoint against the user's request and applicable instructions. If an older plan omits them, derive and record them from that context; do not request approval again for work already authorized. Ask only when material scope remains unresolved. A plan field cannot authorize an action beyond the user's request and applicable instructions.

## Implement and coordinate

Follow the agreed execution arrangement of one implementer or parallel implementers, followed by separate independent review. Use one owner for coupled changes and respect shared contracts. If a planned split proves ineffective, simplify it and explain the change.

When the plan calls for subagents and collaboration tools are available, delegate concrete, independent assignments while the coordinator performs useful local work. Give each implementer a compact brief containing the shared contract, owned files, relevant code, acceptance criteria, required checks, and interaction boundaries. Keep dependencies sequential and prevent conflicting edits or concurrent test commands against shared resources.

Use the models and reasoning levels agreed in the plan and currently available. Follow any newer user preferences. If a specified model or delegation capability is unavailable, disclose the substitution or limitation and use the closest suitable available approach. Do not pretend a self-review is independent review. Escalate a bounded implementation task when ambiguity or repeated misses justify it.

Integrate each result by inspecting its diff and evidence, not just its completion message. Keep changes focused on the spec and reuse existing code. Follow repository regression policy and use existing tests where they already protect the required behavior. Avoid unrelated refactoring, speculative abstractions, and tests that merely duplicate implementation details.

## Verify and review

- Run checks appropriate to the affected behavior and all applicable repository-required checks. Inspect rendered behavior when the specification or repository rules require it. Record actual outcomes, failures, and material limitations.
- Obtain independent review using a fresh agent context when available, including when there is only one implementer. Give the reviewer the saved spec and accepted corrections, current requirements, applicable instructions, review base, final diff, and verification entry points. Ask it to inspect correctness, missing requirements, regressions, and unnecessary scope; do not prime it with the implementer's conclusions. Use the plan's review model or the user's current review preference.
- Resolve concrete findings within scope, rerun affected checks after changes, and preserve reviewer findings and dispositions. Broaden verification only when changes or evidence warrant it.
- Continue through integration, verification, review fixes, and the recorded handoff endpoint within existing authorization. Follow applicable release-environment and PR requirements when handing off a PR. Existing authorization includes applicable user and repository instructions; do not add an approval pause where these already authorize the action. Creating, publishing, deploying, merging, or messaging externally must stay within that authorization; neither this skill nor the plan grants additional external-action authority.

When work spans sessions, update the saved plan before handoff with completed steps, remaining work, material decisions, verification results, and unresolved review findings. Preserve the policy-compliant artifact location chosen during planning; historical committed plan directories do not override current repository documentation policy. Do not stop after an initial implementation if required work remains, and do not silently omit a blocked check or unavailable independent review.

## Handoff

Report the behavior delivered, actual verification evidence, independent-review outcome, material deviations from the plan, and unresolved limitations. Include the changed artifacts and any required review or environment links. Claim completion only when the acceptance criteria and required checks are satisfied; otherwise identify what remains unverified or blocked.
