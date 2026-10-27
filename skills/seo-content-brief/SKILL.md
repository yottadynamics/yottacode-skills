---
name: seo-content-brief
description: Create an evidence-aware SEO content brief from a supplied topic, keyword, audience, or business goal. Use when planning an article, landing page, guide, or FAQ before drafting. Include intent, audience, outline, questions, internal-link ideas, and evidence gaps without pretending to have live keyword or competitor data.
license: MIT
metadata:
  author: yottacode
  category: content
  tags: seo, content, brief, intent, editorial-planning
  runtime: none
  network: optional-user-requested
---

# SEO content brief

Turn an approved topic or keyword into a practical brief that separates supplied facts from editorial suggestions.

## Workflow

1. Capture the supplied topic, audience, business goal, geography, language, format, and constraints.
2. Classify the likely intent as informational, navigational, commercial, transactional, or mixed. Explain uncertainty.
3. Define the primary promise, useful subquestions, audience knowledge level, and success criteria that do not depend on invented metrics.
4. Build an outline with a direct opening, logical sections, examples, objections, and a useful conclusion or next step.
5. Suggest internal links only from URLs or content areas the user supplies. Otherwise mark them as link opportunities.
6. List evidence gaps, claims needing review, and optional research tasks. Do not imply live keyword volume, rankings, or competitor analysis unless provided.

## Output format

```markdown
# Content brief

## Inputs and evidence boundary
- Supplied:
- Assumptions:
- Not researched:

## Audience and intent
- Audience:
- Primary intent:
- Secondary questions:

## Content goal
- Reader promise:
- Desired next step:

## Recommended outline
1. ...

## Editorial guidance
- Tone:
- Must include:
- Avoid:

## Internal-link opportunities
- ...

## Evidence gaps and verification queue
- ...
```

## Safety rails

- Never present model suggestions as keyword research, search volume, ranking data, or competitor findings.
- Avoid keyword stuffing and write for the reader's actual task.
- Preserve the requested audience, intent, geography, and compliance constraints.
