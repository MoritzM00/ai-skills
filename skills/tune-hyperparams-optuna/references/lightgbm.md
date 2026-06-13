# LightGBM sklearn Tuning with Optuna

Read this when tuning `lightgbm.LGBMRegressor` or
`lightgbm.LGBMClassifier`. Use the sklearn estimator API by default. Do not
switch to `lightgbm.train()`, native `Dataset`, or `LightGBMTuner` unless the
user or project explicitly wants the native API.

## Source Anchors

- `LGBMRegressor` sklearn API: https://lightgbm.readthedocs.io/en/stable/pythonapi/lightgbm.LGBMRegressor.html
- `LGBMClassifier` sklearn API: https://lightgbm.readthedocs.io/en/stable/pythonapi/lightgbm.LGBMClassifier.html
- LightGBM parameters: https://lightgbm.readthedocs.io/en/stable/Parameters.html
- LightGBM parameter tuning guide: https://lightgbm.readthedocs.io/en/stable/Parameters-Tuning.html
- LightGBM early stopping callback: https://lightgbm.readthedocs.io/en/stable/pythonapi/lightgbm.early_stopping.html

## sklearn Parameter Names

Use sklearn constructor names in sklearn code:

| sklearn name | LightGBM core name | Role |
| --- | --- | --- |
| `num_leaves` | `num_leaves` | Leaf-wise complexity |
| `max_depth` | `max_depth` | Optional depth cap |
| `min_child_samples` | `min_data_in_leaf` | Minimum records in a leaf |
| `min_child_weight` | `min_sum_hessian_in_leaf` | Minimum Hessian in a leaf |
| `subsample` | `bagging_fraction` | Row sampling fraction |
| `subsample_freq` | `bagging_freq` | Enables row sampling |
| `colsample_bytree` | `feature_fraction` | Feature sampling per tree |
| `reg_alpha` | `lambda_l1` | L1 regularization |
| `reg_lambda` | `lambda_l2` | L2 regularization |
| `min_split_gain` | `min_gain_to_split` | Minimum split gain |

Avoid putting core aliases in `**kwargs` when an explicit sklearn constructor
argument exists. Pass core-only parameters such as `scale_pos_weight`,
`max_bin`, or `verbosity` deliberately, not as a dumping ground for copied
native-API configs.

## Starter Search Space

LightGBM grows leaf-wise, so `num_leaves`, `min_child_samples`, and `max_depth`
are more important than in depth-wise tree learners. If `max_depth` is positive,
keep `num_leaves <= 2 ** max_depth`.

```python
def suggest_lgbm_params(trial: optuna.Trial) -> dict[str, object]:
    max_depth = trial.suggest_int("max_depth", 3, 12)
    max_leaves = min(256, 2**max_depth)
    subsample = trial.suggest_float("subsample", 0.6, 1.0)

    return {
        "boosting_type": "gbdt",
        "n_estimators": 5000,  # high upper bound; early stopping chooses the length
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.2, log=True),
        "max_depth": max_depth,
        "num_leaves": trial.suggest_int("num_leaves", 8, max_leaves, log=True),
        "min_child_samples": trial.suggest_int("min_child_samples", 5, 300, log=True),
        "min_child_weight": trial.suggest_float("min_child_weight", 1e-3, 10.0, log=True),
        "subsample": subsample,
        "subsample_freq": 1 if subsample < 0.999 else 0,
        "colsample_bytree": trial.suggest_float("colsample_bytree", 0.5, 1.0),
        "reg_alpha": trial.suggest_float("reg_alpha", 1e-8, 100.0, log=True),
        "reg_lambda": trial.suggest_float("reg_lambda", 1e-3, 100.0, log=True),
        "min_split_gain": trial.suggest_float("min_split_gain", 0.0, 2.0),
    }
```

For very large datasets, raise the lower bound of `min_child_samples`. For small
datasets, lower `num_leaves` and narrow the regularization space before
increasing trial count.

Do not tune `n_estimators` directly in the first-pass search unless model size
or latency is part of the objective. Use a high upper bound and early stopping,
then materialize the final tree count only when the final refit must avoid an
internal validation set.

## Estimator Setup

Regression:

```python
import lightgbm as lgb

model = lgb.LGBMRegressor(
    **params,
    objective="regression",
    random_state=seed,
    n_jobs=lgbm_threads,
    verbosity=-1,
)
model.fit(
    X_train,
    y_train,
    eval_set=[(X_valid, y_valid)],
    eval_names=["valid"],
    eval_metric="rmse",
    callbacks=[lgb.early_stopping(100, first_metric_only=True, verbose=False)],
)
```

