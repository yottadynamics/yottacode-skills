---
name: to-tickets
description: Break a spec, plan, or large request into tracer-bullet implementation tickets with dependencies, acceptance criteria, and verification. Use when work is too large for one safe PR or one agent session.
license: MIT
metadata:
  author: yottacode
  inspired-by: https://github.com/mattpocock/skills/tree/main/skills/engineering/to-tickets
---

# To tickets

Tickets are not a task dump. Each ticket should be a small vertical slice that leaves the repo in a better, verifiable state and unblocks the next slice.

Use this skill after a spec exists, or when a conversation has grown too large for one implementation plan.

## 1. Identify the delivery graph

Read the spec or plan and extract:

- User-visible outcomes.
- Technical prerequisites.
- Risky decisions that need discovery before implementation.
- Natural seams by package, API, UI, migration, or workflow.
- Blocking relationships.

If there are unresolved decisions, create decision/research tickets before implementation tickets.

## 2. Prefer tracer bullets

A good implementation ticket cuts vertically:

- One small behavior change.
- Tests proving that behavior.
- Docs or release notes if user-visible.
- No unrelated cleanup.

Avoid horizontal tickets like "add all models", "add all handlers", "write all tests" unless they are genuinely standalone infrastructure.

## 3. Ticket template

Use this shape for every ticket:

```markdown
## <imperative title>

### Intent
One paragraph: what outcome this ticket creates and why it exists.

### Scope
- In: concrete files, behavior, tests, docs.
- Out: adjacent work not included.

### Dependencies
- Blocks: #...
- Blocked by: #...

### Acceptance criteria
- [ ] Criterion that can be tested or reviewed.

### Verification
- Exact test/build/manual check.

### Notes for implementer
- Existing helpers, domain terms, risks, or gotchas.
```

Titles should be short enough for GitHub issues and phrased as work to land, not vague themes.

## 4. Publish safely

When GitHub is available and the user wants issues:

1. Draft the issue titles and bodies first.
2. Show the list with dependencies.
3. Create issues with `gh_issue_create` only after approval.
4. Add dependency notes in the body using issue numbers once created if native blocking links are unavailable.

For local-only work, write a Markdown checklist in the repo's existing planning/docs location. Do not invent a new tracker if the repo already has one.

## 5. Keep the graph small

Default to 3-7 tickets. More than that usually means the work needs `wayfinder` first.

Merge tickets when they cannot be verified independently. Split tickets when one failure would leave half-completed behavior behind.

## Quality bar

- Every ticket has acceptance criteria and verification.
- Every dependency is named explicitly.
- At least one ticket creates user-visible or integration-visible behavior early.
- The first implementation ticket is safe to start without re-planning the whole project.
- The set avoids "big bang" merge risk.
