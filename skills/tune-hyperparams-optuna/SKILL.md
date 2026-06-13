---
name: tune-hyperparams-optuna
description: Use when designing, implementing, reviewing, or debugging hyperparameter optimization with Optuna for boosted-tree models, especially xgboost.XGBRegressor/XGBClassifier and LightGBM sklearn estimators LGBMRegressor/LGBMClassifier. Covers search spaces, pruning, early stopping, validation design, objective functions, sampler/pruner choices, and final refit/evaluation.
---

# Tune Hyperparams with Optuna

Use this skill to build robust Optuna studies for boosted-tree models. Prefer
the sklearn estimator APIs unless the local codebase already uses native
training APIs or the user explicitly asks for them.

## Workflow

1. Define the modeling target.
   - Identify regression, binary classification, multiclass classification, or
     ranking before writing a search space.
   - Choose the metric and Optuna direction up front. Maximize scores such as
     AUC, average precision, F1, accuracy, and R2. Minimize losses such as RMSE,
     MAE, log loss, and error rates.
   - Check whether the validation scheme must be stratified, grouped, temporal,
     or nested.

2. Protect evaluation integrity.
   - Keep preprocessing inside a `Pipeline` or apply identical transformations
     within each fold. Avoid fitting encoders, imputers, scalers, or selectors on
     validation or test data.
   - Put oversampling, undersampling, and outlier rejection inside the CV
     pipeline. Never resample the full dataset before splitting.
   - Tune only on train/validation data. Use a final test set once, after the
     study and final refit.
   - Reuse the same CV folds or validation split across trials so trial scores
     are comparable.

3. Load only the reference needed for the estimator.
   - Read [references/optuna-patterns.md](references/optuna-patterns.md) when
     writing or reviewing the Optuna study, objective function, pruning, storage,
     parallelism, or final refit.
   - Read [references/xgboost.md](references/xgboost.md) for
     `XGBRegressor` or `XGBClassifier`.
   - Read [references/lightgbm.md](references/lightgbm.md) for
     `LGBMRegressor` or `LGBMClassifier`. Use the sklearn API, not
     `lightgbm.train()` or `LightGBMTuner`, unless requested.

4. Start with a compact, high-leverage search space.
   - Tune 5-9 parameters first: learning rate, tree complexity, child/leaf
     constraints, row/feature subsampling, and regularization.
   - Use log distributions for positive scale parameters such as
     `learning_rate`, `reg_alpha`, `reg_lambda`, and Hessian/child-weight terms.
   - Do not tune aliases for the same underlying parameter in one study.
   - Add conditional branches only when the branch changes valid parameters, for
     example `booster="dart"` or positive `max_depth` with a constrained
     `num_leaves`.
   - When tuning a sklearn or imblearn pipeline, use step-prefixed parameter
     names such as `clf__learning_rate` and `imputer__n_neighbors`, and keep
     preprocessing search narrow.

5. Use early stopping by default and pruning selectively.
   - Prefer a large boosting budget (`n_estimators`) plus estimator-native early
     stopping over treating `n_estimators` as the main search parameter.
   - Tune `n_estimators` directly only when model size or latency is a hard
     constraint, early stopping cannot be wired cleanly, or validation scores are
     too noisy for reliable stopping.
   - Record the fitted boosting length from each trial or fold so the final
     model can be refit with either early stopping or a materialized tree count.
   - Default to no Optuna pruning for small datasets, fast trials, and
     cross-validation objectives where one fold can be noisy.
   - Use pruning mainly for expensive studies with a stable validation signal,
     especially large datasets with a single train/validation split.
   - Create fresh callback objects for each trial and fold; callback state must
     not be reused across fits.
   - Use `optuna-integration` pruning callbacks only when pruning is justified
     and a validation metric is reported each boosting iteration. Match the
     callback's metric name to the estimator's `eval_metric` and validation-set
     name.

6. Refit and report the result.
   - Store useful trial attributes such as per-fold scores, best iteration, and
     training time.
   - After optimization, refit with `study.best_params` using the agreed
     training data. If early stopping was used, either keep an internal
     validation set for the final fit or set `n_estimators` from the observed
     boosting-round policy before fitting on more data.
   - Report the validation protocol, metric direction, number of completed and
     pruned trials, best parameters, best value, final test metric if available,
     and any known caveats.

## Study Defaults

Use Optuna's default TPE sampler as the baseline, but instantiate it explicitly
when reproducibility or distributed behavior matters:

```python
sampler = optuna.samplers.TPESampler(seed=seed)
pruner = optuna.pruners.NopPruner()
study = optuna.create_study(
    direction=direction,
    sampler=sampler,
    pruner=pruner,
    study_name=study_name,
    storage=storage_url,
    load_if_exists=True,
)
study.optimize(objective, n_trials=n_trials, timeout=timeout, n_jobs=1)
```

Use `NopPruner` as the default for small and CV-based boosted-tree studies. For
expensive single-split studies, switch to `HyperbandPruner` with TPE or
`MedianPruner` as a simpler conservative fallback. Keep `study.optimize(n_jobs=1)`
unless estimator threads are controlled; nested parallelism can make studies
slower and less reproducible.

## Review Checklist

- The objective returns a single scalar whose direction matches the study.
- Validation data used for early stopping is not the final test set.
- The model is recreated inside every trial and fold.
- Search ranges are valid for the estimator and do not duplicate aliases.
- Pruning is disabled for small or fast CV studies unless there is a deliberate
  aggregate-fold pruning policy.
- Pruning callbacks, when used, observe the same metric and validation name used
  in fit.
- Classification metrics use probabilities or decision scores when required.
- Class imbalance handling is metric-aligned and not mixed with incompatible
  weighting options.
- Pipeline parameters use the correct `step__parameter` names, and any
  resampling is inside the cross-validation loop.
- The final model is refit and evaluated outside the Optuna objective.
