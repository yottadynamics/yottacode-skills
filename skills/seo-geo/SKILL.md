---
name: seo-geo
description: Improve supplied content for Generative Engine Optimization (GEO), meaning clearer, self-contained, evidence-aware content for answer-oriented retrieval systems. Use when making passages easier to understand, quote, summarize, or attribute. Do not promise AI citations, recommendations, visibility, or rankings.
license: MIT
metadata:
  author: yottacode
  category: content
  tags: seo, geo, generative-engine-optimization, answerability
  runtime: none
  network: none
---

# SEO GEO

Revise supplied content to improve answerability and sourceability without inventing evidence.

## Workflow

1. State that GEO means Generative Engine Optimization and define the evidence boundary.
2. Identify the intended audience, question, entity, and claims the passage should answer.
3. Rewrite for a direct answer first, clear headings, concise definitions, explicit scope, and useful context.
4. Preserve supplied facts and flag claims that need a source, date, owner, or subject-matter review.
5. Add a compact question-and-answer or summary section only when it improves comprehension and remains faithful to the source.
6. Review for ambiguity, unsupported superlatives, hidden assumptions, and citation opportunities.

## Output format

```markdown
# GEO content review

## Target question and audience
...

## Revised passage
...

## Evidence and uncertainty
- Supported by supplied material:
- Needs verification:
- Sources to add or confirm:

## Change notes
- ...
```

## Safety rails

- GEO is guidance for clarity and answerability, not a ranking or citation mechanism.
- Never manufacture sources, quotes, statistics, credentials, consensus, or first-hand experience.
- Do not add claims merely because they may be attractive to an answer engine.
- Keep important qualifications, dates, conditions, and disagreements visible.
