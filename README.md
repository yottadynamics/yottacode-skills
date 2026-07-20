# yottacode skills

Official public Agent Skills for yottacode.

This repository contains the free, curated skill catalog that yottacode can install with the `official/<name>` source shortcut. Skills live under the `skills/` directory for compatibility with the broader skills.sh / `npx skills` ecosystem.

Each skill directory contains a `SKILL.md` playbook plus optional `scripts/`, `references/`, and `assets/` directories for supporting material.

## Available skills

| Skill | Purpose |
|---|---|
| `brainstorming` | Clarifies scope, constraints, and trade-offs before planning a non-obvious change. |
| `cve-explainer` | Explains CVEs from public sources with severity, impact, exposure, and remediation guidance. |
| `diagnose` | Debugs reported failures through reproduce, minimize, hypothesize, instrument, fix, and regression-test. |
| `dockerfile-review` | Reviews Dockerfiles for correctness, caching, image size, security, and reproducibility. |
| `documentation-and-adrs` | Captures architectural decisions, why-comments, and durable project documentation. |
| `executing-plans` | Executes an approved plan step-by-step with checkpoints and visible progress. |
| `git-investigation` | Uses Git history, blame, pickaxe, and bisect to answer historical code questions. |
| `handoff` | Writes concise handoff notes for continuing work in a fresh session. |
| `improve-codebase-architecture` | Reviews module or service structure for coupling, cohesion, boundaries, and public surface. |
| `performance-profiler` | Finds and fixes performance problems with baseline measurements and profiling. |
| `prototype` | Builds explicitly disposable prototypes to answer design or API questions. |
| `receiving-code-review` | Verifies and applies PR review feedback without blindly accepting bad comments. |
| `remote-ops` | Provides SSH, scp, rsync, and port-forwarding patterns for remote operations. |
| `security-auditor` | Reviews code for vulnerability patterns and defense-in-depth gaps. |
| `security-scanner` | Runs Trivy-based scans for CVEs, misconfigurations, secrets, licenses, and SBOMs. |
| `test-driven-development` | Guides feature and bug-fix work through the red-green-refactor loop. |
| `verification-before-completion` | Requires concrete verification before calling a fix or feature done. |
| `webapp-testing` | Drives local web apps through headless browser checks. |
| `writing-plans` | Produces verifiable implementation plans for multi-file engineering tasks. |

## Layout

Each skill follows this directory shape:

```text
skills/
  <skill-slug>/
    SKILL.md
    scripts/      # optional helper scripts
    references/   # optional long-form supporting material
    assets/       # optional images or other static files
```

For example:

```text
skills/
  test-driven-development/
    SKILL.md
  verification-before-completion/
    SKILL.md
```

## Install

Prerequisite: install yottacode and make sure the `yottacode` command is on your `PATH`.

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

The repository is also compatible with Vercel's `skills` CLI. Use `--list` to discover skills and `--skill <slug>` to install one:

```bash
npx skills add yottadynamics/yottacode-skills --list
npx skills add yottadynamics/yottacode-skills --skill test-driven-development
```

## Authoring rules

- Each skill lives under `skills/<slug>/SKILL.md`.
- The `name` frontmatter must match `<slug>`.
- Keep descriptions keyword-rich; yottacode uses descriptions to decide when a skill applies.
- Keep skill bodies focused and move long supporting material into `references/`.
- Do not include secrets, credentials, customer data, unreleased APIs, or machine-specific paths.

## Contributing

Contributions are welcome when they fit the public/free official catalog.

### Good candidates

- Reusable engineering workflows that apply across many repositories.
- Debugging, testing, review, documentation, planning, and delivery playbooks.
- Skills that work with yottacode's built-in tools without requiring paid services.
- Skills with clear, vendor-neutral wording and no organization-specific assumptions.

### Not a fit for this repo

- Paid or customer-specific skill packs.
- Runtime plugins or executable extensions; those belong in a separate plugin repository.
- Skills that require secrets, customer infrastructure, or unreleased internal APIs.
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

8. Confirm you have the rights to submit the material under the repository's MIT license.
9. Do not include credentials, customer data, unreleased URLs, or machine-specific paths.

### Review expectations

A skill should read like a durable playbook, not a transcript. Reviewers will check that it has a clear trigger, generalizes beyond one project, stays safe by default, and uses yottacode terminology/tool names where tool references are necessary.
