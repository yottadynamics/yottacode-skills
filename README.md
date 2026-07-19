# yottacode skills

Official public Agent Skills for yottacode.

This repository contains the free, curated skill catalog that yottacode can install with the `official/<name>` source shortcut. Skills live under the `skills/` directory for compatibility with the broader skills.sh / `npx skills` ecosystem. Each skill directory contains a `SKILL.md` file plus optional `scripts/`, `references/`, and `assets/` directories.

[![skills.sh](https://skills.sh/b/yottadynamics/yottacode-skills)](https://skills.sh/yottadynamics/yottacode-skills)

## Layout

```text
skills/
  test-driven-development/
    SKILL.md
  verification-before-completion/
    SKILL.md
```

## Install

From yottacode, install an official skill by slug:

```bash
yottacode skills install official/test-driven-development
```

The shortcut expands to the GitHub source:

```text
yottadynamics/yottacode-skills/skills/test-driven-development
```

Existing GitHub installs still work directly:

```bash
yottacode skills install yottadynamics/yottacode-skills/skills/test-driven-development
```

The repository is also compatible with Vercel's `skills` CLI discovery:

```bash
npx skills add yottadynamics/yottacode-skills --list
npx skills add yottadynamics/yottacode-skills --skill test-driven-development
```

## Authoring rules

- Each skill lives under `skills/<slug>/SKILL.md`.
- The `name` frontmatter must match `<slug>`.
- Keep descriptions keyword-rich; yottacode uses descriptions to decide when a skill applies.
- Keep skill bodies focused and move long supporting material into `references/`.
- Do not include secrets, credentials, or private infrastructure details.

## Contributing

Contributions are welcome when they fit the public/free official catalog.

### Good candidates

- Reusable engineering workflows that apply across many repositories.
- Debugging, testing, review, documentation, planning, and delivery playbooks.
- Skills that work with yottacode's built-in tools without requiring private services.
- Skills with clear, vendor-neutral wording and no organization-specific assumptions.

### Not a fit for this repo

- Paid, customer-specific, or private skill packs.
- Runtime plugins or executable extensions; those belong in a separate plugin repository.
- Skills that require secrets, private infrastructure, or unreleased internal APIs.
- Thin wrappers around one command unless the surrounding workflow adds durable value.

### Pull request checklist

1. Create a new directory named with a lowercase kebab-case slug under `skills/`, for example `skills/flaky-test-debugging/`.
2. Add `SKILL.md` with frontmatter:

   ```markdown
   ---
   name: flaky-test-debugging
   description: Use this when diagnosing an intermittently failing test, especially when repetition, timing, shared state, or order dependence may be involved.
   ---
   ```

3. Make sure the `name` exactly matches the directory name.
4. Keep the description specific and trigger-oriented; it is what yottacode uses to decide when the skill applies.
5. Keep the main body concise. Put longer examples or background material in `references/`.
6. Add any helper scripts under `scripts/` and document what they do before asking the agent to run them.
7. Validate locally before opening a PR:

   ```bash
   yottacode skills validate ./skills/flaky-test-debugging
   ```

8. Do not include credentials, customer data, private URLs, or machine-specific paths.

### Review expectations

A skill should read like a durable playbook, not a transcript. Reviewers will check that it has a clear trigger, generalizes beyond one project, stays safe by default, and uses yottacode terminology/tool names where tool references are necessary.

## Public vs private packs

This repository is for public/free official skills. Paid, customer-specific, or private skill packs should live in separate private repositories and can use the same `SKILL.md` layout. Runtime plugins are separate from skills and should live in a plugin-specific repository with their own versioning, signing, permissions, and compatibility rules.