Binary classification:

```python
model = lgb.LGBMClassifier(
    **params,
    objective="binary",
    random_state=seed,
    n_jobs=lgbm_threads,
    verbosity=-1,
)
model.fit(
    X_train,
    y_train,
    eval_set=[(X_valid, y_valid)],
    eval_names=["valid"],
    eval_metric="binary_logloss",
    callbacks=[lgb.early_stopping(100, first_metric_only=True, verbose=False)],
)
```

For multiclass classification, use `objective="multiclass"` and
`eval_metric="multi_logloss"` as the first stable metric. Report task-specific
metrics after refit.

After fitting with early stopping, LightGBM exposes `best_iteration_`,
`n_estimators_`, and `n_iter_`. Prediction uses the best iteration by default
when it exists and `num_iteration=None`, so explicit extraction is mainly needed
when setting a fixed `n_estimators` for a later final refit.

## Pruning

Use pruning mainly for large, expensive single train/validation split studies.
For small CV studies, skip pruning and rely on early stopping. If pruning is
justified, use `LightGBMPruningCallback` from `optuna-integration` with the
metric and validation name passed to `fit`.

```python
from optuna_integration import LightGBMPruningCallback

callbacks = [
    lgb.early_stopping(100, first_metric_only=True, verbose=False),
    LightGBMPruningCallback(trial, "binary_logloss", valid_name="valid"),
]
model.fit(
    X_train,
    y_train,
    eval_set=[(X_valid, y_valid)],
    eval_names=["valid"],
    eval_metric="binary_logloss",
    callbacks=callbacks,
)
```

Use `"rmse"` or `"l2"` for regression, `"binary_logloss"` or `"auc"` for binary
classification, and `"multi_logloss"` for multiclass. The metric string must
match LightGBM's reported metric.

## Parameter Effects

- `learning_rate`: smaller values often improve generalization but need more
  trees. Pair with early stopping and a high `n_estimators`.
- `num_leaves`: main complexity control. Larger values can improve accuracy but
  overfit quickly because LightGBM grows trees leaf-wise.
- `max_depth`: caps tree depth. If positive, constrain `num_leaves` under the
  depth-derived limit.
- `min_child_samples`: very important anti-overfit control. Increase it when
  leaves become too small or validation variance is high.
- `min_child_weight`: Hessian-based leaf constraint. Tune on a log scale.
- `subsample` plus `subsample_freq`: row bagging. `subsample` has no effect
  unless `subsample_freq` is nonzero.
- `colsample_bytree`: feature sampling per tree. Useful for speed and overfit
  control.
- `reg_alpha`, `reg_lambda`: L1 and L2 regularization.
- `min_split_gain`: suppresses weak splits.
- `max_bin`: larger can improve accuracy but slows training; smaller can speed
  training and reduce overfit.
- `extra_trees`: can reduce overfit and speed split search, but changes model
  behavior enough to deserve a separate conditional branch.

## Classification and Imbalance

For binary imbalance, choose one strategy:

- Use `class_weight="balanced"` for a quick sklearn-compatible baseline.
- Use `scale_pos_weight` when optimizing ranking metrics such as AUC or average
  precision.
- Use explicit `sample_weight` when weights come from business costs or sampling
  design.

Do not combine multiple imbalance mechanisms casually. Weighted training can
hurt raw probability calibration; calibrate probabilities after model selection
when calibrated probabilities matter.

## Search Expansion

Add these only after the starter study:

- `boosting_type="dart"` with DART-specific conditional parameters when slower,
  noisier training is acceptable.
- `max_bin` for speed/accuracy tradeoffs.
- `extra_trees=True` as a conditional branch for overfit-prone data.
- `cat_smooth`, `cat_l2`, and categorical controls only when using native
  categorical features.
- Monotone constraints only from domain knowledge, not as ordinary search
  parameters.

## Common Mistakes

- Setting `subsample < 1.0` but leaving `subsample_freq=0`.
- Tuning `n_estimators` directly while also relying on early stopping.
- Pruning a whole CV trial from one noisy fold's boosting curve.
- Letting `num_leaves` exceed the practical limit implied by `max_depth`.
- Tuning LightGBM core aliases and sklearn names together.
- Using native `Dataset` advice in sklearn-wrapper code without checking
  whether it applies.
- Passing the final test set to `eval_set` for early stopping.
- Optimizing thread counts as hyperparameters instead of fixing them for the
  compute environment.
