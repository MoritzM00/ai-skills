---
name: python-tooling
description: Modern Python project tooling — uv for environments and dependencies, ruff for linting and formatting, ty for type checking, and pyproject.toml configuration.
origin: ECC
---

# Python Tooling

The modern Python toolchain: `uv` for environments and dependency management,
`ruff` for linting and formatting, and `ty` for type checking — all configured
through a single `pyproject.toml`.

## When to Activate

- Setting up a new Python project
- Configuring linting, formatting, or type checking
- Managing dependencies, virtual environments, or lockfiles
- Reviewing or updating `pyproject.toml`
- Wiring tooling into CI or pre-commit

## Toolchain at a Glance

| Concern | Tool | Replaces |
|---------|------|----------|
| Environments & dependencies | `uv` | pip, pip-tools, virtualenv, poetry |
| Lint & format | `ruff` | flake8, isort, black, pylint |
| Type checking | `ty` | mypy, pyright |
| Testing | `pytest` (via `uv run`) | — |

`uv` and `ruff` are production-ready. `ty` is Astral's type checker and is still
pre-1.0 (preview) — pin it loosely and expect its config schema to evolve.

## Essential Commands

```bash
# Linting and formatting (ruff handles both)
ruff check .          # lint
ruff check --fix .    # lint and autofix
ruff format .         # format

# Type checking (ty)
ty check

# Testing
uv run pytest --cov=mypackage --cov-report=html

# Dependency management (uv)
uv sync               # install/sync the locked environment
uv add requests       # add a dependency
uv add --dev pytest   # add a dev dependency
uv lock --upgrade     # refresh the lockfile
uv run <command>      # run a command inside the project environment
```

## Dependency Management with uv

`uv` manages the virtual environment, resolves dependencies, and maintains a
`uv.lock` lockfile for reproducible installs.

```bash
# Start a new project (creates pyproject.toml, .venv, uv.lock)
uv init mypackage

# Add / remove dependencies (updates pyproject.toml and uv.lock)
uv add "pydantic>=2.0.0"
uv remove requests

# Reproduce the locked environment exactly
uv sync --frozen

# Run tools without activating the venv manually
uv run pytest
uv run python -m mypackage
```

- Commit `uv.lock` for applications; libraries usually rely on version ranges in
  `pyproject.toml` and may omit the lockfile from releases.
- `uv run` resolves and syncs the environment before running, so it always uses
  the declared dependencies.

## pyproject.toml Configuration

A single file configures the project metadata and every tool.

```toml
[project]
name = "mypackage"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.31.0",
    "pydantic>=2.0.0",
]

# Managed with uv: `uv add --dev <pkg>` writes here.
[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=5.0.0",
    "ruff>=0.6.0",
    "ty>=0.0.1",
]

[tool.ruff]
line-length = 100   # ruff's built-in default is 88
target-version = "py312"

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # isort (import sorting)
    "N",    # pep8-naming
    "UP",   # pyupgrade (modernize syntax)
    "B",    # flake8-bugbear (likely bugs, e.g. mutable defaults)
    "C4",   # flake8-comprehensions
    "SIM",  # flake8-simplify
    "PTH",  # flake8-use-pathlib (prefer pathlib over os.path)
    "NPY",  # NumPy-specific rules (deprecated API usage)
    "PD",   # pandas-vet (idiomatic pandas; some rules are opinionated)
    "A",    # flake8-builtins (don't shadow list, dict, id, ...)
    "D",    # pydocstyle (enforce docstrings; see convention below)
]
extend-ignore = [
    "E501",    # line length (let the formatter handle wrapping)
    "A003",    # builtin as a class attribute is fine (e.g. model.id, .type)
    "D100",    # Missing docstring in public module
    "D104",    # Missing docstring in public package
    "D203",    # blank line required before class docstring
    # "PD901",  # allow `df` as a DataFrame variable name
    # "PD011",  # .values false-positives on non-pandas objects
]

[tool.ruff.lint.extend-per-file-ignores]
"__init__.py" = ["F401"]  # re-exports: unused imports are intentional

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ty.environment]
python-version = "3.12"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=mypackage --cov-report=term-missing"
```

