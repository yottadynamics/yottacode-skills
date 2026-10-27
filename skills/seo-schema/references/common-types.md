# Common schema types

Use the narrowest type supported by the page and evidence. These are drafting cues, not a guarantee that a search engine will display a feature.

| Page situation | Common type | Typical facts to map |
|---|---|---|
| Editorial article or post | `Article`, `NewsArticle`, or `BlogPosting` | headline, description, author, datePublished, dateModified, image, mainEntityOfPage |
| Product detail | `Product` | name, description, image, brand, sku, offers, aggregateRating only when supplied |
| Organization profile | `Organization` or a more specific subtype | name, url, logo, sameAs, contact information when public and supplied |
| Event page | `Event` or a specific event subtype | name, startDate, endDate, location, organizer, offers when supplied |
| Frequently asked questions | `FAQPage` | visible questions and accepted answers; do not add hidden or invented questions |
| How-to instructions | `HowTo` | name, description, steps, tools, supplies, duration when actually documented |
| Person profile | `Person` | name, url, image, jobTitle, worksFor, sameAs when supplied |

## Drafting rules

- Prefer one primary entity and connect supporting entities with explicit relationships.
- Use canonical absolute URLs only when supplied or confidently derived from the reviewed page.
- Use ISO 8601 date values only when the source gives an unambiguous date and timezone policy.
- Keep `description` and `answer` text aligned with visible page content.
- Treat ratings, prices, availability, reviews, and publication dates as evidence-bearing fields.
