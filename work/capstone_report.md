# Capstone Report — Refresh / Content Opportunity Scoring

* **Author:** Supreet Chinnannavar
* **Lane:** Refresh / Content Opportunity Scoring
* **Repo:** SupreetOP/Flyrank
* **Date:** August 31, 2026

## 0. Abstract

This study examines whether observed content-performance signals can support the prioritization of content items for human review. The analysis uses the FlyRank ML Internship dataset, with March 2026 performance signals used as features and whether a content item received at least one Google Search Console click in April 2026 used as the outcome. A Random Forest classifier using five observed search and Analytics features was evaluated against a transparent rule-based baseline on the same held-out validation data. The Random Forest measured a ROC-AUC of 0.9312 and a PR-AUC of 0.8397, compared with 0.8447 and 0.4858 for the baseline, while accuracy was 0.8762 compared with 0.8325. The resulting ranked output is intended as directional decision-support for human review and not as proof that refreshing a content item will cause additional clicks.

## 1. Problem framing

The decision supported by this analysis is which content items should receive earlier human review for a possible refresh opportunity.

The unit of analysis is a client-content item pair observed over a monthly performance window. March 2026 performance signals are used as the feature window, and April 2026 Google Search Console clicks are used as the outcome window.

The model produces a ranked decision-support output that can help prioritize content items for review. The Week-4 rule-based baseline and the Week-7 action playbook provide interpretable reason codes and actions alongside the learned model output.

A human reviewer can use the ranking to decide which content items to inspect first. The reviewer should then consider the actual content, search intent, search-result context, seasonality, data quality, and other relevant editorial context before deciding whether a refresh is appropriate.

The cost of a wrong call is primarily inefficient use of editorial review and content resources. A false positive may cause time to be spent reviewing a page that does not represent a useful opportunity, while a false negative may cause a potentially useful review candidate to receive less attention.

Data and ML are useful because the warehouse contains a large number of content-performance observations. A consistent scoring and ranking approach can measure relationships between historical performance signals and a defined future outcome and help prioritize a large review queue.

The output is therefore intended as **decision-support**, not an automated content-management system and not evidence that a particular refresh will cause an increase in future clicks.


## 2. Data safety

The analysis uses the FlyRank internship warehouse table `fact_content_daily_performance`.

The observed warehouse range was:

* January 27, 2025 to June 30, 2026
* 78,835,655 warehouse rows

The modeling windows were:

* **Feature window:** March 2026
* **Outcome window:** April 2026

The final model used five features:

* `gsc_impressions`
* `gsc_clicks`
* `gsc_avg_position`
* `ga4_pageviews`
* `ga4_engaged_sessions`

Client and content identifiers were deliberately excluded from model features. The pseudonymous identifiers are used only to identify/group content observations in the analysis and action queue.

The following fields were also excluded because they were not required for the final feature vector, could represent availability metadata, could duplicate other measures, or were not appropriate as predictive inputs for this task:

* `client_has_gsc`
* `client_has_ga4`
* `gsc_data_available`
* `ga4_data_available`
* `gsc_sum_position`
* `ga4_sessions`
* `ga4_users`
* `ga4_total_engagement_sec`
* `sessions_organic`
* `sessions_direct`
* `sessions_referral`
* `sessions_social`
* `sessions_paid`
* `sessions_ai`
* `ai_chatgpt`
* `ai_perplexity`
* `ai_gemini`
* `ai_copilot`
* `ai_claude`
* `ai_meta`
* `ai_other`
* `scroll_events`
* `report_date`
* `month`

Future outcome information was not included in the model feature set. The target `future_gsc_clicks` was used only to define the outcome and was not supplied as a model input.

Potential label-derived fields such as `trend_direction` and `trend_pct` were also treated as inappropriate predictive inputs because they could encode information derived from outcomes or future changes.

No client names, private queries, or other directly identifying client information are included in the public analysis.

## 3. Baseline

The first comparison system was a transparent Week-4 rule-based baseline.

The baseline identifies a potential refresh opportunity when:

* impressions are at least 100, and
* CTR is below 2%.

The baseline score increases with observed impressions and lower CTR among items satisfying the rule. Items not meeting the rule receive the `LOW_SEARCH_SIGNAL` reason code and a `MONITOR` action.

