# 1. Problem framing

This project investigates how content metadata can help prioritize content items for human review.

The unit of analysis is an individual content item. The system produces a ranked list of content items using model scores and human-readable reason codes.

The intended users are SEO analysts, content strategists, and editorial teams. They can use the ranked output to identify which content items may deserve attention first.

A false positive may cause an editor to spend time reviewing content that does not require attention. A false negative may leave a potentially declining content item unreviewed.

Machine learning is useful for this task because it can consider multiple content characteristics together and support a more systematic prioritization process. The output is intended for decision support only and does not make automatic content-refresh decisions.

# 2. Data safety

This project uses the anonymized `content_refresh_anonymized.csv` dataset from the FlyRank ML Internship dataset.

The analysis uses content metadata, including search volume, competition, cost per click (CPC), content type, main search intent, word count, character count, content age, and days since the last update.

The historical `trend_direction` field is used only to create the declining-content proxy label. The fields `trend_direction` and `trend_pct` are excluded from model features because they are directly related to the target definition and could introduce target leakage.

The pseudonymous identifiers `content_id` and `client_id` are excluded from predictive features. The client identifier is used only to create client-grouped validation splits, ensuring that clients in the training set do not overlap with clients in the grouped test set.

Client names, domains, private search queries, URLs, credentials, raw client-identifying exports, and content-level identifiers are excluded from public project artifacts.

# 3. Baseline

A rule-based baseline was implemented to provide a transparent comparison against the Logistic Regression model.

The baseline combines simple content-review signals, including low click-through rate, low engagement, older update dates, and weak average search position. Thresholds are calculated using training data to avoid using test-set information during baseline construction.

The baseline provides an understandable starting point because its rules can be inspected directly. The Logistic Regression model is evaluated against this baseline using Precision@K, which measures the proportion of relevant proxy-positive items among the top-ranked results.

# 4. Model / analysis

This project uses Logistic Regression for binary classification. It provides an interpretable model that can estimate the likelihood of a content item matching the historical declining-content proxy.

The target variable is `is_declining_proxy`, defined as:

- `1` when `trend_direction` is `down`.
- `0` otherwise.

The model uses the following features:

**Numeric features**
- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `content_age_days`
- `days_since_last_update`

**Categorical features**
- `competition_level`
- `content_type`
- `main_intent`

Numeric missing values are handled using median imputation with missingness indicators. Categorical missing values are represented as `Unknown`, and categorical features are one-hot encoded. Numeric features are standardized before model fitting.

The model uses balanced class weights and a fixed random seed of 42.

The model deliberately excludes `trend_direction` and `trend_pct` from its input features because they are related to the target definition. Pseudonymous identifiers and performance-related fields are also excluded from the predictive feature set.

Some of the stronger positive coefficients included `competition_level_Unknown` (+1.278), `main_intent_informational` (+0.478), `main_intent_commercial` (+0.346), and `days_since_last_update` (+0.237). These coefficients represent associations learned by the model and should not be interpreted as causal effects.

# 5. Evaluation

The model was evaluated using two validation strategies: a random row split and a client-grouped split.

The random split used a stratified 75:25 train-test split. It contained 22,500 training records and 7,500 test records. The client-grouped split used `GroupShuffleSplit` with a test size of 25% and a random seed of 42. It contained 22,885 training records and 7,115 test records, with 24 training clients and 8 test clients.

The random split had 31 clients appearing in both training and test sets. In contrast, the client-grouped split had zero client overlap. Therefore, the grouped split provides a more appropriate estimate of performance on previously unseen clients.

**Validation results**

| Metric | Random row split | Client-grouped split |
|---|---:|---:|
| Training records | 22,500 | 22,885 |
| Test records | 7,500 | 7,115 |
| Client overlap | 31 | 0 |
| Precision@10 | 0.70 | 0.60 |
| Precision@50 | 0.70 | 0.64 |
| Precision@100 | 0.66 | 0.55 |
| Test positive-class rate | 0.5420 | 0.5165 |

