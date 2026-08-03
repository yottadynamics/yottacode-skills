---
name: spec-code-review
description: Review a diff on two independent axes: repo standards and spec fidelity. Use for PR reviews, pre-merge checks, "review my branch", or any change where both code quality and "did we build the right thing?" matter.
license: MIT
metadata:
  author: yottacode
  inspired-by: https://github.com/mattpocock/skills/tree/main/skills/engineering/code-review
---

# Code review: two-axis

A good review answers two different questions:

1. **Standards** — is the change well-written for this repo?
2. **Spec** — does the change faithfully implement the requested behavior?

Keep those axes separate. A clean implementation of the wrong thing is still wrong; a correct feature with brittle code still needs work.

## 1. Establish the review base

Use the cheapest correct review surface first:

- Local branch or working tree: `code_review_context` or `git_branch_diff`.
- Existing PR: `gh_pr_review_context`.
- Specific files: targeted `git_diff_files` after the summary identifies risk.

Identify the originating intent: issue, PR body, plan, spec doc, user request, or commit message. If no spec exists, say so and review spec fidelity against the user's latest request only.

## 2. Standards axis

Check whether the diff follows the repo's engineering standards:

- Existing architecture, naming, error handling, logging, and test style.
- Small interfaces and clean seams where abstraction is justified.
- No unrelated cleanup or scope creep.
- Security-sensitive code uses defense-in-depth and safe defaults.
- Tests are deterministic, isolated, and meaningful.
- Docs changed when behavior, public API, or workflow changed.

Use language-server references or targeted grep when needed to verify call sites. Do not infer impact from one file when callers are easy to inspect.

## 3. Spec axis

Check whether the implementation matches intent:

- Every acceptance criterion is represented in code and tests.
- Edge cases from the request are handled or explicitly deferred.
- Naming and user-visible behavior match the spec.
- Migration / compatibility promises are kept.
- The diff does not solve a different problem than the one requested.

When the spec is ambiguous, classify the issue as `QUESTION`, not as a bug.

## 4. Use subagents when the review is large

For large diffs, split the axes:

- Send a `review` subagent for standards findings.
- Send a `code-verifier` or `review` subagent for one specific spec-fidelity concern.
- For independent areas, dispatch parallel reviewers by package or file group.

Do not duplicate the same review yourself after dispatching. Merge the findings and remove duplicates.

## 5. Findings format

Only report actionable findings. If an audit step found a false positive, say what it got wrong in one sentence.

```markdown
## Standards findings
- [CRITICAL|STRONG|NIT] `file:line` — problem
  - Why: ...
  - Fix: ...

## Spec findings
- [CRITICAL|STRONG|QUESTION] `file:line` — mismatch
  - Spec: ...
  - Observed: ...
  - Fix: ...

## Clean checks
- Standards checked: ...
- Spec checked against: ...
```

Severity rules:

- `CRITICAL`: incorrect behavior, security issue, data loss, broken build, or missing required acceptance criterion.
- `STRONG`: maintainability, test, compatibility, or UX issue that should be fixed before merge.
- `NIT`: small clarity or style issue that is safe to batch.
- `QUESTION`: material ambiguity that needs the author or product owner.

## Safety rails

- Cite file and line for every code finding.
- Do not request broad rewrites when a surgical fix addresses the issue.
- Do not mix personal taste with repo standards; name the existing convention when possible.
- Do not claim "LGTM" unless both axes were checked and no actionable findings remain.
