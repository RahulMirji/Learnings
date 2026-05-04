
***

# ML Systems & MLOps – Deep-Dive Guide

This guide focuses on what interviewers really test: understanding *why* things are done, not just *how*.

***

## 1. Data Splitting & Leakage Prevention

### 1.1 What is data leakage?

Data leakage happens when information that would not be available at prediction time “sneaks” into training. This leads to unrealistically high validation/test metrics and models that fail in production. [scikit-learn](https://scikit-learn.org/stable/common_pitfalls.html)

Two common root causes:

- Temporal leakage: Model sees future information when predicting the past.
- Relationship leakage: Correlated entities (same user, device, household, session) are split across train and test.

Interview mental model:  
“Is there *any* way the model, during training, can indirectly access test-time information?”

### 1.2 Time-based splitting

Use when there is any ordering or time dependency: logs, transactions, clickstreams, credit risk, time-series features.

- Sort by timestamp.
- Choose a cut-off date.
- Train on data strictly *before* the cut-off, validate in a later window, test in an even later window. [highdigital.co](https://www.highdigital.co.uk/blog/data-leakage-train-test-split-guide/)

Example:

- Train: Jan–Jun  
- Val: Jul  
- Test: Aug–Sep  

Key rules:

- Do not shuffle across time boundaries.
- Feature engineering must respect time: no aggregations that “peek” into the future (e.g., computing a user’s lifetime average using future events).

Interview example of leakage:

- You compute a “user_90d_spend” feature for all rows using full data (Jan–Dec) and then split by time. The January rows now use information from March–December, which is impossible in production.

### 1.3 Group-based splitting

Use when samples are not independent and identically distributed (IID) because they share an entity:

- Multiple rows per user_id, device_id, session_id, household_id, or patient_id.
- Many ML tasks: personalization, recommendation, fraud, ad CTR.

Group-based split idea:

- Group all rows by an identifier.
- Ensure each group appears in *one* of {train, val, test}.
- In sklearn: `GroupKFold`, `GroupShuffleSplit` with `groups=user_id`. [scikit-learn](https://scikit-learn.org/stable/common_pitfalls.html)

If you ignore this:

- The model can “memorize” user-specific behavior from train rows and look great on test rows from the same users.
- Offline metrics collapse once you face unseen users or devices in production.

### 1.4 Combined: Time + group

Real systems often require both:

- You want to test generalization to *future* behavior.
- You also want to avoid user overlap.

Typical strategy:

1. Filter to a time window (e.g., data up to Dec 2025).
2. Hold out the last month as test.
3. Within train+val, use `GroupKFold` on user_id.

Interview example:

- Fraud model for card transactions:
  - Train: Jan–Sep (user-based grouped CV).
  - Test: Oct (users not necessarily disjoint, but time strictly after training).
  - If you really want strict user isolation, you can also hold out some users entirely for test.

### 1.5 Key anti-patterns to mention

- Doing any normalization/encoding/feature selection on full data before splitting. [knime](https://www.knime.com/blog/3-must-avoid-pitfalls-splitting-datasets-train-test-data)
- Using full-history aggregates for time-series features.
- Using derived columns that encode label information (e.g., “was_defaulted” used indirectly).

**Interview phrase:**  
“First, I define my splits in a leakage-safe way (time-based and/or grouped). Then I construct all preprocessing steps *inside* the training fold only, usually via a pipeline.”

***

## 2. Drift Detection & Monitoring

### 2.1 Types of drift

Drift is change over time in the data or the relationship between inputs and outputs. [arize](https://arize.com/blog-course/population-stability-index-psi/)

- Data drift (covariate shift): Input distribution \(P(X)\) changes.  
  Example: New country added, distribution of user_age shifts older.
- Concept drift: Relationship between X and y (conditional \(P(y|X)\)) changes.  
  Example: Fraudsters change behavior; same features now map to different fraud probability.
- Prediction drift: Model output distribution \(P(\hat{y})\) changes.  
  Example: Suddenly model starts predicting far fewer positives.

Interview hook:  
Data drift is not always bad; it’s a *signal* that the environment changed and you should check performance.

### 2.2 PSI (Population Stability Index)

PSI compares the distribution of a variable between a baseline (e.g., training) and a current production window. [fiddler](https://www.fiddler.ai/blog/measuring-data-drift-population-stability-index)

High-level steps:

1. Choose bins (e.g., deciles or domain-specific buckets).
2. For each bin:
   - Compute baseline proportion.
   - Compute current proportion.
3. For each bin, compute contribution \((p_{\text{current}} - p_{\text{base}}) \cdot \ln(p_{\text{current}} / p_{\text{base}})\).
4. Sum across bins to get PSI.

Common interpretation rules of thumb: [scholarworks.wmich](https://scholarworks.wmich.edu/cgi/viewcontent.cgi?article=4249&context=dissertations)

- PSI < 0.1 → little/no shift.
- 0.1–0.2/0.25 → moderate shift; watch carefully.
- > 0.2/0.25 → significant shift; investigation and potential retraining recommended.

Typical uses:

- Feature-level PSI to identify which features are drifting.
- Score-level PSI to see if risk score distribution has shifted.

### 2.3 Prediction monitoring

Even without labels, you can monitor:

- Mean score.
- Distribution of scores (histogram, quantiles).
- Rate of positive predictions.

Example:

- Yesterday: 5% of events predicted “fraud” (positive).
- Today: 1% with similar traffic mix → suspicious drop; either data changed, the model is broken, or upstream pipeline changed.

This can trigger alerts or automatic checks before labels arrive. [arize](https://arize.com/blog-course/population-stability-index-psi/)

### 2.4 Business metrics

For every production ML system, business KPIs are the ultimate ground truth:

- Conversion rate, click-through rate, sign-ups.
- Fraud loss, chargebacks.
- Revenue per user, churn rate.

Even if technical metrics are stable, a change in business metrics might indicate:

- Concept drift, hidden data issues, or unmodeled factors.
- A misaligned threshold or bad rollout strategy.

Interview pro-tip:  
Always mention *aligning model monitoring with business KPIs*.

### 2.5 Retrain vs rollback

A practical decision table:

- Data pipeline broken, features missing, or obviously wrong values (e.g., all zeros):  
  → Rollback or disable the model; fix the pipeline first.
- Gradual, steady performance degradation as environment shifts:  
  → Retrain on recent data; maybe shorten training window or add time-aware features.
- Sudden big drop in business KPI right after a new model/threshold release:  
  → Rollback to previous stable version; treat it as a failed experiment.

Interview pattern:  
“Rollback when stability and safety are more important than fast adaptation; retrain when the environment genuinely changed and the old model is obsolete.”

***

## 3. NumPy – Views vs Copies

### 3.1 Why this matters

In ML preprocessing code, bugs often come from unintentionally modifying arrays through views, or copying when you thought you had a view (memory/performance). [scikit-learn](https://scikit-learn.org/stable/common_pitfalls.html)

### 3.2 Assignment: `b = a`

- `b` references the same underlying array object.
- `b is a` evaluates to `True`.
- Modifying `b` directly modifies `a`.

Use case: Cheap alias when you *intend* to mutate in place.

### 3.3 View: `b = a[:]` or slicing

- New Python object, same underlying data buffer.
- `b is a` is `False`, but `b.base` often points to `a`.
- Changing a slice modifies the original array and vice versa.

This includes many slicing operations like `a[:, 0]`, `a[::2]`.

### 3.4 Copy: `b = a.copy()`

- Independent data buffer.
- Changes to `b` do not affect `a`.

Use when:

- You want to protect original data (e.g., original labels or raw features).
- You are about to modify an array and need isolation.

Interview-style explanation:

- Views are efficient but dangerous if you don’t realize they alias the original.
- Copies are safer but cost memory and compute.
- In data pipelines, be explicit when you need copies to avoid weird bugs.

***

## 4. Pandas – Merge vs Join

### 4.1 Conceptual mapping to SQL

- `pd.merge` is SQL-like join with explicit keys.
- `DataFrame.join` is index-based, more “relational” in feel but less explicit.

Both can do inner, left, right, outer joins under the hood. [scikit-learn](https://scikit-learn.org/stable/common_pitfalls.html)

### 4.2 `merge`

Typical usage:

```python
pd.merge(df1, df2, on="user_id", how="inner")
```

Characteristics:

- Explicit `on`, `left_on`, `right_on`.
- Flexible for joining non-index columns.
- Good default for most ML data engineering tasks.

### 4.3 `join`

Typical usage:

```python
df1 = df1.set_index("user_id")
df2 = df2.set_index("user_id")
df1.join(df2, how="left")
```

Characteristics:

- Joins on index by default.
- Convenient when you already model your tables with meaningful indices.

### 4.4 Join types: inner vs left

- Inner join: Only rows with matching keys in both tables.
- Left join: All rows from left, matching rows from right when available.

Why this matters in ML:

- Left joins are often used to “enrich” the main fact table (events, users) with features from other tables.
- Inner joins can unintentionally drop rows, leading to biased training data.

### 4.5 Many-to-many explosion pitfall

Example:

- `df_users` has 1 row per user.
- `df_transactions` has multiple rows per user.
- If both sides have multiple rows per key, you get Cartesian product for those keys → row explosion.

Impact:

- Duplicate labels.
- Inflated sample size.
- Distorted feature distribution.

Interview line:  
“Whenever I join, I check the cardinality of keys before and after joining to avoid many-to-many explosions.”

***

## 5. Pipelines & Leakage Prevention

### 5.1 Why pipelines matter

In serious ML systems, the same transformations must be applied:

- During training (fitted on training data only).
- During validation and testing.
- In production, online.

Pipelines provide:

- Consistency of preprocessing.
- Protection against leakage.
- Easier deployment. [machinelearningmastery](https://www.machinelearningmastery.com/data-preparation-without-data-leakage/)

### 5.2 Wrong pattern

```python
scaler = StandardScaler()
scaler.fit(X)        # fitting on full dataset (train + test)
X_scaled = scaler.transform(X)
```

Problem:

- The scaler uses information from data that should be held out (test).
- This contaminates evaluation and inflates metrics. [knime](https://www.knime.com/blog/3-must-avoid-pitfalls-splitting-datasets-train-test-data)

### 5.3 Correct pattern

1. Split data into `X_train`, `X_val`, `X_test` using leakage-safe strategies.
2. Build a pipeline:

```python
pipeline = Pipeline([
    ("preprocess", ColumnTransformer(...)),
    ("model", LogisticRegression())
])
pipeline.fit(X_train, y_train)
y_pred = pipeline.predict(X_test)
```

Benefits:

- All transforms in `preprocess` are fitted only on training data.
- Test (and val) are transformed using parameters learned from training.

### 5.4 Cross-validation safety

With CV, the rule is:

- For each fold:
  - Fit preprocessing (e.g., scaling, encoding, feature selection) *only* on the training fold.
  - Apply to validation fold.

Using `Pipeline` + sklearn CV classes ensures this automatically. [machinelearningmastery](https://www.machinelearningmastery.com/data-preparation-without-data-leakage/)

Interview soundbite:  
“Every operation that learns from data—scaling, encoding, feature selection—must be inside the CV pipeline, not done before the split.”

***

## 6. ETL & Data Engineering for ML

### 6.1 Schema drift

Schema drift is when the structure of incoming data changes:

- New columns appear.
- Existing columns disappear.
- Types change (int → string, numeric → categorical).

Detection:

- Compare `set(incoming.columns)` with `set(expected.columns)`.
- Check types per column.

Typical handling strategy:

- New column: ignore initially, log an alert for investigation.
- Missing column: fill with default values or model-safe sentinel; decide if model can run safely.
- Type change: attempt safe cast; if impossible, fall back to defaults and raise alerts.

Interview:  
“Schema drift handling is both a data quality and operational safety problem; you want to fail loud or degrade gracefully, but never silently produce nonsense predictions.”

### 6.2 Incremental load strategies

For ML feature stores and training data:

- Full reload:
  - Recompute everything from scratch.
  - Simple but expensive; suitable for small datasets.
- Watermark-based:
  - Keep a “max timestamp” processed.
  - Next run reads rows with `timestamp > watermark`.
  - Good for medium-scale batch systems.
- CDC (Change Data Capture):
  - Capture inserts/updates/deletes from source systems.
  - Suitable for large-scale near-real-time ingestion.

These choices affect:

- Freshness of features.
- Cost and complexity of pipelines.
- Latency for model updates.

### 6.3 Idempotency

An ETL job is idempotent if running it multiple times for a given time window gives the same final state.

Why important:

- Retries after failures.
- Backfilling historical data.
- Avoiding double-counting features.

Patterns:

1. Merge/upsert (e.g., `MERGE INTO`):
   - If `id` exists, update.
   - Otherwise, insert.
2. Partition overwrite:
   - Delete all data for a given partition (e.g., date).
   - Insert exactly one canonical version.

Interview phrase:  
“In ML data pipelines, idempotency prevents silent data duplication, which would corrupt training and monitoring metrics.”

***

## 7. Imbalanced ML Metrics

### 7.1 Why accuracy fails

If 99% of samples are negative:

- A model that always predicts negative (0) has 99% accuracy.
- But it is useless, because it never detects positive cases.

Hence, accuracy hides poor performance on minority class. [deepchecks](https://deepchecks.com/f1-score-accuracy-roc-auc-and-pr-auc-metrics-for-models/)

### 7.2 Precision, recall, F1

- Precision: among predicted positives, how many are truly positive.  
  \( \text{Precision} = \frac{\text{TP}}{\text{TP} + \text{FP}} \)
- Recall (sensitivity): among actual positives, how many are detected.  
  \( \text{Recall} = \frac{\text{TP}}{\text{TP} + \text{FN}} \)
- F1: harmonic mean of precision and recall; punishes extreme imbalance between them. [deepchecks](https://deepchecks.com/f1-score-accuracy-roc-auc-and-pr-auc-metrics-for-models/)

Use cases:

- High recall for fraud detection, health alerts.
- High precision for human-review workflows (limited capacity).
- F1 when both matter and you need a single summary metric.

### 7.3 PR AUC vs ROC AUC

- ROC AUC:
  - Plots TPR vs FPR across thresholds.
  - More stable under moderate class imbalance.
  - Can still look “good” when model performs poorly on rare positives, because TN dominates. [blog.alliedoffsets](https://blog.alliedoffsets.com/boost-your-binary-classification-game-auc-roc-vs-auc-pr-which-one-should-you-use)
- PR AUC:
  - Plots precision vs recall for the positive class.
  - Directly measures performance on minority class.
  - Much more sensitive to performance on rare positives; better for extreme imbalance. [manishmazumder5.substack](https://manishmazumder5.substack.com/p/beyond-accuracy-a-deep-dive-into)

Rule of thumb:

- For highly imbalanced tasks (e.g., 1% positives), PR AUC is often more informative.
- In interviews, mention both and explain why PR AUC is often preferred in these settings.

### 7.4 Threshold selection

Rather than blind 0.5:

1. Cost-based:
   - Define cost of FP and FN.
   - Compute expected cost across thresholds.
   - Choose threshold minimizing expected cost.

2. Precision-constrained:
   - If manual review capacity is limited, require precision ≥ X%.
   - Pick the lowest threshold that satisfies that constraint to maximize recall subject to precision constraint.

3. Capacity-based:
   - You can only handle N cases/day.
   - Choose threshold that yields approximately N positives/day based on validation distribution.

Interview example:

- “For a fraud model with manual review team capacity of 1,000 cases/day, I would choose the threshold that yields ~1,000 predicted positives/day on validation, while ensuring acceptable precision.”

***

## 8. Monitoring Without Labels

### 8.1 The problem

Labels often arrive with delay:

- Credit default known months later.
- Fraud confirmed after investigations.
- Churn only known after a period.

So you cannot compute real-time accuracy, recall, F1, etc. [arize](https://arize.com/blog-course/population-stability-index-psi/)

### 8.2 Delayed metrics

Strategy:

- For each prediction, store:
  - Features (or feature fingerprints).
  - Model version and timestamp.
  - Prediction and score.
- When labels arrive later, join them back:
  - Compute performance metrics for each window and model version.
  - Track trends over time.

This enables:

- Backward-looking performance dashboards.
- Model comparison and offline A/B validation.

### 8.3 Proxy signals

While waiting for labels, track:

- Data drift (PSI, KL-like ideas).
- Prediction drift (score distribution).
- Business proxies (e.g., alert volume, manual-review acceptance rate).

If any of these shift strongly, trigger alerts or decrease trust in the model until full evaluation.

Interview:  
“Without labels, I rely on drift and business proxies as early warning systems, then confirm via delayed label-based evaluation.”

***

## 9. Model Registry

### 9.1 Purpose

A model registry is a central system to:

- Track versions of models.
- Associate each model with metadata.
- Control promotion stages: `staging`, `production`, `archived`.

It is crucial for:

- Reproducibility.
- Compliance and auditability.
- Safe rollbacks.

### 9.2 What to store

For each model version:

- Model artifact (weights, serialized pipeline).
- Preprocessing pipeline definition and its version.
- Training data snapshot or version ID.
- Metrics (ROC AUC, PR AUC, precision/recall at chosen threshold).
- Operational configs: threshold, features used, environment dependencies.

Example minimal metadata (like your JSON):

```json
{
  "model_version": 42,
  "precision": 0.91,
  "threshold": 0.8,
  "data_version": "2025Q1"
}
```

Interview lines:

- “For any prediction, I want to be able to answer: which model version produced it, with what code and data?”
- “The registry enables easy rollback to a previous good version with known performance.”

***

## 10. Docker Versioning

### 10.1 Why tags matter

Docker images encapsulate:

- Code.
- Dependencies.
- Sometimes model artifacts.

For reproducibility and rollback, tags must be meaningful and immutable references.

### 10.2 Good vs bad tags

- Good: `model-service:v42-git-a1b2c3`
  - Encodes model version and git commit.
  - Can be traced back to exact source code.
- Bad: `latest`
  - Overwritten frequently.
  - No guarantee what code or model it contains.

Interview-ready explanation:

- “I never deploy `:latest` to production. I always use immutable tags tied to git commit and model version so rollbacks are deterministic.”

***

## 11. Kubernetes Deployment

### 11.1 Rolling updates

Rolling update strategy:

- Gradually replace old pods with new ones.
- `maxUnavailable: 0`: ensure no capacity reduction during rollout.
- `maxSurge: 1`: allow at most one extra pod above desired count during rollout.

This yields zero-downtime deployments *if* probes and readiness are correct.

### 11.2 Health checks

- Readiness probe:
  - Indicates when pod is ready to receive traffic (e.g., model loaded, warm caches prepared).
  - Until ready, pod is excluded from load balancing.
- Liveness probe:
  - Detects stuck or unhealthy pods (e.g., deadlock, memory leak).
  - Triggers automatic restart.

For ML services:

- Readiness: after model is loaded and a simple test prediction passes.
- Liveness: periodic health endpoint that checks core dependencies.

### 11.3 Rollback

Command example:

```bash
kubectl rollout undo deployment/model-service
```

Because of versioned images + registries, you can:

- Roll back to last known good image.
- Or a specific revision.

### 11.4 Advanced strategies

- Canary deployment:
  - Route small percentage of traffic (e.g., 5–10%) to new version.
  - Monitor metrics and business KPIs.
  - Gradually ramp up if healthy; rollback if not.

- Blue-green deployment:
  - Run two environments (blue & green).
  - Route traffic to one at a time.
  - Switch traffic instantly between them for fast rollback.

Interview phrase:

- “For high-risk model changes, I prefer canary: 5% → 25% → 50% → 100% with monitoring of key metrics.”

***

## 12. Overall Mental Model

High-level lifecycle:

1. **Data**: Ingest, clean, handle schema drift, store in a robust, idempotent way.
2. **Split**: Respect time and groups, avoid leakage.
3. **Train**: Use pipelines, repeated CV, proper metrics for imbalance.
4. **Evaluate**: Offline metrics + business metrics; pick thresholds via cost/capacity.
5. **Deploy**: Containerize, version images, deploy via Kubernetes with safe rollout.
6. **Monitor**: Drift, predictions, proxy metrics, delayed label metrics.
7. **Retrain/Rollback**: Retrain for gradual drift; rollback for acute failures.

You can use this flow as your “narrative” in interviews when describing any ML system.

***

