## 1. Problem framing

This project investigates how content metadata can help prioritize content items for human review.

The unit of analysis is an individual content item. The output is a ranked list of content items with a model score and human-readable reason codes.

The intended users are SEO analysts, content strategists, and editorial teams. They can use the ranked list to decide which content items should be reviewed first.

A false positive may cause an editor to spend time reviewing content that does not need attention. A false negative may leave a potentially declining content item unreviewed.

Machine learning is useful here because multiple content characteristics can be considered together to support prioritization. The output is decision-support, not an automatic content-refresh decision.

## 2. Data safety

This project uses the anonymized `content_refresh_anonymized.csv` dataset from the FlyRank ML Internship dataset.

The analysis uses content metadata, including search volume, competition, content type, main intent, word count, character count, content age, and days since the last update.

The historical `trend_direction` field is used only to create the declining-content proxy label. The fields `trend_direction` and `trend_pct` are excluded from model features because they are directly related to the target definition.

The pseudonymous identifiers `content_id` and `client_id` are not used as predictive features. They are used only for content traceability and client-grouped validation.

Client names, domains, private search queries, URLs, credentials, and raw client-identifying exports are excluded from public project artifacts.