The random split produced higher Precision@K values, but it also allowed client overlap between training and testing. This may result in an optimistic estimate of generalization performance.

The client-grouped split is treated as the primary evaluation because it tests the model on clients that were not present during training. The two split results should be interpreted as validation-design sensitivity rather than a perfectly controlled comparison, because their test records and positive-class rates differ.

**Baseline comparison on the client-grouped test set**

| Metric | Rule-based baseline | Logistic Regression |
|---|---:|---:|
| Precision@10 | 0.50 | 0.60 |
| Precision@50 | 0.56 | 0.64 |
| Precision@100 | 0.59 | 0.55 |

Logistic Regression performs better than the baseline at Precision@10 and Precision@50, while the baseline performs better at Precision@100. This suggests that the model may be useful for prioritizing a smaller review list, but it does not outperform the baseline at every ranking depth.

On the 7,115-record grouped test set, Logistic Regression achieved an accuracy of 0.53. The positive proxy class contained 3,675 records, representing approximately 51.65% of the test set.

For the positive class, precision was 0.54, recall was 0.64, and the F1-score was 0.58. The confusion matrix contained 2,340 true positives, 1,445 true negatives, 1,995 false positives, and 1,335 false negatives.

The accuracy is only slightly above the positive-class base rate. Therefore, accuracy alone does not establish strong predictive discrimination. The Precision@K results provide a more relevant view of the model's usefulness for a ranked human-review workflow.

A feature leakage audit also confirmed that the target-related fields, pseudonymous identifiers, proxy-label columns, and excluded performance-related fields were not included in the final predictive feature set.

# 6. Interpretation

The model's coefficients indicate that the unknown competition category, informational and commercial search intent, and longer time since the last update are positively associated with the declining-content proxy.

Some of the stronger negative coefficients included navigational search intent (-0.559), low competition (-0.516), and content age (-0.322).

These coefficients describe patterns learned from the available dataset. For example, the positive coefficient for `competition_level_Unknown` may reflect a missing-data pattern rather than a direct business relationship. None of these associations should be treated as causal evidence.

The error analysis identified 1,995 false positives and 1,335 false negatives. Some false positives were older content items that had been updated relatively recently. Some false negatives were very old content items that the model did not classify as declining.

These errors show that content age and freshness alone do not fully explain the proxy label. The model may help identify items for further investigation, but an editor should consider additional context before deciding whether to refresh content.

The validation results also show that performance was lower under client-grouped validation than under random row validation. This highlights the importance of testing generalization across clients rather than relying only on random record-level splits.

Overall, the model should be treated as a directional prioritization signal. Human review remains necessary before any content-refresh decision.
# 7. Recommendation

The action playbook converts model scores into a ranked review queue for human users. The client-grouped test set contains 7,115 scored content items, ranked by their model scores.

Each item includes human-readable reason codes based on review signals such as low CTR, low engagement, older update dates, and weak average search position. These reason codes provide additional context for analysts instead of relying on the model score alone.

In the generated queue, the most common reason-code combination was low CTR and low engagement, appearing for 2,693 items. A further 1,610 items had low engagement as their listed warning signal, while 1,045 items had no listed warning signal. The presence of no listed warning signal does not necessarily mean that an item is performing well; it means that none of the defined rules was triggered.

The recommended workflow is to review higher-ranked items first, verify their content relevance and search intent, check the quality and completeness of the underlying data, and then make an editorial decision.

The model must not automatically delete content, rewrite or publish pages, create URL redirects, or guarantee ranking and traffic improvements. A model score alone is not sufficient evidence for taking action.

The model should be reviewed or retrained when input distributions or missingness change substantially, evaluation performance declines on later-period data, new content types or client populations appear, or human reviewers report poor recommendation quality.

The action playbook is therefore a decision-support workflow. Final content-refresh decisions remain with human analysts and editorial teams.
