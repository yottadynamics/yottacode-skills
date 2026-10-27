---
name: seo-audit
description: Audit user-provided web pages, files, or pasted content for technical SEO and on-page quality. Use when reviewing titles, headings, links, metadata, accessibility-adjacent discoverability, structured content, or evidence gaps. Do not imply live fetching, ranking data, traffic data, or completed checks unless the evidence is present.
license: MIT
metadata:
  author: yottacode
  category: content
  tags: seo, audit, technical-seo, on-page
  runtime: none
  network: optional-user-requested
---

# SEO audit

Produce a prioritized, evidence-based SEO audit from the material the user supplies.

## Workflow

1. Establish the evidence boundary. Record whether content was pasted, read from files, or fetched through an approved tool. If a URL was not actually fetched, say so.
2. Inspect the page or content against the checklist in `references/checklist.md`. Separate observed facts, likely issues, and recommendations.
3. Prioritize findings by impact and confidence. Prefer a short list of actionable findings over exhaustive speculation.
4. Identify missing evidence that prevents a confident conclusion, such as missing rendered HTML, redirects, robots policy, analytics, search data, or backlinks.
5. End with an ordered remediation plan and a verification checklist.

## Output format

```markdown
# SEO audit

## Evidence boundary
- Supplied/fetched material:
- Not checked:

## Executive summary
...

## Prioritized findings
### P1 — finding
- Evidence:
- Why it matters:
- Recommended action:
- Confidence: high|medium|low

## Evidence gaps
- ...

## Verification checklist
- [ ] ...
```

## Safety rails

- Never invent crawl results, rankings, traffic, competitors, citations, or performance metrics.
- Treat SEO recommendations as hypotheses to validate, not guarantees.
- Do not publish or mutate a site. Ask before making repository or content changes.
