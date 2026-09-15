Capstone Report —
# Capstone Report — Google Search Ranking & Discoverability

- **Author:** Tayyba Ghaffar
- **Lane:** General AI Fluency — Google Search Ranking & Discoverability
- **Repo:** `tayyba6/ML_Internship`
- **Date:** September 2026
## 1. Problem framing

The capstone supports a content-review prioritization decision: which pages should be reviewed first for possible content improvement using signals available before the review decision.
The unit of analysis is a page within a client, represented by the combination of `client_hash_id` and `content_hash_id`. The output is a ranked review queue, where pages are ordered by their learned decline probability and can also be grouped into priority tiers.
A FlyRank editor could use the queue to decide which pages to inspect first for possible content-quality, search-intent, or performance issues. The model is intended to support human review rather than automatically recommend or apply content changes.
The cost of a wrong call is asymmetric. A false positive sends an editor to review a page that may not have experienced a decline, consuming review time. A false negative can cause a page experiencing a decline to be missed or reviewed later. Because editorial review capacity is limited, improving the ordering of the highest-priority pages is more useful than trying to classify every page perfectly.
Data and ML help because the dataset contains many page-level observations and multiple pre-decision search-performance signals. A learned model can identify combinations of these signals and produce a consistent ranking that can be compared with a transparent rule-based baseline.

## 2. Data safety

The analysis used page-level performance data from the FlyRank internship warehouse. The March 2026 feature window was used because these signals were available before the review decision cutoff of March 31, 2026. April 2026 clicks were used only to construct the evaluation outcome.
The main features used were:
- `gsc_impressions_march`
- `gsc_ctr_march`
- `gsc_avg_position_march`
- `ga4_sessions_march`
The target was constructed from March and April GSC clicks. A page was labeled as a decline when April clicks were lower than March clicks.
Pages with zero March clicks were excluded from the modeling population because a strict decline defined as `april_clicks < march_clicks` cannot occur when March clicks are zero and April clicks are non-negative. This left 68,837 eligible pages from the 176,738-page feature frame.
Several fields were deliberately excluded from the model. Outcome-derived fields such as `trend_direction`, `trend_pct`, `decline_label`, `april_clicks`, and other April outcome information were not used as predictive features because they would leak future information into the decision.
`client_hash_id` and `content_hash_id` were retained only for grouping, joining, and identifying rows within the analysis. They were not used as model features.
The decision cutoff was March 31, 2026, while April 2026 was treated as the outcome window. The leakage audit confirmed that the model features came from the March feature window and did not contain the April outcome.
GA4 sessions were substantially incomplete. Missing GA4 values were handled by median imputation inside the modeling pipeline rather than by inventing business measurements.
The public-facing analysis uses hashed identifiers only and excludes client names, URLs, private search queries, and other client-identifying information.

## 3. Baseline

The baseline was a transparent rule-based action score designed to prioritize pages showing a combination of high search exposure, lower CTR, and weaker average position.
For each eligible page, the March impressions, March CTR, and March average position were converted to percentile scores. The baseline action score combined:
- high impressions,
- lower CTR, and
- weaker average position.
The baseline was evaluated only on the same held-out test pages and using the same Precision@K metrics as the learned model. This makes the comparison a ranking comparison rather than a comparison between different datasets or evaluation procedures.
On the seed-42 client-grouped test set of 5,062 pages, the baseline achieved:
- Precision@20: **70%**
- Precision@50: **64%**
- Average Precision: **0.6808**
The test-set decline base rate was **68.31%**.
The baseline therefore provided a reasonable transparent starting point, but its ranking performance left room for improvement, particularly for a larger review queue.

## 4. Model / analysis

A shallow Decision Tree classifier was selected because the capstone is intended to produce an interpretable prioritization method rather than an opaque high-complexity model. A maximum tree depth of 3 limits the complexity of the learned rules and makes the resulting signal relationships easier to inspect.
The model used four March features:
1. `gsc_impressions_march`
2. `gsc_ctr_march`
3. `gsc_avg_position_march`
4. `ga4_sessions_march`
Missing feature values were handled using median imputation inside a scikit-learn pipeline. The model used `DecisionTreeClassifier(max_depth=3, random_state=42)`.
The target is a binary proxy for observed short-term search-performance decline: a page receives label 1 when April GSC clicks are lower than March GSC clicks.
The model is not intended to predict Google's ranking system or establish why a page declined. It learns a ranking signal for prioritizing pages for human review based on the available March observations.

## 5. Evaluation

