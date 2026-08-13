# Capstone Report — <your lane>

- **Author:**
- **Lane:**
- **Repo:**
- **Date:**

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

This study asks whether search-performance and query-mix signals can identify content items whose search visibility is declining and help prioritize them for review. The analysis uses the FlyRank ML Internship Warehouse, including daily content-performance and query-level data, with historical 30-day windows used to construct pre-decision features. A Random Forest classifier was trained using eight pre-decision features and evaluated using a client-grouped split so that 12 held-out clients were not seen during training. On this grouped test set, the model achieved 83.82% accuracy and 0.839 weighted F1, compared with 60.85% accuracy and 0.616 weighted F1 for the transparent Week 4 baseline on the same test rows. The resulting model is intended as a ranked decision-support tool for prioritizing content review, rather than as a causal explanation of search-engine behavior.

## 1. Problem framing

This project supports the decision of which content items should be reviewed when search visibility is declining. The unit of analysis is a content item for a client, using aggregated search-performance signals from defined historical windows. The output is a ranked prediction of whether a content item is declining, which can support editorial review and prioritization. A wrong call can waste editorial effort on content that does not need attention or cause a genuinely declining item to be overlooked. Machine learning is useful because multiple historical search and query-mix signals can be combined into one repeatable decision-support model rather than relying only on a simple hand-written score.

The target is `is_declining`, defined as an item whose last-30-day impressions are less than 80% of its previous-30-day impressions. The model is intended for decision support, not for claiming causal effects or predicting Google's ranking algorithm.


## 2. Data safety

This project uses the FlyRank ML Internship Warehouse release hosted on Hugging Face. The analysis uses daily content-performance data and query-level aggregates. The main modeling frame is built from historical 30-day and previous-30-day windows so that the features are available before the decline outcome being evaluated.

The model uses the following pre-decision features: `imp_prev30`, `clk_last30`, `pos_last30`, `visible_queries`, `rare_share`, `anon_share`, `top_query_share`, and `ctr_last30`. These features describe historical impressions, clicks, position, query coverage, and query-mix characteristics.

The target `is_declining` is calculated from the relationship between `imp_last30` and `imp_prev30`. Because the target uses the last-30-day outcome, `imp_last30` is not used as a model feature. Similarly, `imp_change_pct` and other fields derived from the outcome window are excluded from the model to avoid label leakage.

Pseudonymous identifiers such as `client_hash_id` and `content_hash_id` are not used as predictive features. `client_hash_id` is used only to create the client-grouped validation split, allowing the final evaluation to test clients that were not present during training. Client names, domains, URLs, private queries, credentials, and other identifying information are not included in the public analysis.

The main leakage risk is allowing information from the outcome window or target definition to enter the feature set. This was addressed by using historical features and excluding label-derived fields. The final validation uses a client-grouped split with 35 training clients and 12 held-out test clients.


## 3. Baseline

The baseline is the transparent rule developed during Week 4. It combines average search position and click-through rate (CTR) into a single score:

`baseline_score = (pos_last30 / 10) + ((1 - ctr_last30) * 5)`

Higher scores indicate a combination of poorer average position and lower CTR and therefore higher review priority. The baseline assigns a simple review action to the highest-scoring content items and provides an interpretable benchmark for the machine-learning model.

For the final evaluation, the baseline was applied to exactly the same client-grouped test set used to evaluate the Random Forest. The baseline threshold on this test set was 6.2242. It achieved 60.85% accuracy, 0.617 macro F1, and 0.616 weighted F1.

The baseline is a fair comparison because it is simple, transparent, deterministic, and does not use the machine-learning model's predictions. Comparing both methods on the same held-out clients makes the difference in performance easier to interpret.

The baseline is not intended to represent an optimal editorial strategy. Its purpose is to establish how much useful signal can be obtained from a straightforward hand-written rule before adding a machine-learning model.


## 4. Model / analysis

The final model is a Random Forest classifier designed to identify content items whose search impressions are declining. Random Forest was selected because it can combine multiple numeric search and query-mix signals, capture non-linear relationships, and provide feature-importance information for interpretation.

The model uses eight pre-decision features:

- `imp_prev30` — impressions during the previous 30-day window
- `clk_last30` — clicks during the last 30-day window
- `pos_last30` — average search position during the last 30-day window
- `visible_queries` — number of visible queries associated with the content item
- `rare_share` — share of impressions associated with rare queries
- `anon_share` — share associated with anonymized queries
- `top_query_share` — share of retained query impressions coming from the top query
- `ctr_last30` — click-through rate during the last-30-day window

The target is `is_declining`, defined as:

`is_declining = 1 when imp_last30 < 0.8 × imp_prev30`

Otherwise, the target is 0.

The Random Forest configuration uses 300 trees, a maximum depth of 15, a minimum of 5 samples per leaf, balanced class weights, and random seed 42. Client and content identifiers are excluded from the feature matrix. `client_hash_id` is used only for grouped validation.

The model intentionally excludes `imp_last30` from the feature set because it directly contributes to the target definition. Other outcome-derived fields, such as impression-change percentages or trend labels, are also excluded to reduce leakage risk.

The model first achieved 86.51% accuracy on a conventional stratified random split. A stricter client-grouped evaluation was then performed to test whether the observed performance remained when the model was evaluated on clients that were not present during training.

## 5. Evaluation

The final evaluation uses a client-grouped train/test split rather than relying only on a random row split. `GroupShuffleSplit` with `random_state=42` was used to separate clients before training. The training set contained 71,805 rows from 35 clients, while the test set contained 29,815 rows from 12 held-out clients. The client identifiers were used only to create the split and were never provided to the model as predictive features.

The Random Forest achieved 83.82% accuracy, 0.826 macro F1, and 0.839 weighted F1 on the held-out clients. The transparent Week 4 baseline achieved 60.85% accuracy, 0.617 macro F1, and 0.616 weighted F1 on exactly the same test rows.

| Method | Accuracy | Macro F1 | Weighted F1 |
|---|---:|---:|---:|
| Week 4 Baseline | 60.85% | 0.617 | 0.616 |
| Random Forest | 83.82% | 0.826 | 0.839 |

The Random Forest therefore improved accuracy by approximately 23 percentage points over the baseline on the same held-out clients. Its weighted F1 was also higher, indicating better overall classification performance across the two classes.

The confusion matrix for the Random Forest was:

| | Predicted 0 | Predicted 1 |
|---|---:|---:|
| Actual 0 | 8,602 | 2,247 |
| Actual 1 | 2,576 | 16,390 |

The model correctly identified 8,602 non-declining items and 16,390 declining items in the grouped test set. It incorrectly classified 2,247 non-declining items as declining and 2,576 declining items as non-declining.

The conventional random split produced a higher accuracy of 86.51%, but the client-grouped result of 83.82% is treated as the primary validation result because it evaluates the model on clients that were not used during training. This provides a more demanding test of cross-client generalization within the available dataset.

These results show measured improvement over the transparent baseline, but they should be interpreted as evidence from this dataset and validation design rather than as proof that the model will perform identically on future data.
## 6. Interpretation

The Random Forest indicates that the model can distinguish declining from non-declining content using a combination of historical search-performance and query-mix signals. The model should be interpreted as identifying patterns associated with the observed decline label, not as establishing causal relationships.

The most important signals are interpreted as follows:

- `imp_prev30` represents the amount of search visibility observed in the previous 30-day window. It provides the model with information about the historical scale of a content item's search performance.
- `clk_last30` and `ctr_last30` provide information about recent click response and how effectively impressions translate into clicks.
- `pos_last30` describes recent average search position and provides a ranking-related signal.
- `visible_queries` captures how broadly a content item appears across visible queries.
- `rare_share`, `anon_share`, and `top_query_share` describe the distribution and concentration of query-level impressions.

The feature-importance analysis provides a useful descriptive view of which signals the Random Forest relied on most strongly. However, feature importance does not mean that a feature causes search visibility to decline. Correlated features can share predictive information, and the importance values should therefore be treated as directional evidence rather than causal explanations.

