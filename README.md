# Lab 10 — Model Validation & Optimization

> **Course:** CS-423 Data Warehousing and Data Mining
> **Tool:** RapidMiner Studio / AI Studio
> **Dataset:** `customer-churn-data.xlsx` (customer churn — predict `churn` vs `loyal`)

A step-by-step walkthrough of building, validating, comparing and tuning classification
models in RapidMiner, using customer churn as the case study.

## Contents

- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [Dataset](#dataset)
- [Task 1 — Testing a Model](#task-1--testing-a-model)
- [Task 2 — Validating a Model](#task-2--validating-a-model)
- [Task 3 — Finding the Right Model](#task-3--finding-the-right-model)
- [Task 4 — Model Optimization](#task-4--model-optimization)
- [Results Summary](#results-summary)
- [Key Takeaways](#key-takeaways)

## Objectives

By the end of this lab you should be able to:

1. Load a spreadsheet into RapidMiner and assign the **label** role to a target attribute.
2. Train a baseline classifier and read a **Performance** report.
3. Replace a naive train-on-everything setup with **cross-validation**.
4. Compare several algorithms at once with **Compare ROCs** and interpret confidence values.
5. Tune hyperparameters systematically with **Optimize Parameters (Grid)** and log the search.

## Prerequisites

- RapidMiner Studio (or AI Studio) — any recent version
- The `customer-churn-data.xlsx` dataset
- Basic familiarity with the RapidMiner process canvas and operator ports

## Dataset

`customer-churn-data.xlsx` contains 600 examples with 1 special attribute and 4 regular
attributes:

| Attribute | Type | Role |
|---|---|---|
| `Gender` | polynominal | regular |
| `Age` | integer | regular |
| `Payment Method` | polynominal | regular |
| `LastTransaction` | integer | regular |
| `Churn` | polynominal (`churn` / `loyal`) | **label** |

Some rows have a missing `Churn` value; these are filtered out before training.

![Source dataset file](images/01-source-dataset-file.png)

---

## Task 1 — Testing a Model

Build a baseline: train a Decision Tree on the whole dataset and score it on the same data.
This is deliberately naive — it establishes the reference point that Task 2 improves on.

### Step 1 — Import the data

Drag a **Read Excel** operator onto the process panel and point it at
`customer-churn-data.xlsx`. Step through the import wizard and select the cell range to import.

![Read Excel import wizard](images/02-read-excel-import-wizard.png)

### Step 2 — Define the target variable

In the *Format your columns* step of the wizard, find the `Churn` column and change its
role to **label**. This tells RapidMiner that `Churn` is the variable to predict.

![Change role to label](images/03-change-role-to-label.png)

![Format columns](images/04-format-columns.png)

### Step 3 — Clean the data

Add a **Filter Examples** operator and connect it to the output of **Read Excel**.

![Read Excel connected to Filter Examples](images/05-read-excel-filter-examples.png)

Configure the filter condition so that `Churn` **is not missing**. Rows without a known
outcome cannot be used for supervised learning.

![Filter condition: Churn is not missing](images/06-filter-condition-churn-not-missing.png)

### Step 4 — Train the model

Drag a **Decision Tree** operator onto the panel and connect the filtered output to its
training (`tra`) port.

![Decision Tree connected](images/07-decision-tree-connected.png)

### Steps 5 & 6 — Run and evaluate

Run the process, then add a **Performance** operator to evaluate the results.

![Decision tree model](images/08-decision-tree-model.png)

The prediction table shows the predicted label alongside per-class confidence values.

![Prediction and confidence table](images/09-prediction-confidence-table.png)

> **Why this isn't enough:** the model is scored on the same rows it learned from, so the
> reported performance is optimistic. Task 2 fixes this.

---

## Task 2 — Validating a Model

Replace the single train/score pass with k-fold cross-validation for a more honest estimate
of how the model generalises.

### Step 1 — Reset the process

Delete the operators that followed the initial **Filter Examples**.

![Process reset back to Filter Examples](images/10-process-reset-to-filter.png)

### Step 2 — Add Cross Validation

Add a **Cross Validation** operator to the workspace and connect the filtered data to it.

![Cross Validation operator added](images/11-cross-validation-added.png)

### Step 3 — Populate the sub-processes

Double-click into **Cross Validation** and fill both sides:

- **Training side:** `Decision Tree`
- **Testing side:** `Apply Model` → `Performance`

![Cross Validation sub-processes](images/12-cross-validation-subprocesses.png)

### Step 4 — Connect the output

Connect the performance (`per`) port to the process results output.

![Performance port connected](images/13-cross-validation-per-port.png)

### Step 5 — Run

Run the process and inspect accuracy and the other performance metrics.

![Cross-validation performance](images/14-cross-validation-performance.png)

**Result:** accuracy **82.11% ± 4.86%** (micro average 82.11%), with class precision of
86.90% for `loyal` and 74.03% for `churn`.

---

## Task 3 — Finding the Right Model

Rather than assuming a Decision Tree is the right choice, evaluate several classifiers side
by side and compare their ROC curves.

### Step 1 — Add a breakpoint

Add a breakpoint **after** the **Filter Examples** operator (`F7`). A breakpoint pauses
execution so you can inspect the intermediate ExampleSet, then resume or restart.

![Breakpoint after Filter Examples](images/16-breakpoint-after-filter.png)

The process halts and shows the filtered data:

![Paused ExampleSet at the breakpoint](images/15-breakpoint-paused-exampleset.png)

![ExampleSet result](images/17-exampleset-result.png)

### Step 2 — Remove the breakpoint

Once you've confirmed the data looks right, remove the breakpoint (*Remove all Breakpoints*)
so the process runs end to end.

![Remove all breakpoints menu](images/19-remove-all-breakpoints-menu.png)

![Breakpoint removed](images/18-breakpoint-removed.png)

### Step 3 — Swap in Compare ROCs

Replace the **Cross Validation** operator with the **Compare ROCs** operator.

![Compare ROCs operator](images/20-compare-rocs-operator.png)

### Step 4 — Add candidate models

Inside **Compare ROCs**, add the classifiers you want to compare. This lab uses:

| Model | Notes |
|---|---|
| Decision Tree | baseline from Task 1 |
| Naive Bayes | probabilistic, assumes feature independence |
| k-NN | instance-based |
| Rule Induction | rule-based, interpretable |
| Random Forest | ensemble of trees |

![Models inside Compare ROCs](images/21-models-inside-compare-rocs.png)

![Compare ROCs connections](images/22-compare-rocs-connections.png)

![Compare ROCs in the main process](images/23-compare-rocs-main-process.png)

Running the process overlays one ROC curve per model, with shaded confidence bands:

![ROC comparison](images/24-roc-comparison-curves.png)

![ROC curves in detail](images/27-roc-curves-detail.png)

### Step 5 — Run cross-validation alongside it

Use a **Multiply** operator to feed the same filtered data into both **Compare ROCs** and
**Cross Validation**, so you can analyse confidence values in parallel with the curves.

![Multiply feeding Compare ROCs and Cross Validation](images/25-multiply-rocs-and-cross-validation.png)

Confidence values establish the **threshold** at which a prediction is assigned to `churn`
or `loyal` — a higher confidence in one label means a higher likelihood of that outcome.

![Confidence and performance table](images/26-confidence-performance-table.png)

---

## Task 4 — Model Optimization

Fine-tune the Decision Tree's hyperparameters with a grid search, and log every trial.

### Step 1 — Add Optimize Parameters (Grid)

Drag the **Optimize Parameters (Grid)** operator onto the canvas.

![Optimize Parameters added](images/28-optimize-parameters-added.png)

### Step 2 — Disable Compare ROCs

Disable the **Compare ROCs** operator (and **Multiply**, if present) so only the optimization
branch executes.

![Compare ROCs disabled](images/29-compare-rocs-disabled.png)

### Steps 3 & 4 — Nest the validation inside the optimizer

Cut the **Cross Validation** operator from the main process and paste it **inside**
**Optimize Parameters**. This creates a nested loop: for every parameter combination, a full
cross-validation runs.

![Optimizer in the main process](images/30-optimizer-in-main-process.png)

![Cross Validation inside the optimizer](images/31-cross-validation-inside-optimizer.png)

### Step 5 — Choose the parameters to search

Open *Edit Parameter Settings* and select the Decision Tree's **criterion** and
**minimal gain**.

| Parameter | Range | Steps | Scale |
|---|---|---|---|
| `minimal gain` | 0.01 – 1 | 100 | linear |
| `criterion` | `gini_index`, `gain_ratio`, `information_gain`, `accuracy` | — | list |

That yields **505 combinations** across the two parameters.

![Grid range settings](images/32-select-parameters-grid-range.png)

### Step 6 — Log the search

Add a **Log** operator to record the internals of the optimization run. Configure it to store:

| Column | Source | Value |
|---|---|---|
| `Gain` | Decision Tree | parameter → `minimal gain` |
| `Criterion` | Decision Tree | parameter → `criterion` |
| `Iteration` | Cross Validation | value → `applycount` |
| `Performance` | Cross Validation | value → `performance 1` |

![Log operator parameter list](images/33-log-operator-parameter-list.png)

![Log operator connected](images/34-log-operator-connected.png)

### Step 7 — Run and analyse

Run the process and inspect the logged parameter sets against their performance.

![Optimization results](images/35-optimization-results-log.png)

Best parameter set found:

```
Decision Tree.criterion    = gini_index
Decision Tree.minimal_gain = 0.0298

accuracy  : 78.96% +/- 5.43%   (micro average: 78.96%)
precision : 74.48% +/- 7.54%   (positive class: churn)
recall    : 61.91% +/- 12.57%  (positive class: churn)
AUC       : 0.769 +/- 0.088    (positive class: churn)

ConfusionMatrix:
True:   loyal   churn
loyal:  513     123
churn:  67      200
```

Plotting logged **Performance vs. Gain**, coloured by criterion, shows performance collapsing
to a floor once `minimal gain` grows past roughly 0.1 — the tree gets pruned to nothing.
The useful region is the narrow band of small gain values on the left.

![Performance vs. minimal gain, by criterion](images/36-performance-vs-gain-by-criterion.png)

---

## Results Summary

| Stage | Setup | Accuracy |
|---|---|---|
| Task 1 | Decision Tree, no validation split | optimistic — trained and scored on the same data |
| Task 2 | Decision Tree + Cross Validation | **82.11% ± 4.86%** |
| Task 3 | 5 models compared via ROC/AUC | Random Forest and Rule Induction curves dominate; Naive Bayes trails |
| Task 4 | Grid-tuned Decision Tree (`gini_index`, gain 0.0298) | 78.96% ± 5.43%, AUC 0.769 |

> **Note:** the tuned score is *lower* than the Task 2 default — a useful reminder that a grid
> search optimises whatever objective and validation scheme you give it, and that a wide
> `minimal gain` range mostly explores parameter values that over-prune the tree.

## Key Takeaways

1. **Testing a model** — loading data, dropping rows with a missing target, training a Decision
   Tree and reading the Performance output shows how a classifier behaves *without* any
   train/test split. It is the wrong way to report accuracy, which is exactly the point.
2. **Validating a model** — Cross Validation partitions the data into folds, training on one
   subset and testing on the held-out one, giving an estimate that reflects generalisation.
3. **Choosing the right model** — Compare ROCs evaluates several algorithms in one run.
   Confidence values then set the decision threshold for assigning a case to `churn` or `loyal`.
4. **Tuning the model** — Optimize Parameters (Grid) searches hyperparameters systematically
   with nested cross-validation, and a Log operator makes the whole search inspectable rather
   than a black box.

---

*Educational lab material. The dataset is a teaching dataset and contains no real customer data.*
