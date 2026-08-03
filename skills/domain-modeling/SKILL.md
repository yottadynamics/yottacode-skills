---
name: domain-modeling
description: Build and maintain a project's domain model — glossary, bounded contexts, invariants, workflows, and ADR candidates. Use when terminology is fuzzy, a feature crosses domains, a plan needs shared language, or repeated explanations should become durable project knowledge.
license: MIT
metadata:
  author: yottacode
  inspired-by: https://github.com/mattpocock/skills/tree/main/skills/engineering/domain-modeling
---

# Domain modeling

Domain modeling keeps the agent from inventing new words for old ideas. Before designing or changing code, make the project's real vocabulary explicit enough that future plans, tests, and docs use the same language.

Use this skill for non-trivial product or architecture work, especially when the request introduces new nouns, reuses overloaded terms, or touches multiple packages/services.

## 1. Find the current language

Start read-only. Look for the language the repo already uses:

- Product docs: `README.md`, `docs/`, `CONTEXT.md`, ADRs, architecture notes.
- Public interfaces: CLI commands, API routes, schemas, config keys, database tables.
- Tests: fixture names and scenario descriptions often reveal the real domain terms.
- Existing memories: use `memory_search` for durable project facts before re-deriving them.

If no domain document exists, say that plainly. Do not invent a full glossary from one code path.

## 2. Normalize terms

For each important concept, capture:

- **Preferred term** — the word the repo should use.
- **Definition** — one sentence, concrete enough to distinguish it from nearby terms.
- **Aliases / rejected terms** — words seen in code or conversation that should not spread.
- **Owner / context** — the bounded context where the term is valid.
- **Example** — a real scenario, test, endpoint, command, or file path.

When two terms appear to mean the same thing, challenge it. When one term means two things, split it.

## 3. Surface invariants and workflows

A useful domain model is not just nouns. Capture rules the code must preserve:

- State transitions and forbidden transitions.
- Permission or trust boundaries.
- Data ownership and lifecycle rules.
- External-system contracts and failure modes.
- Portability constraints and platform assumptions.

Prefer precise examples over generic wording. "A skill slug must match its directory name" is useful; "names should be valid" is not.

## 4. Decide where the knowledge lives

Pick the narrowest durable home:

- Update an existing `CONTEXT.md`, `docs/architecture.md`, or domain doc when the repo has one.
- Add or update an ADR when the decision is expensive to reverse.
- Save a project-scope memory when the fact is durable but no repo doc should change yet.
- Keep the result in the answer only when it is exploratory and not yet accepted.

Do not create a parallel documentation system unless the user asks for it. If the repo already has an ADR format, follow it.

## 5. Use the model in the work

Before planning or implementing, translate the task into the domain language:

- Name the primary bounded context.
- Name the terms that will appear in code, tests, docs, issue titles, and PR text.
- Call out any term you are intentionally retiring or avoiding.
- Tie acceptance tests to workflows or invariants from the model.

If the model changes during implementation, update the same durable source before calling the task done.

## Output format

For a modeling-only turn, respond with:

```markdown
## Domain model updates

| Term | Definition | Context | Notes |
|---|---|---|---|
| ... | ... | ... | ... |

## Invariants / workflows
- ...

## Documentation target
- Update: `path/to/doc.md`
- ADR needed: yes/no — why
- Memory candidate: yes/no — why
```

For implementation work, fold the model into the normal plan and verification steps instead of producing a separate report.

## Safety rails

- Do not rename code just to satisfy a new glossary unless the user approved a refactor.
- Do not persist speculative terms as memory or docs; mark them as proposed.
- Do not write ADRs for low-cost choices. Use ADRs for decisions future maintainers will otherwise re-litigate.