One important observation is that the model's performance decreased from 86.51% on a conventional random split to 83.82% on the stricter client-grouped split. This decrease is expected because the grouped evaluation requires the model to generalize to clients that were not represented in training. The relatively strong grouped result suggests that the learned patterns are not limited entirely to individual rows from the same clients.

The model also produced both false positives and false negatives. A false positive may cause an editor to review content that is not actually declining, while a false negative may cause genuinely declining content to be missed. These errors reinforce that the model should be used as a prioritization and decision-support tool rather than an automatic content decision maker.

Overall, the interpretation is directional: historical search and query-mix signals contain useful measured information for identifying the defined decline outcome in this dataset, but the analysis does not establish why the decline occurred or whether the same relationships will hold in other datasets or future periods.
## 7. Recommendation

The model output should be used as a ranked decision-support queue for content review. A FlyRank editor could begin with the highest-priority content items identified by the model and inspect their historical search performance and query-mix signals before deciding whether further content work is justified.

The recommended action playbook is:

1. **Prioritize declining content** — review items with a high predicted probability of the `is_declining` outcome.
2. **Inspect historical performance** — compare previous-30-day impressions, recent clicks, CTR, and average position before taking action.
3. **Inspect query coverage** — use visible query count and query-concentration signals to understand whether the decline appears broad or concentrated.
4. **Review before changing content** — the model should identify candidates for human investigation rather than automatically triggering a rewrite, merge, or removal.
5. **Monitor after an intervention** — if an editor changes content, future performance should be evaluated separately rather than treating the model prediction as proof that an intervention caused an improvement.

The ranked recommendations should therefore be interpreted as an editorial prioritization queue. Higher-ranked items receive attention first, while lower-ranked items can be monitored or reviewed later depending on available editorial resources.

Confidence in the model is supported by its measured improvement over the transparent baseline on the same client-grouped test set: 83.82% accuracy and 0.839 weighted F1 for the Random Forest compared with 60.85% accuracy and 0.616 weighted F1 for the baseline.

However, the recommendations have important limits. The model identifies an observed statistical pattern associated with the defined 30-day decline outcome. It does not establish the cause of a decline, guarantee that an editorial change will improve performance, or predict Google's ranking decisions. Human review remains necessary before taking action.
## 8. Reproducibility
## 8. Reproducibility

The analysis is implemented in the capstone notebook under `work/notebooks/capstone.ipynb`. The notebook connects to the FlyRank ML Internship Warehouse through DuckDB and reads the hosted Parquet data without downloading the complete warehouse into memory.

The main workflow is:

1. Connect DuckDB to the hosted warehouse using an authenticated Hugging Face read token.
2. Aggregate historical daily search-performance data into content-level features.
3. Join the content-level features with query-level signals.
4. Define the `is_declining` target using the 30-day impression comparison.
5. Remove outcome-derived and identifier fields from the predictive feature set.
6. Train the Random Forest using `random_state=42`.
7. Evaluate the model using a client-grouped train/test split.
8. Evaluate the Week 4 baseline on the same held-out clients.
9. Generate feature-importance and model-versus-baseline charts.
10. Produce ranked outputs for content-review recommendations.

The main Random Forest settings are 300 estimators, maximum depth 15, minimum samples per leaf 5, balanced class weights, and random seed 42. The client-grouped validation also uses random seed 42.

The evaluation is reproducible from the notebook rather than relying on an undocumented manual result. The notebook contains the cells used to construct the feature frame, train the model, perform the grouped evaluation, calculate the metrics, and generate the figures.

Credentials are not stored in the notebook or repository. A Hugging Face read token is supplied through the runtime environment or a secure notebook secret when the data connection is required.

The repository contains the weekly assignment notebooks and the capstone notebook under `work/notebooks/`. Generated data outputs are not treated as public source data, and no client-identifying information, private queries, credentials, or domains are included in the public report.

## 9. Acknowledgments & data credit

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

This project was completed as part of the FlyRank ML Internship and uses the public-safe, pseudonymized Search Intelligence warehouse provided for the internship. The analysis and conclusions in this paper are my own and should be interpreted as decision-support research rather than as claims about Google's ranking algorithm or causal search-engine behavior.
---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