The primary evaluation used a client-grouped 80/20 holdout. Pages from the same client were kept entirely within either the training or test partition so that the model was evaluated on clients it had not seen during training.
For the seed-42 split:
- Training pages: 63,775
- Test pages: 5,062
- Training clients: 35
- Test clients: 9
- Clients appearing in both sets: 0
- Test-set decline rate: **68.31%**
Precision@K was chosen because the practical decision is to inspect a limited number of pages first. Average Precision was also used to evaluate the overall quality of the ranking.
On the same held-out test set:
| Metric            | Baseline | Decision Tree |
| Precision@20      | 70%      | **80%**       |
| Precision@50      | 64%      | **92%**       |
| Average Precision | 0.6808   | **0.7388**    |
The model improved Precision@20 by 10 percentage points and Precision@50 by 28 percentage points relative to the baseline. Average Precision improved by 0.0580.
The model's ranking was also tested across five client-grouped holdouts using seeds 42, 7, 21, 99, and 123. Mean Precision@50 was **92.8%** for the model compared with **66.4%** for the baseline, an average improvement of **26.4 percentage points**. The model outperformed the baseline on all five splits.
### Error analysis
In the seed-42 Top-50 review queue, 46 of 50 pages were actual April click declines and 4 were false positives. This corresponds to the observed 92% Precision@50.
The broader classification error counts were 1,581 false positives and 18 false negatives at a 0.50 probability threshold. These counts are not the primary success criterion because the capstone's operational objective is ranking pages for review rather than assigning every page a perfect binary class.
The error analysis also showed that the decision tree produces a small number of probability tiers. Therefore, the scores should be interpreted as ranking/priority signals rather than precise individual probabilities.

## 6. Interpretation

The learned tree relied primarily on two of the four available features:
| Feature                 | Tree feature importance |
| `gsc_ctr_march`         | 63.1% |
| `gsc_impressions_march` | 36.9% |
| `gsc_avg_position_march`| 0.0% |
| `ga4_sessions_march`    | 0.0% |
The model therefore did not use average position or GA4 sessions in its learned splits, despite these features being available. This is a useful negative result: adding a feature to the modeling frame does not guarantee that the fitted model will find it useful.
The learned model also did not simply reproduce the hand-built baseline. The baseline explicitly combined impressions, CTR, and average position, while the learned tree relied on CTR and impressions. This suggests that the learned ranking found a different weighting of the available signals.
The priority-tier analysis showed an ordered relationship between model priority and observed April decline rate:
| Priority tier | Pages | Observed decline rate |
| Lower         | 261   | 51.7% |
| Medium        | 3,680 | 65.3% |
| High          | 903   | 79.6% |
| Highest       | 218   | 92.2% |
The increasing decline rate across tiers supports the use of the model as a prioritization signal.
However, these relationships are observational. Feature importance does not establish causation, and the model does not show that changing CTR or impressions would cause a page's future performance to change.

## 7. Recommendation

The recommended workflow is to use the learned model as a triage layer before human editorial review.
### Highest priority
Review pages at the top of the ranked queue first. The highest model tier contained 218 test pages with an observed decline rate of 92.2%. These pages should be inspected for possible content-quality, search-intent, relevance, or recent-performance issues.
### High priority
Review the next group of highly ranked pages after the highest-priority queue. The observed decline rate for this tier was 79.6%.
### Medium priority
Review these pages when additional editorial capacity is available. Their observed decline rate was 65.3%.
### Lower priority
Defer these pages when review capacity is limited unless additional business or editorial context indicates that they should be inspected.
A FlyRank editor could therefore use the output as a ranked work queue: start with the highest-ranked pages, inspect the underlying page and search context, and then decide whether an actual content change is appropriate.
The model should not automatically trigger content changes. Human review remains necessary because a measured click decline does not establish its cause and may reflect factors outside the page itself.
Confidence in the ranking result is supported by the improvement over the transparent baseline and by consistent improvement across five client-grouped holdouts. However, the result should not be treated as production-ready or as a guarantee for every future client.
The analysis is based on a March feature window and an April outcome window, so temporal generalization to other periods has not been fully tested. Client-level decline rates also vary substantially, which means performance may differ across clients.

## 8. Reproducibility

The analysis was developed in the `work/notebooks/capstone.ipynb` notebook using Python, DuckDB, pandas, NumPy, and scikit-learn.
The model uses a fixed random seed of `42` for the primary client-grouped holdout and for the Decision Tree. Robustness testing used client-grouped holdouts with seeds:
`42, 7, 21, 99, 123`
The primary model configuration was:
- `DecisionTreeClassifier(max_depth=3, random_state=42)`
- `SimpleImputer(strategy="median")`
The feature frame was constructed from the FlyRank internship warehouse, with March 2026 used as the feature window and April 2026 used as the outcome window.
The Hugging Face access token is stored in Google Colab Secrets and is not written into the notebook or report.
The final analysis should be rerun from the beginning of `work/notebooks/capstone.ipynb` before submission so that the reported numbers are confirmed against the current notebook outputs.

## Claims checklist

- Results are described as observed, measured, directional, or decision-support findings.
- Precision@K is reported alongside the test-set decline base rate.
- The model is compared with the baseline on the same held-out pages and metrics.
- No causal claims are made.
- The model is not described as predicting Google's ranking algorithm.
- Hashed identifiers are used only for grouping and row identification, not as model features.
- Client-identifying information, URLs, and private queries are excluded from the public-facing analysis.
- Final reported numbers are based on the fresh capstone notebook evaluation.
