# Optuna Patterns for Boosted Trees

Read this when implementing or reviewing the Optuna study, objective function,
pruning setup, persistence, parallelism, and final refit around boosted-tree
estimators.

## Contents

- [Source Anchors](#source-anchors)
- [Study Setup](#study-setup)
- [Objective Shape](#objective-shape)
- [Cross-Validation Pattern](#cross-validation-pattern)
- [Pipeline-Level Tuning](#pipeline-level-tuning)
- [Custom Probability Metrics](#custom-probability-metrics)
- [Boosting Rounds Policy](#boosting-rounds-policy)
- [Early Stopping and Pruning](#early-stopping-and-pruning)
- [Trial Budgets](#trial-budgets)
- [Final Refit](#final-refit)

## Source Anchors

- Optuna Pythonic search spaces: https://optuna.readthedocs.io/en/stable/tutorial/10_key_features/002_configurations.html
- Optuna samplers and pruners: https://optuna.readthedocs.io/en/stable/tutorial/10_key_features/003_efficient_optimization_algorithms.html
- Optuna `create_study`: https://optuna.readthedocs.io/en/stable/reference/generated/optuna.create_study.html
- Reusing the best trial: https://optuna.readthedocs.io/en/stable/tutorial/20_recipes/010_reuse_best_trial.html
- XGBoost pruning callback: https://optuna-integration.readthedocs.io/en/stable/reference/generated/optuna_integration.XGBoostPruningCallback.html
- LightGBM pruning callback: https://optuna-integration.readthedocs.io/en/stable/reference/generated/optuna_integration.LightGBMPruningCallback.html

## Study Setup

Use Optuna's define-by-run API for conditional spaces. Prefer
`suggest_float`, `suggest_int`, and `suggest_categorical`; use `log=True` for
positive parameters that span orders of magnitude. Avoid adding low-impact
parameters early because search difficulty grows quickly with dimension.

Good defaults:

```python
import optuna

def make_study(direction: str, seed: int, storage_url: str | None = None) -> optuna.Study:
    sampler = optuna.samplers.TPESampler(seed=seed)
    pruner = optuna.pruners.NopPruner()
    return optuna.create_study(
        direction=direction,
        sampler=sampler,
        pruner=pruner,
        storage=storage_url,
        study_name="boosting-hpo",
        load_if_exists=storage_url is not None,
    )
```

Use in-memory studies for disposable local experiments. Use an RDB storage URL
such as SQLite for resumable local runs and a proper database for multi-worker
runs. For distributed optimization with expensive trials, consider
`TPESampler(seed=seed, constant_liar=True)` so workers avoid sampling very
similar configurations.

Avoid deprecated or low-level `TPESampler` knobs unless a project already has a
reasoned policy for them. The normal decision surface is search-space design,
validation design, trial budget, sampler seed, and pruner choice.

## Objective Shape

Keep the objective deterministic except for the trial suggestions. Recreate the
model and callbacks inside the objective.

```python
import numpy as np
from sklearn.metrics import mean_squared_error

def objective(trial: optuna.Trial) -> float:
    params = build_params(trial)
    model = make_estimator(params)
    model.fit(
        X_train,
        y_train,
        eval_set=[(X_valid, y_valid)],
        verbose=False,
    )
    pred = model.predict(X_valid)
    rmse = float(np.sqrt(mean_squared_error(y_valid, pred)))
    trial.set_user_attr("best_iteration", getattr(model, "best_iteration_", None))
    return rmse
```

For classification, compute metrics from the right prediction surface:
`predict_proba()` for log loss, ROC AUC, average precision, and calibration;
`predict()` for accuracy, F1, balanced accuracy, and confusion-matrix metrics.

## Cross-Validation Pattern

Use fixed splits for all trials. Recreate the estimator for each fold.

```python
import numpy as np

def objective(trial: optuna.Trial) -> float:
    scores: list[float] = []
    for fold, (train_idx, valid_idx) in enumerate(cv.split(X, y, groups)):
        X_train, X_valid = X.iloc[train_idx], X.iloc[valid_idx]
        y_train, y_valid = y.iloc[train_idx], y.iloc[valid_idx]

        model = make_estimator(build_params(trial, fold=fold))
        model.fit(X_train, y_train, eval_set=[(X_valid, y_valid)], verbose=False)
        scores.append(score_model(model, X_valid, y_valid))

    trial.set_user_attr("fold_scores", scores)
    trial.set_user_attr("fold_score_std", float(np.std(scores)))
    return float(np.mean(scores))
```

For time series, use forward-chaining splits and never shuffle. For grouped data,
use group-aware splitting. For imbalanced classification, use stratified splits
unless a stronger grouped or temporal constraint prevents that.

## Pipeline-Level Tuning

Tune preprocessing and model parameters together when preprocessing choices
materially affect the model. Keep this search narrow; model parameters already
consume most of the trial budget.

Use sklearn's `step__parameter` naming convention for pipeline parameters. This
works with `Pipeline.set_params()` and makes Optuna's best parameters directly
reusable.

```python
from imblearn.pipeline import Pipeline
from imblearn.over_sampling import SMOTE
from sklearn.impute import KNNImputer
from sklearn.model_selection import cross_val_score
from sklearn.preprocessing import RobustScaler
import xgboost as xgb

def objective(trial: optuna.Trial) -> float:
    clf = xgb.XGBClassifier(
        objective="binary:logistic",
        tree_method="hist",
        eval_metric="logloss",
        random_state=seed,
        n_jobs=estimator_threads,
    )
    pipeline = Pipeline(
        [
            ("imputer", KNNImputer()),
            ("scaler", RobustScaler()),
            ("oversampler", SMOTE(random_state=seed)),
            ("clf", clf),
        ]
    )
    params = {
        "imputer__n_neighbors": trial.suggest_int("imputer__n_neighbors", 3, 15),
        "oversampler__k_neighbors": trial.suggest_int("oversampler__k_neighbors", 3, 15),
        "clf__learning_rate": trial.suggest_float("clf__learning_rate", 0.01, 0.2, log=True),
        "clf__max_depth": trial.suggest_int("clf__max_depth", 2, 10),
        "clf__subsample": trial.suggest_float("clf__subsample", 0.6, 1.0),
        "clf__colsample_bytree": trial.suggest_float("clf__colsample_bytree", 0.5, 1.0),
    }
    pipeline.set_params(**params)
    scores = cross_val_score(pipeline, X, y, scoring=scorer, cv=cv, n_jobs=1)
    trial.set_user_attr("fold_scores", scores.tolist())
    return float(scores.mean())
```

Use `imblearn.pipeline.Pipeline` when a sampler such as SMOTE is part of the
workflow. Samplers must run only on each fold's training split; applying
oversampling or undersampling before CV leaks information and produces
optimistic validation scores.

Good pipeline-level candidates:

- imputer family or a single imputer knob such as `n_neighbors`;
- sampler on/off or sampler strength for imbalanced classification;
- a small feature-selection or dimensionality-reduction choice;
- estimator hyperparameters.

Poor pipeline-level candidates:

- many correlated preprocessing knobs in the same first-pass study;
- feature engineering based on validation or test labels;
- nested parallelism with Optuna workers, CV workers, and estimator threads all
  unconstrained.

## Custom Probability Metrics

Competition and business metrics often differ from built-in estimator metrics.
For probability metrics, make the objective compute or score probabilities
explicitly and align the Optuna direction with the returned value.

For imbalanced binary log-loss variants, clip probabilities before taking logs
and average class-specific losses when each class should contribute equally:

```python
import numpy as np

def balanced_log_loss(y_true, proba_positive):
    y_true = np.asarray(y_true)
    p1 = np.clip(np.asarray(proba_positive), 1e-15, 1 - 1e-15)
    p0 = 1.0 - p1
    n0 = np.sum(y_true == 0)
    n1 = np.sum(y_true == 1)
    loss0 = -np.sum((y_true == 0) * np.log(p0)) / n0
    loss1 = -np.sum((y_true == 1) * np.log(p1)) / n1
    return float((loss0 + loss1) / 2.0)
```

If using `cross_val_score`, remember that sklearn scorers are usually
"higher-is-better". Either return a positive score and maximize it, or return a
loss and create the study with `direction="minimize"`. Do not mix a negated
sklearn scorer with an Optuna direction without checking the sign in the final
report.

## Boosting Rounds Policy

Prefer a generous `n_estimators` upper bound plus early stopping instead of
treating `n_estimators` as a normal independent Optuna parameter. The number of
trees is strongly coupled to `learning_rate`: smaller steps normally need more
rounds. Let early stopping adapt the tree count to each trial's learning rate
and regularization settings.

Use direct `n_estimators` tuning only when:

- model size, latency, memory, or export format imposes a strict tree budget;
- the estimator is inside an interface where early stopping cannot receive a
  valid per-fold validation set;
- validation metrics are too noisy for stable early stopping;
- doing a low-fidelity first pass where a small fixed budget is intentional.

During CV-based studies, record the fitted boosting length per fold:

```python
def fitted_boosting_rounds(model) -> int | None:
    if hasattr(model, "best_iteration"):  # XGBoost sklearn
        return int(model.best_iteration) + 1
    if hasattr(model, "best_iteration_") and model.best_iteration_:
        return int(model.best_iteration_)  # LightGBM sklearn
    if hasattr(model, "n_estimators_"):
        return int(model.n_estimators_)
    if hasattr(model, "n_iter_"):
        return int(model.n_iter_)
    return None
```

For an estimator at the end of a pipeline, fetch the final step first:

```python
model = pipeline.named_steps["clf"]
trial.set_user_attr("boosting_rounds", fitted_boosting_rounds(model))
```

For final refit, choose one policy deliberately:

1. Keep early stopping and reserve a final internal validation set. This is the
   most faithful to the study procedure.
2. Materialize `n_estimators` from the CV history, for example median, mean, or
   75th percentile of fold best rounds plus a small margin, then fit on more
   training data without early stopping.
3. If deployment cost matters, choose a conservative percentile or explicit cap
   and measure the validation/test tradeoff.

Do not report an Optuna best value without also recording how the final boosting
length was chosen.

## Early Stopping and Pruning

Early stopping decides when a model stops adding trees within a trial. Optuna
pruning decides whether an entire trial should stop early. Use early stopping
freely; use Optuna pruning only when the trial is expensive and the intermediate
validation signal is stable enough to support early trial termination.

Default pruning policy:

- Use `NopPruner` for small datasets, fast trials, and normal CV objectives.
- Use pruning for large datasets with a single train/validation split, where
  each boosting run is expensive and a validation curve is meaningful.
- Avoid pruning a whole CV trial from one noisy fold. A weak first fold can
  discard a configuration that would have averaged well across folds.
- If CV pruning is truly needed, report only aggregate fold-level values after
  each completed fold, wait for at least 2-3 folds before pruning, and use a
  conservative pruner/warmup.

Study setup:

```python
pruner = optuna.pruners.NopPruner()

if use_pruning_for_expensive_single_split:
    pruner = optuna.pruners.HyperbandPruner()

study = optuna.create_study(direction=direction, sampler=sampler, pruner=pruner)
```

Install `optuna-integration` when using pruning callbacks:

```python
from optuna_integration import LightGBMPruningCallback, XGBoostPruningCallback
```

Use observation names exactly as the estimator reports them.

- XGBoost sklearn API: include the validation-set index, for example
  `validation_0-logloss`, `validation_0-auc`, or `validation_0-rmse`.
- LightGBM sklearn API: pass `eval_names=["valid"]` and use
  `LightGBMPruningCallback(trial, metric, valid_name="valid")`.

If a pruning callback is not available or does not fit the local code, use
manual reporting after each training stage only when the estimator exposes
meaningful staged validation scores. Otherwise skip pruning and rely on early
stopping.

For fold-level CV pruning, avoid estimator integration callbacks and report
completed-fold aggregates instead:

```python
scores: list[float] = []
for fold, (train_idx, valid_idx) in enumerate(cv.split(X, y, groups), start=1):
    score = fit_and_score_one_fold(train_idx, valid_idx, trial)
    scores.append(score)

    running_score = float(np.mean(scores))
    trial.report(running_score, step=fold)
    if fold >= min_folds_before_pruning and trial.should_prune():
        raise optuna.TrialPruned()

return float(np.mean(scores))
```

This saves less compute than boosting-iteration callbacks, but it avoids making
the pruning decision from a single fold's early validation curve.

## Trial Budgets

Use a small smoke test first: 5-10 trials with a reduced `n_estimators` or
timeout to validate code paths, metrics, callbacks, and storage. Then run the
real study.

Rough starting budgets:

- 30-60 trials: quick search or expensive CV.
- 100-300 trials: normal tabular boosted-tree search.
- More trials: only after narrowing the space from study history or parameter
  importance.

Prefer `timeout` when running interactively or in CI. Prefer `n_trials` for
scheduled experiments with predictable costs.

## Final Refit

After optimization, do not treat the best validation score as the final model
quality. Refit and evaluate deliberately.

1. Extract `study.best_params`.
2. Decide how to set the final boosting length:
   - keep early stopping with a final internal validation set, or
   - set `n_estimators` from the observed fold-level boosting rounds plus a small
     margin, then fit on more training data.
3. Evaluate once on untouched test data.
4. Save best parameters, package versions, random seeds, split policy, metric,
   direction, number of trials, final boosting-length policy, and final
   estimator configuration.

When reusing `study.best_trial` to rerun objective-like code, remember it is a
frozen trial. Do not expect pruning behavior in that replay.
