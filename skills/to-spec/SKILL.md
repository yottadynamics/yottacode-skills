---
name: to-spec
description: Convert a conversation, issue, or rough idea into a concise implementation spec with goals, non-goals, acceptance criteria, and verification. Use before planning or ticketing when the request is bigger than one obvious code change.
license: MIT
metadata:
  author: yottacode
  inspired-by: https://github.com/mattpocock/skills/tree/main/skills/engineering/to-spec
---

# To spec

A spec turns discussion into a stable target. It should be short enough to execute against and precise enough that review can ask, "did we build this?"

Use this skill when the user has described desired behavior but the implementation target is not yet crisp.

## 1. Gather source material

Use the most authoritative source available:

- Current conversation and explicit user constraints.
- GitHub issue or PR body via `gh_issue_read`, `gh_issue_list`, `gh_pr_read`, or `gh_pr_review_context`.
- Existing docs, ADRs, and project memories for domain terms and constraints.
- Code only when needed to understand feasibility or current behavior.

Do not over-research. The goal is a spec, not a codebase audit.

## 2. Resolve material ambiguity first

If an answer would change goals, users, data model, security posture, or touched systems, stop and ask. Offer a recommended answer when you have one.

Do not hide unresolved material questions in an "Open questions" section and proceed as if the spec is stable.

## 3. Write the spec

Use this structure:

```markdown
# <short title>

## Problem
One paragraph describing the user pain or product gap.

## Goals
- Observable outcomes this change must deliver.

## Non-goals
- Tempting adjacent work explicitly out of scope.

## Users / actors
- Who or what uses this capability.

## Proposed behavior
- Concrete behavior, states, commands, APIs, errors, and edge cases.

## Acceptance criteria
- [ ] Testable criterion 1.
- [ ] Testable criterion 2.

## Verification
- Unit/integration/manual checks that prove the criteria.

## Rollout / compatibility
- Migration, defaults, back-compat, and docs implications.

## Open questions
- Only non-blocking details.
```

Keep the spec behavior-first. Avoid implementation details unless they are externally visible or constrain acceptance.

## 4. Publish in the right place

Prefer native project workflow:

- If the user wants an issue and GitHub is available, draft a `gh_issue_create` title/body and show it before creating.
- If the repo uses local specs, write under the existing docs/specs directory.
- If neither is true, return the spec in the answer and ask where to save it.

Do not create issues or files without approval when the user only asked for a draft.

## 5. Connect to follow-on work

A spec is done when it can feed one of these next steps:

- `to-tickets` for multi-step delivery.
- `writing-plans` for a single implementation plan.
- `spec-code-review` as the spec axis during review.

Include enough acceptance criteria that later review can trace changed code and tests back to the spec.

## Quality bar

- The title is specific and less than 72 characters when it may become a GitHub issue.
- Every goal has at least one acceptance criterion.
- Every acceptance criterion is verifiable.
- Non-goals prevent obvious scope creep.
- The spec uses project domain terms, not new synonyms.