This is a fair comparison because the baseline is simple, interpretable, and based on the same general historical performance signals used to identify content opportunities. The Random Forest was evaluated against the baseline using the same held-out validation data and metrics.

Measured Week-4 baseline results:

| Metric   | Week-4 Rule Baseline |
| -------- | -------------------: |
| ROC-AUC  |               0.8447 |
| PR-AUC   |               0.4858 |
| Accuracy |               0.8325 |

The baseline therefore provides a transparent reference point for assessing whether the learned model provides additional measured discrimination.

## 4. Model / analysis

The model is a Random Forest classifier.

Random Forest was selected because the task involves a binary outcome and potentially nonlinear relationships between observed performance signals. It also provides feature-importance information that can be used for interpretation.

The final feature set contains:

1. `gsc_impressions`
2. `gsc_clicks`
3. `gsc_avg_position`
4. `ga4_pageviews`
5. `ga4_engaged_sessions`

The target is defined as:

**April 2026 Google Search Console clicks > 0.**

The resulting modeling data contained 331,436 observations:

* Positive target: 62,098
* Negative target: 269,338
* Positive rate: 18.74%
* Negative rate: 81.26%

Client and content identifiers, future outcome fields, availability metadata, and other non-selected warehouse fields were excluded from the final model feature vector.

## 5. Evaluation

The feature and outcome windows were separated in time: March 2026 signals were used to define the features and April 2026 clicks were used to define the outcome. This prevents the April outcome from being used directly to construct the March feature vector.

The resulting modeling frame was divided into training and held-out validation data while preserving the observed target proportion:

* Training rows: 265,148
* Validation rows: 66,288
* Training positive rate: 18.74%
* Validation positive rate: 18.74%

The evaluation compares the Random Forest and Week-4 baseline on the same validation set.

| Method               |    ROC-AUC |     PR-AUC |   Accuracy |
| -------------------- | ---------: | ---------: | ---------: |
| Week-4 Rule Baseline |     0.8447 |     0.4858 |     0.8325 |
| Random Forest        | **0.9312** | **0.8397** | **0.8762** |

Measured improvement of the Random Forest over the baseline:

* ROC-AUC: +0.0865
* PR-AUC: +0.3539
* Accuracy: +0.0437

The target base rate was 18.74%, meaning the majority class represented 81.26% of observations. Accuracy should therefore not be interpreted alone. ROC-AUC and PR-AUC provide more useful measures of discrimination for this imbalanced classification task.

### Error analysis

There were 4,881 disagreements between the model and the Week-4 baseline.

The confusion matrix for the Random Forest was:

|          | Predicted 0 | Predicted 1 |
| -------- | ----------: | ----------: |
| Actual 0 |      47,577 |       6,291 |
| Actual 1 |       1,918 |      10,502 |

Observed false-positive examples included pages with substantial impressions and clicks but no positive April outcome. This shows that strong historical search visibility does not guarantee the defined future outcome.

Observed false-negative examples included pages with little or no March search activity that nevertheless received many April clicks. These examples illustrate a limitation of using historical performance signals alone: demand can change, and future search activity may not always be well represented by the previous month's signals.

The errors therefore support treating the model as decision-support rather than as a definitive refresh decision.

## 6. Interpretation

The measured feature importances from the Random Forest were:

| Feature                | Importance |
| ---------------------- | ---------: |
| `gsc_impressions`      |     0.4875 |
| `gsc_clicks`           |     0.3472 |
| `ga4_pageviews`        |     0.0867 |
| `gsc_avg_position`     |     0.0776 |
| `ga4_engaged_sessions` |     0.0010 |

The model relied most strongly on observed Google Search Console impressions and clicks. Analytics pageviews and average position contributed less, while engaged sessions had very low measured feature importance in this model.

This suggests that the observed search-performance signals contained most of the predictive information used by the Random Forest for this particular target and validation setup.

An important limitation is that feature importance does not establish causality. For example, the high importance of impressions does not mean that increasing impressions would necessarily cause more future clicks.

The observed false-negative cases were also informative. Some content items had little or no March search activity but subsequently received substantial April clicks. This indicates that the selected features do not capture every source of future demand.

## 7. Recommendation

The ranked action queue is intended to help editors prioritize human review.

### Priority 1 — REFRESH review

The Week-4 action-playbook rule identifies items with:

* at least 100 impressions, and
* CTR below 2%.

These items receive the reason code:

`HIGH_IMPRESSIONS_LOW_CTR`

There were 100,660 such items in the ranked queue, representing 30.37% of the 331,437 content items.

These items can be reviewed earlier because they show observed search visibility together with relatively low CTR under the rule.

### Priority 2 — MONITOR

Items that do not satisfy the refresh rule receive:

`LOW_SEARCH_SIGNAL`

There were 230,777 such items, representing 69.63% of the queue.

These items should generally receive lower priority under this particular action-playbook rule unless additional evidence suggests a review is warranted.

### Human review

Before taking action, an editor should check:

* search intent and relevance;
* the current search-result context;
* content quality and completeness;
* seasonality and changes in demand;
* whether the page has recently changed;
* data availability or quality issues;
* other known business or editorial context.

The queue is directional decision-support. It does not establish that refreshing an item will increase clicks.

### No-go cases

The following should not be automated from this analysis:

* publishing content;
* deleting content;
* redirecting URLs;
* changing metadata;
* changing titles or descriptions;
* rewriting content;
* making other irreversible content changes.

The final action remains a human decision.

## 8. Reproducibility

The analysis is organized across the notebooks under `work/notebooks/`, with the capstone notebook assembling the final paper artifacts.

Relevant notebooks include:

* `w01_research_question.ipynb`
* `w02_ml_task_framing.ipynb`
* `w03_data_contract.ipynb`
* `w03_feature_leakage_check.ipynb`
* `w04_signal_audit.ipynb`
* `w04_baseline_score.ipynb`
* `w05_model.ipynb`
* `w06_validation_audit.ipynb`
* `w07_action_playbook.ipynb`
* `capstone.ipynb`

The analysis uses DuckDB and pandas to query the FlyRank internship warehouse. The feature window is March 2026 and the outcome window is April 2026.

The final model uses five features:

* `gsc_impressions`
* `gsc_clicks`
* `gsc_avg_position`
* `ga4_pageviews`
* `ga4_engaged_sessions`

The target is defined as April 2026 `gsc_clicks > 0`.

### Validation configuration

The modeling frame contains 331,436 observations. An 80/20 held-out split was created using `train_test_split` with:

* `test_size = 0.20`
* `random_state = 42`
* `stratify = y`

This produced:

* Training rows: 265,148
* Validation rows: 66,288
* Training positive rate: 18.74%
* Validation positive rate: 18.74%

The split configuration and target definition are recorded in `w05_model.ipynb`.

### Random Forest configuration

The final Random Forest was configured with:

* `n_estimators = 200`
* `max_depth = 10`
* `min_samples_leaf = 20`
* `class_weight = "balanced"`
* `random_state = 42`
* `n_jobs = -1`

The model was trained only on the March feature window and evaluated against the April outcome.

The Week-4 baseline was evaluated on the same validation observations, allowing the model and baseline to be compared using the same target and metrics.

### Environment

The analysis environment uses Python with the main analysis libraries including:

* pandas
* NumPy
* DuckDB
* scikit-learn
* Matplotlib

The notebooks contain the executable analysis steps needed to reproduce the reported results. The Hugging Face warehouse is accessed using the `HF_TOKEN` secret supplied through the Colab environment; the token itself is not stored in the repository.

The reported validation metrics are:

| Method               | ROC-AUC | PR-AUC | Accuracy |
| -------------------- | ------: | -----: | -------: |
| Week-4 Rule Baseline |  0.8447 | 0.4858 |   0.8325 |
| Random Forest        |  0.9312 | 0.8397 |   0.8762 |

These values are produced by the executed modeling notebook.

The action-playbook notebook regenerates the ranked queue and associated paper artifacts rather than requiring the generated queue data to be committed to the repository.


## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. Data source: FlyRank.

The analysis is intended for educational and research purposes, with results presented using public-safe, measured, and decision-support language.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support language throughout. No causal claims are made. The analysis does not claim to predict Google's algorithm. No client-identifying details are included.
>
> **Metrics vs. base rate:** the modeling task has an 18.74% positive rate and an 81.26% negative rate. Accuracy is therefore reported alongside ROC-AUC and PR-AUC rather than used as the sole measure of model quality.
>
> **Final evidence standard:** reported numbers should match the fresh executed notebooks before deployment.

