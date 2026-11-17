# Can Content Metadata Help Prioritize Content for Human Review?

## Abstract

Can content metadata help identify which content items should be reviewed first by editorial teams? This study uses an anonymized FlyRank ML Internship dataset containing 30,000 records and 44 columns to investigate this question. A Logistic Regression classifier was trained using content metadata and evaluated with client-grouped validation against a rule-based baseline. On the grouped test set, Logistic Regression achieved Precision@10 of 0.60 and Precision@50 of 0.64, compared with 0.50 and 0.56 for the baseline, while the baseline performed better at Precision@100. These results suggest that metadata-based ranking may support human review prioritization, but the model is not evidence of guaranteed content-refresh or search-performance improvements.

## 1. Introduction: The Problem

Content teams often manage large numbers of web pages and need to decide which items deserve attention first. Reviewing every page with equal priority can consume substantial editorial time.

This project investigates whether available content metadata can help rank content items for further human review. The intended users are SEO analysts, content strategists, and editorial teams.

The system produces a ranked list using model scores and human-readable reason codes. It is designed as a decision-support tool rather than an automated content-management system.

A false positive may cause an editor to spend time reviewing an item that does not require attention. A false negative may leave a potentially declining item unreviewed.

## 2. Data

The analysis uses the anonymized `content_refresh_anonymized.csv` dataset from the FlyRank ML Internship dataset.

The dataset contains 30,000 records and 44 columns. The analysis uses metadata such as:

- Search volume, competition, and cost per click (CPC).
- Content type and main search intent.
- Word count and character count.
- Content age and days since the last update.

The historical `trend_direction` field was used to construct the proxy target. The fields `trend_direction` and `trend_pct` were excluded from predictive features to avoid target leakage.

The pseudonymous identifiers `content_id` and `client_id` were excluded from predictive features. The client identifier was used only to create grouped validation splits.

No client names, domains, private search queries, URLs, credentials, raw client-identifying exports, or content-level identifiers are included in the public research artifacts.

## 3. Methodology

### 3.1 Target Definition

The binary target, `is_declining_proxy`, was defined as:

- `1` when `trend_direction` was `down`.
- `0` otherwise.

This is a historical proxy label, not a direct measurement of whether a future content refresh will succeed.

### 3.2 Predictive Features

The model used seven numeric features:

- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `content_age_days`
- `days_since_last_update`

It also used three categorical features:

- `competition_level`
- `content_type`
- `main_intent`

Numeric missing values were handled through median imputation with missingness indicators. Categorical missing values were represented as `Unknown` and one-hot encoded. Numeric features were standardized.

### 3.3 Model and Baseline

Logistic Regression was selected as the classification model. It used balanced class weights, a maximum iteration limit of 2,000, and a fixed random seed of 42.

A rule-based baseline combined review signals such as low CTR, low engagement, older update dates, and weak average search position. Its thresholds were calculated using training data.

### 3.4 Validation Design

Two validation strategies were evaluated:

1. Random stratified 75:25 train-test split.
2. Client-grouped split using `GroupShuffleSplit`, a 25% test size, and random seed 42.

The client-grouped split had zero client overlap between training and testing. It is therefore treated as the primary evaluation design for assessing performance on previously unseen clients.

A feature leakage audit checked that target-related fields and excluded identifiers were not included in the final predictive feature set.

## 4. Results

### 4.1 Validation Strategy Comparison

| Metric | Random split | Client-grouped split |
|---|---:|---:|
| Training records | 22,500 | 22,885 |
| Test records | 7,500 | 7,115 |
| Client overlap | 31 | 0 |
| Precision@10 | 0.70 | 0.60 |
| Precision@50 | 0.70 | 0.64 |
| Precision@100 | 0.66 | 0.55 |
| Positive-class rate | 54.20% | 51.65% |

The random split produced higher Precision@K values, but allowed clients to appear in both training and test sets. The grouped split provides a more appropriate test of generalization to unseen clients.

These results should be interpreted as sensitivity to validation design, not as a perfectly controlled comparison, because the test records and positive-class rates differ.

### 4.2 Model Versus Baseline

The following results use the same client-grouped test set.

| Metric | Rule-based baseline | Logistic Regression |
|---|---:|---:|
| Precision@10 | 0.50 | 0.60 |
| Precision@50 | 0.56 | 0.64 |
| Precision@100 | 0.59 | 0.55 |

Logistic Regression performed better at the top 10 and top 50 positions. The baseline performed better at the top 100 positions.

This indicates that the model may be more useful for prioritizing a smaller review list, but it does not outperform the baseline at every ranking depth.

### 4.3 Classification Results

On the grouped test set, Logistic Regression achieved:

