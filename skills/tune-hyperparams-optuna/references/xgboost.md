# XGBoost sklearn Tuning with Optuna

Read this when tuning `xgboost.XGBRegressor` or `xgboost.XGBClassifier`.
Prefer the sklearn estimator API and keep native `xgboost.train()` patterns out
of new code unless the project already uses them.

## Source Anchors

- XGBoost sklearn API: https://xgboost.readthedocs.io/en/stable/python/python_api.html#xgboost.XGBRegressor
- XGBoost parameters: https://xgboost.readthedocs.io/en/stable/parameter.html
- XGBoost parameter tuning notes: https://xgboost.readthedocs.io/en/stable/tutorials/param_tuning.html

## Starter Search Space

Use this as the first-pass space for `gbtree` with `tree_method="hist"`.
Set `n_estimators` high and rely on early stopping; do not tune it directly
unless model size or latency is part of the optimization target.

```python
def suggest_xgb_params(trial: optuna.Trial) -> dict[str, object]:
    return {
        "booster": "gbtree",
        "tree_method": "hist",
        "n_estimators": 4000,
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.2, log=True),
        "max_depth": trial.suggest_int("max_depth", 2, 10),
        "min_child_weight": trial.suggest_float("min_child_weight", 1e-2, 64.0, log=True),
        "subsample": trial.suggest_float("subsample", 0.6, 1.0),
        "colsample_bytree": trial.suggest_float("colsample_bytree", 0.5, 1.0),
        "gamma": trial.suggest_float("gamma", 0.0, 10.0),
        "reg_alpha": trial.suggest_float("reg_alpha", 1e-8, 100.0, log=True),
        "reg_lambda": trial.suggest_float("reg_lambda", 1e-3, 100.0, log=True),
    }
```

Use `max_leaves` and `grow_policy="lossguide"` only when there is a concrete
reason to prefer leaf-wise growth. Do not tune `max_depth` and `max_leaves` as
if they were independent complexity controls.

## Estimator Setup

Regression:

```python
import xgboost as xgb

model = xgb.XGBRegressor(
    **params,
    objective="reg:squarederror",
    eval_metric="rmse",
    random_state=seed,
    n_jobs=xgb_threads,
    verbosity=0,
    early_stopping_rounds=100,
)
model.fit(X_train, y_train, eval_set=[(X_valid, y_valid)], verbose=False)
```

Binary classification:

```python
model = xgb.XGBClassifier(
    **params,
    objective="binary:logistic",
    eval_metric="logloss",
    random_state=seed,
    n_jobs=xgb_threads,
    verbosity=0,
    early_stopping_rounds=100,
)
model.fit(X_train, y_train, eval_set=[(X_valid, y_valid)], verbose=False)
```

For AUC-based early stopping, prefer an explicit callback so the direction is
unambiguous:

```python
callbacks = [
    xgb.callback.EarlyStopping(
        rounds=100,
        metric_name="auc",
        data_name="validation_0",
        maximize=True,
        save_best=True,
    )
]
model = xgb.XGBClassifier(**params, eval_metric="auc", callbacks=callbacks)
```

Create callbacks inside each trial and fold.

After fitting with early stopping, `best_iteration` is 0-based. If a later
refit needs a concrete tree count, use `best_iteration + 1`. XGBoost's sklearn
prediction methods use `best_iteration` automatically after early stopping; use
`xgb.callback.EarlyStopping(save_best=True)` when you want to discard trees
after the best iteration during training.

## Pruning

Use pruning mainly for large, expensive single train/validation split studies.
For small CV studies, skip pruning and rely on early stopping. If pruning is
justified, use `XGBoostPruningCallback` from `optuna-integration` with the
sklearn validation-set index in the observation key.

```python
from optuna_integration import XGBoostPruningCallback

callbacks = [XGBoostPruningCallback(trial, "validation_0-logloss")]
model = xgb.XGBClassifier(
    **params,
    objective="binary:logistic",
    eval_metric="logloss",
    callbacks=callbacks,
    random_state=seed,
)
model.fit(X_train, y_train, eval_set=[(X_valid, y_valid)], verbose=False)
```

Use `validation_0-rmse` for RMSE, `validation_0-auc` for AUC, and
`validation_0-mlogloss` for multiclass log loss. The key must match the
reported metric name exactly.

## Parameter Effects

- `learning_rate`: smaller values are usually safer but require more boosting
  rounds. Search on a log scale.
- `max_depth`: primary depth-wise complexity control. Larger values fit
  interactions but overfit faster.
- `min_child_weight`: raises the minimum Hessian needed in a child. Larger
  values make splits more conservative.
- `gamma`: requires a minimum split gain. Increase to suppress weak splits.
- `subsample`: row sampling per boosting iteration. Values below 1.0 add
  randomness and can reduce overfitting.
- `colsample_bytree`, `colsample_bylevel`, `colsample_bynode`: feature sampling
  controls. Tune one first, usually `colsample_bytree`; combinations multiply.
- `reg_alpha`: L1 regularization. Useful for sparse/high-dimensional data.
- `reg_lambda`: L2 regularization. A strong general-purpose stabilizer.
- `scale_pos_weight`: useful for imbalanced binary classification when the
  chosen metric tolerates reweighting, such as ROC AUC or average precision.
- `max_delta_step`: consider for severely imbalanced logistic tasks where
  probability calibration matters.

## Classification Details

For imbalanced binary classification, start with:

```python
ratio = negative_count / positive_count
params["scale_pos_weight"] = trial.suggest_float(
    "scale_pos_weight",
    ratio * 0.25,
    ratio * 4.0,
    log=True,
)
```

Do not blindly optimize threshold-dependent metrics inside the estimator search.
Tune model hyperparameters with a probability/ranking metric first, then tune
the decision threshold separately on validation data if the business metric
requires it.

For multiclass classification, use `XGBClassifier` with
`objective="multi:softprob"` when probabilities are needed. Use `mlogloss` for a
stable first metric, then report task-specific metrics after refit.

## Search Expansion

Add these only after the starter space has run:

- `max_bin`: for histogram speed/accuracy tradeoffs.
- `sampling_method`: usually keep `uniform`; try `gradient_based` only with
  compatible histogram settings and a reason to lower `subsample`.
- `booster="dart"`: use a conditional branch with DART-specific dropout
  parameters. DART can improve generalization but makes runs slower and search
  noisier.
- `enable_categorical`, `max_cat_to_onehot`, `max_cat_threshold`: use only when
  the project intentionally relies on native categorical handling.
- `monotone_constraints` and `interaction_constraints`: encode domain knowledge;
  do not treat them as ordinary hyperparameters.

## Common Mistakes

- Tuning `n_estimators` tightly while also using early stopping.
- Forgetting that `best_iteration` is 0-based when converting it to
  `n_estimators` for a final refit.
- Pruning a whole CV trial from one noisy fold's boosting curve.
- Using the final test set as `eval_set`.
- Reusing callback objects across trials or folds.
- Tuning both an alias and its canonical sklearn parameter name.
- Optimizing accuracy on heavily imbalanced data without checking ranking or
  calibration metrics.
- Letting Optuna parallel workers and XGBoost threads oversubscribe the CPU.