## Ruff: Lint and Format

`ruff` replaces flake8, isort, and black with a single fast tool.

- `ruff format` is a near drop-in for black; set `line-length` in `[tool.ruff]`
  (the example uses 100; ruff's own default is 88).
- Import sorting is the `I` lint rule — `ruff check --fix .` sorts imports, so a
  separate `isort` run is unnecessary.
- Add rule families to `[tool.ruff.lint].select` as needed (e.g. `UP` for
  pyupgrade, `B` for flake8-bugbear, `SIM` for flake8-simplify).

```bash
ruff check --fix . && ruff format .   # typical pre-commit sequence
```

## ty: Type Checking

`ty` is Astral's type checker, configured under `[tool.ty]`.

```bash
ty check          # check the project
ty check src/     # check a specific path
```

Pin `ty` loosely (`ty>=0.0.1`) while it is in preview; verify config keys against
the installed version, since the schema is still changing.

### Adopting ty incrementally

Turning a type checker on across an existing codebase produces a wall of errors.
Roll it out gradually: scope it to the code you have cleaned up, and don't fail
the build on warnings yet.

```toml
[tool.ty.src]
# Only check what's ready; expand this list as modules get annotated.
include = ["src", "scripts/inference.py"]
exclude = [
    "src/mypackage/data/**",
    "src/mypackage/legacy/**",
]

[tool.ty.terminal]
# See warnings without failing CI while the codebase is brought up to standard.
error-on-warning = false
output-format = "full"
```

As modules get typed, shrink `exclude` and eventually flip `error-on-warning`
to `true`.

## Pre-commit Hooks

Run `ruff` and `ty` as `repo: local` hooks with `language: system`, so they use
the exact versions pinned in your dev dependency group — no separate mirror
version to keep in sync with CI.

```yaml
# .pre-commit-config.yaml
repos:
  # Strip notebook outputs before committing
  - repo: https://github.com/kynan/nbstripout
    rev: 0.9.1
    hooks:
      - id: nbstripout

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: check-yaml
      - id: check-toml
      - id: check-added-large-files
      - id: end-of-file-fixer
      - id: trailing-whitespace

  - repo: local
    hooks:
      - id: ruff-check
        name: ruff check
        entry: ruff check --fix
        language: system
        types_or: [python, pyi, jupyter]   # also lints notebooks
      - id: ruff-format
        name: ruff format
        entry: ruff format
        language: system
        types_or: [python, pyi, jupyter]
      - id: ty-check
        name: ty
        entry: ty check
        language: system
        types: [python]
        pass_filenames: false   # ty checks the whole project, not per-file
```

Two non-obvious details: the `ty` hook sets `pass_filenames: false` because `ty`
analyzes the project as a whole rather than a file list, and the ruff hooks
include `jupyter` in `types_or` so notebooks are linted and formatted too.

Install the hooks once with `uv run pre-commit install`. When starting work on a
project, run `uv run pre-commit autoupdate` to bump the pinned `rev:` tags of the
remote hook repos (e.g. `nbstripout`, `pre-commit-hooks`) to their latest
releases. The `repo: local` hooks aren't pinned here — they track the `ruff`/`ty`
versions in your dev dependency group, so bump those via `uv lock --upgrade`.

## CI Sketch

```bash
# Run the same checks in CI
uv sync --frozen
uv run ruff check .
uv run ruff format --check .
uv run ty check
uv run pytest --cov=mypackage
```

## Quick Reference

| Task | Command |
|------|---------|
| Create project | `uv init` |
| Add dependency | `uv add <pkg>` |
| Add dev dependency | `uv add --dev <pkg>` |
| Sync environment | `uv sync` |
| Refresh lockfile | `uv lock --upgrade` |
| Run in env | `uv run <cmd>` |
| Lint | `ruff check .` |
| Lint + autofix | `ruff check --fix .` |
| Format | `ruff format .` |
| Type check | `ty check` |

**Remember**: one `pyproject.toml`, three tools (`uv`, `ruff`, `ty`). Prefer
`uv run` over manually activating a virtualenv so commands always use the
declared, locked environment.
