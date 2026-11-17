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

### Classification performance and error analysis

The Logistic Regression model achieved an overall accuracy of 0.53 on 7,115 test records. The positive proxy class contained 3,675 records, representing a base rate of approximately 51.65%.

For the positive class, the model achieved precision of 0.54, recall of 0.64, and an F1-score of 0.58. The confusion matrix contained 2,340 true positives, 1,445 true negatives, 1,995 false positives, and 1,335 false negatives.

The model identifies a relatively large proportion of positive proxy cases, but it also produces a substantial number of false positives. Its accuracy is only slightly above the positive-class base rate, so accuracy alone does not establish strong predictive discrimination.

These results support cautious use as a prioritization aid, with human review required before any content-refresh decision.
## 4. Model / analysis

This project uses Logistic Regression because the task involves binary classification and the model provides interpretable coefficients that help explain which input features are associated with the declining-content proxy.

The target variable is a proxy label: `is_declining_proxy = 1` when `trend_direction` is `down`, and `0` otherwise.

The model uses the following features:

- Numeric: `search_volume`, `competition`, `cpc`, `word_count`, `char_count`, `content_age_days`, `days_since_last_update`
- Categorical: `competition_level`, `content_type`, `main_intent`

Numeric missing values are imputed using the median, with missingness indicators. Categorical missing values are represented as `Unknown`, and categorical features are one-hot encoded. Numeric features are standardized.

The model deliberately excludes `trend_direction` and `trend_pct` from its input features because they are related to the target definition. Pseudonymous identifiers are also excluded from predictive features.

The strongest positive coefficients included `competition_level_Unknown` (+1.278), `main_intent_informational` (+0.478), `main_intent_commercial` (+0.346), and `days_since_last_update` (+0.237). These coefficients describe associations in the fitted model, not causal effects.

## 6. Interpretation

The model's coefficients indicate that the unknown competition category, informational and commercial search intent, and longer time since the last update are positively associated with the declining-content proxy.

The strongest negative coefficients included navigational search intent (-0.559), low competition (-0.516), and content age (-0.322).

The error analysis identified 1,995 false positives and 1,335 false negatives. Some false positives were older content items that had been updated relatively recently. Some false negatives were very old content items that the model did not classify as declining.

These errors show that content age and freshness alone do not fully explain the proxy label. The results should be treated as directional signals for further human investigation.
