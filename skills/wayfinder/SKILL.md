---
name: wayfinder
description: Map work that is too large for one session into decisions, research tasks, and ordered delivery tickets. Use for multi-day features, migrations, architecture changes, or any request where the path is not yet knowable end-to-end.
license: MIT
metadata:
  author: yottacode
  inspired-by: https://github.com/mattpocock/skills/tree/main/skills/engineering/wayfinder
---

# Wayfinder

Wayfinder is for work where a normal plan would lie. When the destination is clear but the route depends on decisions you have not made yet, build a map instead of pretending to know every step.

Use this skill for large features, migrations, cross-cutting architecture work, or anything likely to exceed one agent session.

## 1. Define the destination

Write a short destination statement:

- What capability or outcome should exist when the work is done?
- Who benefits?
- What constraints are non-negotiable?
- What would make the effort a failure?

Keep this behavior- and value-oriented. Do not start with file changes.

## 2. Inventory known terrain

Gather only the context needed to map the route:

- Existing docs, ADRs, specs, and domain model.
- Relevant packages/modules and ownership boundaries.
- Current tickets/PRs that overlap.
- Risks: migration, security, compatibility, performance, UX, release timing.

Use subagents for independent research areas if the map spans multiple systems.

## 3. Identify decisions before tasks

List the decisions that determine the route. For each one:

- Question.
- Why it blocks implementation.
- Options under consideration.
- Evidence needed.
- Who/what can decide.
- Smallest artifact that resolves it: prototype, ADR, benchmark, spike, user answer.

Do not turn uncertain implementation into fake tickets. Decision tickets come first.

## 4. Build the route map

Create a staged map:

```markdown
## Destination
...

## Current terrain
- ...

## Decision tickets
1. <decision> — resolves <uncertainty>; output: ADR/prototype/spec update.

## Delivery tickets
1. <slice> — blocked by <decision/ticket>; proves <behavior>.

## Parallel work
- Workstreams that can proceed independently.

## Stop / revisit points
- Conditions that require replanning.
```

The map should make the next safe action obvious without pretending the final implementation details are known.

## 5. Publish and update

Use the repo's normal planning surface:

- GitHub issues for shared work when available and approved.
- Existing local planning docs when the repo uses them.
- A chat response only when the user asked for a quick map.

As decisions resolve, update the map. A stale wayfinding map is worse than no map because it gives false confidence.

## Quality bar

- The first next step is a concrete decision, research task, or tracer-bullet ticket.
- Every implementation ticket is blocked only by named decisions or prior tickets.
- The map shows where parallel work is safe.
- The map has explicit revisit points.
- The map is shorter than the work itself; it guides execution rather than becoming a second project.
