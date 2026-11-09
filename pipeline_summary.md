# Model Studio Pipeline Summary

This file describes the SAS Viya Model Studio project **Data Mining** (Adult Census Income dataset) in readable form. The project export (`Data_Mining.zip`) stores the pipelines in a compressed block that can't be read on GitHub, so the node settings below were extracted from it. The model results come from the team's evaluation of each model.

- **Tool:** SAS Viya Model Studio (data mining component, version V2026.03)
- **Dataset:** [UCI Adult Census Income](https://archive.ics.uci.edu/dataset/2/adult), loaded as a table named `ADULT`
- **Target:** `income` (binary: `<=50K` / `>50K`)
- **Pipelines:** 2 (`Pipeline 1` for supervised learning, `Clustering` for segmentation)
- **Random seed:** 12345 in every node that uses one

---

## Pipeline 1: Income classification (decision trees)

```
Data -> Replacement -> Imputation -> 16 decision trees -> Model Comparison
```

### Preprocessing

| Node | Setting | Value |
|---|---|---|
| Replacement | Interval outlier method | Standard deviation from the mean, cutoff 3 |
| Replacement | Outlier replacement value | Missing |
| Replacement | Unknown class levels | Replaced with Missing |
| Imputation | Class variables | Count (most frequent level) |
| Imputation | Interval variables | Median |
| Imputation | Missing-percentage cutoff | 50% |
| Imputation | Original variables | Rejected after imputation |
| Imputation | Random seed | 12345 |

The `native_country` variable was rejected by the Data node because it exceeds the maximum number of levels (42 levels).

### Decision tree experiments

All 16 trees share these settings: maximum depth 10, pruning by cost complexity, missing values used in the search, interval-target criterion Variance, seed 12345. They differ in three settings, which the names encode as `CRITERION_<leaf size code>_<max branches>`.

| Setting | Values tried |
|---|---|
| Split criterion (class target) | CHAID, Entropy, Gini, IGR (information gain ratio) |
| Minimum leaf size | 1,954 (code `6`, about 6% of the data) and 1,140 (code `35`, about 3.5%) |
| Maximum branches | 2 and 4 |

4 criteria x 2 leaf sizes x 2 branch limits = **16 trees**.

### Evaluation approach

Each tree was scored with a weighted combined performance score (CPS):

```
CPS = 0.60 x Accuracy + 0.05 x Simplicity + 0.20 x Lift + 0.15 x Stability
```

| Measure | Definition | Weight | Threshold |
|---|---|---|---|
| Accuracy | (TP + TN) / (TP + TN + FP + FN) | 0.60 | 0.70 |
| Simplicity | 1 / tree complexity, using the number of leaves | 0.05 | none |
| Lift | Response rate / baseline response rate, at the 2nd decile, target event income >50K | 0.20 | none |
| Stability | 1 if performance is stable across train, validation and test, otherwise 0 | 0.15 | 1 |

### Results (16 trees)

| Tree | Accuracy | Leaves | Lift (2nd decile) | Stable | Overall score |
|---|---|---|---|---|---|
| CHAID_35_4 | 0.76 | 2 | 1.26 | Yes | 0.8620 |
| CHAID_35_2 | 0.79 | 2 | 0.86 | No | 0.6488 |
| CHAID_6_4 | 0.76 | 2 | 1.26 | Yes | 0.8620 |
| CHAID_6_2 | 0.76 | 2 | 1.26 | Yes | 0.8620 |
| ENTROPY_35_4 | 0.82 | 9 | 1.14 | Yes | 0.9238 |
| ENTROPY_35_2 | 0.83 | 8 | 1.16 | Yes | 0.9338 |
| ENTROPY_6_4 | 0.82 | 7 | 1.17 | Yes | 0.9298 |
| **ENTROPY_6_2** | 0.82 | 4 | 1.38 | Yes | **0.9474** |
| GINI_35_4 | 0.82 | 9 | 1.14 | Yes | 0.9238 |
| GINI_35_2 | 0.83 | 8 | 1.16 | Yes | 0.9338 |
| GINI_6_4 | 0.82 | 7 | 1.17 | Yes | 0.9298 |
| GINI_6_2 | 0.82 | 4 | 1.38 | No | 0.7974 |
| IGR_35_4 | 0.83 | 6 | 1.19 | Yes | 0.9398 |
| IGR_35_2 | 0.83 | 6 | 1.19 | Yes | 0.9398 |
| IGR_6_4 | 0.82 | 4 | 1.38 | No | 0.7974 |
| IGR_6_2 | 0.82 | 4 | 1.38 | No | 0.7974 |

"Stable = No" means the lift curve changed behavior at the 3rd decile across the data partitions.

**Best tree among the 16: ENTROPY_6_2** (overall score 0.9474). It reaches about 0.82 accuracy with only 4 leaves. Its logic: people whose `relationship` is Husband or Wife are split further by `education`, and Masters, Prof-school and Doctorate holders are classified as `>50K`. Everyone else is classified as `<=50K`.

### Extra combination: ENTROPY_4_2

After the 16 trees above, a further combination was tried to look for a better result. It follows the same naming pattern (`ENTROPY_<leaf size code>_<max branches>`), with a leaf-size code of 4.

| Setting | Value |
|---|---|
| Split criterion (class target) | Entropy |
| Maximum branches | 2 |
| Minimum leaf size | about 1,302 (code `4`, 4% of the 32,561 rows) |
| Maximum depth | 10 |
| Pruning | Cost complexity |
| Missing values | Used in search |
| Random seed | 12345 |

The leaf size is calculated from the naming pattern, and the other settings are carried over from the 16 trees. This tree is not in the exported project file.

| Tree | Accuracy | Leaves | Lift (2nd decile) | Stable | Overall score |
|---|---|---|---|---|---|
| **ENTROPY_4_2** | 0.8246 | 4 | 1.38 | Yes | **0.9498** |

It scores slightly higher than ENTROPY_6_2 (0.9474) because of its higher accuracy at the same tree size. GINI_8_2 reached the same accuracy, leaf count and score.

---

## Pipeline 2: Clustering (k-means)

```
Data -> Replacement -> Imputation -> Km_2, Km_3, Km_4, Km_5, Manage Variables
Km_3 -> 4 decision trees (CHAID, Entropy, Gini, IGR) -> Model Comparison
```

### k-means settings (same in all four nodes except the number of clusters)

| Setting | Value |
|---|---|
| Number of clusters | User-specified: 2, 3, 4 or 5 (nodes Km_2 to Km_5) |
| Standardization | Standard deviation (STD) |
| Initialization | Forgy |
| Distance | Euclidean |
| Maximum iterations | 10 |
| Random seed | 12345 |

The ID variable and the target variable were excluded from clustering.

### Segmentations

| Segmentation | Clusters | Cluster sizes | Smallest cluster | Contains outlier cluster (<10%) |
|---|---|---|---|---|
| km_2 | 2 | 16,930 / 15,631 | 15,631 | No |
| km_3 | 3 | 12,015 / 7,122 / 13,424 | 7,122 | No |
| km_4 | 4 | 1,261 / 12,634 / 6,888 / 11,778 | 1,261 | Yes |
| km_5 | 5 | 1,252 / 3,666 / 6,128 / 13,129 / 8,386 | 1,252 | Yes |

The 10% outlier threshold is about 3,300 observations. km_4 and km_5 are the segmentations that contain a small (outlier) cluster, so they are the ones to review if outlier segments are of interest.

### Explaining the km_3 clusters with decision trees

For each cluster tree in the exported project, the settings are: maximum branches 2, maximum depth 10, minimum leaf size 1,575, pruning by cost complexity. The trees were scored with a weighted score of 0.60 accuracy and 0.40 simplicity, with no thresholds.

- **Best tree for the km_3 clusters:** IGR with overall score 0.89, using `relationship` as the main splitter and `education` as the second most important variable.
- **Cluster profiles:** Cluster_0 is the youngest group with the fewest hours worked and low capital gain (low activity). Cluster_1 is a middle group with moderate age and hours. Cluster_2 has the highest capital gain and the most hours per week (high activity, likely higher income).
- **Largest-cluster trees:** a separate set of trees on the biggest cluster found GINI as the best (overall score 0.88, accuracy 0.8237), slightly below the cluster-ID tree's accuracy of 0.8246.

---

## Reproducing the project

1. Download the UCI Adult dataset and load it into SAS Viya as a table named `ADULT`.
2. Import `Data_Mining.zip` into Model Studio (Projects page, Import).
3. If the data node shows a missing table, re-select your own `ADULT` table in the project's Data tab.
4. Run each pipeline.

The exported project was created by the project owner's account, so the import may only work from that same account. This summary is meant to document the work either way.
