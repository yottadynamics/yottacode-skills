---
name: skill-authoring
description: Write and edit yottacode Agent Skills so they trigger reliably, stay concise, validate cleanly, and behave predictably. Use when creating a new SKILL.md, reviewing a skill, deciding skill scope, or turning a workflow into a reusable playbook.
license: MIT
metadata:
  author: yottacode
  inspired-by: https://github.com/mattpocock/skills/tree/main/skills/productivity/writing-great-skills
---

# Skill authoring

A skill turns a good workflow into a predictable process. The goal is not to force the same output every run; it is to make the agent follow the same useful shape whenever the skill applies.

Use this skill when creating or revising a yottacode skill under `skills/<slug>/SKILL.md`, `.yottacode/skills/<slug>/SKILL.md`, or `~/.yottacode/skills/<slug>/SKILL.md`.

## 1. Decide whether this should be a skill

Create a skill when the workflow is reusable and benefits from a named playbook:

- It has a recognizable trigger.
- It has a repeatable process or checklist.
- It applies across more than one repo, task, or session.
- It changes agent behavior beyond generic advice.

Do not create a skill for one-off task instructions, secrets, customer-specific runbooks, stale project state, or a thin wrapper around one command unless the workflow around the command is the value.

## 2. Pick a tight slug and scope

Use a lowercase kebab-case slug that names the work, not the implementation detail:

- Good: `domain-modeling`, `security-auditor`, `to-spec`.
- Weak: `misc-helper`, `better-coding`, `github-stuff`.

The directory and frontmatter `name` must match exactly. If the skill overlaps an existing skill, either merge the new behavior into the existing one or pick a name that makes the distinction obvious.

## 3. Write trigger-rich frontmatter

Every yottacode skill needs valid frontmatter:

```markdown
---
name: skill-authoring
description: Write and edit yottacode Agent Skills so they trigger reliably, stay concise, validate cleanly, and behave predictably. Use when creating a new SKILL.md, reviewing a skill, deciding skill scope, or turning a workflow into a reusable playbook.
license: MIT
metadata:
  author: yottacode
---
```

Description rules:

- Start with what the skill does.
- Include distinct trigger branches with `Use when...` phrasing.
- Keep it specific enough for model invocation but short enough not to waste context.
- Avoid synonym stuffing. Multiple phrasings for the same trigger are duplication.
- Avoid private, employer-specific, or machine-specific details.

## 4. Structure the body as a playbook

Prefer ordered steps with completion criteria. A reader should know what to do first, what done looks like, and where to stop.

A strong body usually has:

1. Purpose: what predictable behavior this skill creates.
2. Trigger context: when to use it and when not to.
3. Process: ordered steps, each ending in a checkable result.
4. Output format: the shape of the final answer or artifact.
5. Safety rails: what to avoid or ask before doing.
6. Validation: how to check the skill itself when changed.

Keep the main `SKILL.md` concise. Move long examples, templates, scripts, or background into sibling `references/`, `scripts/`, or `assets/` files and link them from the body.

## 5. Make it yottacode-native

Name yottacode surfaces directly when they matter:

- `todo_write` for visible multi-step execution.
- Plan mode for read-only planning before implementation.
- `Skill` for composing with another playbook.
- `Agent`, `dispatch`, and `integrate` for subagents and parallel work.
- Typed GitHub tools for issues and PRs instead of shelling out.
- `memory_search` / `memory_save` when durable knowledge should be reused.
- `yottacode skills validate ./skills/<slug>` for validation.

Do not over-specify tool calls. A skill should guide behavior; it should not become brittle pseudo-code for every possible environment.

## 6. Prune aggressively

Review every sentence with the no-op test: would behavior change if this sentence were removed?

Cut:

- Generic advice the agent already follows.
- Duplicated rules stated in multiple sections.
- Stale caveats from earlier revisions.
- Long prose that should be a template or reference file.
- Negative instructions that can be phrased as positive target behavior.

Keep hard prohibitions only when safety requires them, and pair them with what to do instead.

## 7. Validate before calling it done

For each changed skill:

```bash
yottacode skills validate ./skills/<slug>
```

For catalog changes, also check:

- The README skill table includes the new or renamed slug.
- Any starter-pack references use the current slug.
- `grep` finds no stale old slug after a rename.
- The skill contains no secrets, customer data, private URLs, or forbidden brand/tool references.
- The license/source attribution is accurate when adapted from another public skill.

## Output format for reviews

When reviewing a skill, report:

```markdown
## Skill review

- Verdict: ready / needs changes
- Trigger clarity: ...
- Scope fit: ...
- Body shape: ...
- yottacode fit: ...
- Validation: ...

## Required changes
- ...
```

When authoring a new skill, produce the file under `skills/<slug>/SKILL.md`, update the README catalog, then validate it.

## Quality bar

A skill is ready when:

- Its trigger is obvious from the description.
- Its body changes behavior, not just tone.
- The process has checkable stopping points.
- It is short enough to load without drowning the task.
- It composes with yottacode tools and approval boundaries.
- It validates cleanly.