- Accuracy: 0.53
- Positive-class precision: 0.54
- Positive-class recall: 0.64
- Positive-class F1-score: 0.58
- Positive-class rate: 51.65%

The confusion matrix contained 2,340 true positives, 1,445 true negatives, 1,995 false positives, and 1,335 false negatives.

Accuracy was only slightly above the positive-class base rate. Therefore, the results do not establish strong overall predictive discrimination.

### 4.4 Model Score Distribution

The ML-10 action playbook generated model scores for 7,115 content items in the client-grouped test set. The scores are used to rank items for review and should not be interpreted as calibrated probabilities.

![ML-10 Model Score Distribution](figures/ml10_score_distribution.png)

*Figure 1: Distribution of model scores across the ranked action queue. The scores provide a relative ranking signal rather than a calibrated probability of decline.*

## 5. Interpretation

The model learned associations between content metadata and the historical declining-content proxy.

Some of the stronger positive coefficients were:

- `competition_level_Unknown`: +1.278
- `main_intent_informational`: +0.478
- `main_intent_commercial`: +0.346
- `days_since_last_update`: +0.237

Some stronger negative coefficients were:

- `main_intent_navigational`: -0.559
- `competition_level_LOW`: -0.516
- `content_age_days`: -0.322

These coefficients describe model associations and must not be interpreted as causal effects. For example, the coefficient for unknown competition may reflect a missing-data pattern rather than a direct business relationship.

The grouped evaluation also produced 1,995 false positives and 1,335 false negatives. This reinforces the need for editorial context and human judgment.

## 6. Limitations and Honest Framing

This study has several limitations:

1. The target is a historical proxy derived from `trend_direction`, not a verified future outcome.
2. The model does not establish that refreshing content will improve rankings, traffic, or engagement.
3. Client-grouped validation uses only the available grouped test population and does not guarantee performance on every future client.
4. Accuracy of 0.53 is only slightly above the positive-class base rate of 51.65%.
5. Logistic Regression performs better than the baseline at Precision@10 and Precision@50, but not at Precision@100.
6. Model coefficients represent associations, not causal explanations.
7. Missing metadata may influence model scores and reason codes.

The model score should be treated as a ranking signal, not a calibrated probability or an instruction to take automatic action.

## 7. Ranked Recommendations

The ML-10 action playbook creates a ranked queue of 7,115 scored items from the client-grouped test set.

Each item receives human-readable reason codes based on review signals such as low CTR, low engagement, older update dates, and weak average search position.

The most frequent reason-code combination was low CTR and low engagement, appearing for 2,693 items. A further 1,610 items had low engagement as their listed warning signal, while 1,045 items had no listed warning signal.

Recommended workflow:

1. Start with higher-ranked items.
2. Review content relevance and search intent.
3. Check the quality and completeness of the underlying metadata.
4. Consider additional editorial and business context.
5. Let a human analyst decide whether a content update is appropriate.

The system must not automatically delete content, rewrite or publish pages, create redirects, or guarantee search-performance improvements.

Monitoring and possible retraining should be considered when input distributions or missingness change substantially, later-period evaluation declines, new client populations or content types appear, or human reviewers report poor recommendation quality.

### 7.1 Action Queue Reason Codes

The action playbook assigns reason codes to help analysts understand which review signals were triggered for each item.

![ML-10 Reason Code Frequency](figures/ml10_reason_codes.png)

*Figure 2: Most frequent reason-code combinations in the generated action queue. These signals support human review and do not independently establish that content needs updating.*

The presence of no listed warning signal does not necessarily mean an item is performing well. It only indicates that none of the defined reason-code rules was triggered.

## 8. Reproducibility

The project uses Python notebooks and a fixed random seed of 42 for the reported model configuration and validation workflow.

The main workflow includes:

1. Loading the anonymized dataset.
2. Constructing the declining-content proxy target.
3. Excluding target-related fields and pseudonymous identifiers from predictive features.
4. Preprocessing numeric and categorical features.
5. Training Logistic Regression.
6. Evaluating the model against the rule-based baseline.
7. Generating the ranked action playbook and aggregate metrics.

The [ML-10 Action Playbook notebook](notebooks/w07_action_playbook.ipynb) is available in the project repository.

The aggregate metrics are stored in `work/outputs/ml10_action_playbook_metrics.json`. Supporting figures are stored in `work/figures/`.

The dataset and row-level action queue are not included in the public repository.

**Project repository:** [FlyRank ML Workspace](https://github.com/suyashverma0/flyrank-ml-workspace2)

## 9. Acknowledgments and Data Credit

This project was completed as part of the FlyRank ML Internship.

The analysis uses the anonymized content-refresh dataset provided for the internship. Care was taken to exclude client-identifying information and row-level content identifiers from public project artifacts.

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
