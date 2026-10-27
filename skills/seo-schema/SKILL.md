---
name: seo-schema
description: Draft and review JSON-LD structured data for common web page types such as articles, products, organizations, events, and FAQs. Use when adding schema markup, checking @context and @type choices, or mapping supplied page facts into machine-readable data. Never invent unavailable values.
license: MIT
metadata:
  author: yottacode
  category: content
  tags: seo, schema, json-ld, structured-data
  runtime: none
  network: none
---

# SEO schema

Create a conservative JSON-LD draft from facts supplied by the user or present in reviewed content.

## Workflow

1. Identify the page type and list the facts available for that type.
2. Use `references/common-types.md` to choose a reasonable type and required or useful properties.
3. Generate syntactically valid JSON-LD with `@context` set to `https://schema.org` and an appropriate `@type`.
4. Omit unknown properties rather than filling them with guesses. Preserve user-provided values exactly unless asked to normalize them.
5. Explain assumptions and provide a validation checklist. Distinguish JSON syntax validity from search-feature eligibility.

## Output format

```markdown
# Structured data draft

## Assumptions
- ...

## JSON-LD
```json
{ ... }
```

## Validation checklist
- [ ] JSON parses successfully.
- [ ] `@context` and `@type` match the page.
- [ ] Every value is supported by supplied evidence.
- [ ] Dates, URLs, names, and identifiers are correct.
- [ ] The markup reflects visible page content.

## Missing values
- ...
```

## Safety rails

- Do not claim that markup guarantees rich results, rankings, or citations.
- Do not fabricate reviews, ratings, prices, authors, dates, images, identifiers, or URLs.
- Do not add schema that misrepresents visible content.
