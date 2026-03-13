# autoresearch

This is an experiment to have the LLM do its own ML research.

## Setup

To set up a new experiment:

1. **Stay on master**: do not create feature branches. All commits go directly to master.
2. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `README.md` — repository context.
   - `prepare.py` — fixed constants, data loading, train/val split, evaluation. Do not modify.
   - `train.py` — the file you modify. Feature engineering, model, hyperparameters.
3. **Verify dependencies**: Run `uv sync` to ensure packages are installed.
4. **Initialize results.tsv**: If `results.tsv` is empty (header only), the baseline will be recorded after the first run.
5. **Confirm and go**: Confirm setup looks good.

3. **Ask the human how long to run**: Before starting the loop, ask "How many minutes should I run the autoresearch loop?" and wait for their answer. Use that as the time budget.

Once you get confirmation, kick off the experimentation.

## Experimentation

Each experiment runs quickly (seconds to a minute). You launch it simply as: `uv run train.py`.

**What you CAN do:**
- Modify `train.py` — this is the only file you edit. Everything is fair game: feature engineering, target transformation (e.g. log price), XGBoost hyperparameters, alternative models from scikit-learn, new derived features, feature selection, interaction terms, etc.

**What you CANNOT do:**
- Modify `prepare.py`. It is read-only. It contains the fixed train/val split and the ground-truth metric.
- Modify `plot.py`. It is read-only. It generates `progress.png` and is called automatically at the end of `train.py`.
- Modify the `# Chart (do not modify)` section at the bottom of `train.py`.
- Install new packages or add dependencies. You can only use what's already in `pyproject.toml` (pandas, numpy, scikit-learn, xgboost, matplotlib).
- Modify the evaluation harness. The `evaluate_model` function in `prepare.py` is the ground truth metric.

**The goal is simple: get the lowest val_rmse (£).** The val split is fixed, so all experiments are directly comparable. Everything in `train.py` is fair game.

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Conversely, removing something and getting equal or better results is a great outcome.

**The first run**: Your very first run should always be to establish the baseline, so run the training script as-is.

## Output format

Once the script finishes it prints a summary and regenerates `progress.png` automatically:

```
---
val_rmse:         1234.56
training_seconds: 1.2
num_samples:      50000
num_features:     6
progress.png updated
```

Extract the key metric from the log file:

```
grep "^val_rmse:" run.log
```

## Logging results

When an experiment is done, log it to `results.tsv` (tab-separated, NOT comma-separated — commas break in descriptions).

The TSV has a header row and 4 columns:

```
commit	val_rmse	status	description
```

1. git commit hash (short, 7 chars)
2. val_rmse achieved (e.g. 1234.56) — use 0.00 for crashes
3. status: `keep`, `discard`, or `crash`
4. short text description of what this experiment tried

## The experiment loop

LOOP FOREVER on master:

1. Read the current `results.tsv` to know the current best val_rmse.
2. Tune `train.py` with an experimental idea.
3. Run: `uv run train.py > run.log 2>&1`
4. Check: `grep "^val_rmse:" run.log`
5. If crashed: `tail -n 50 run.log` to debug.
6. Append result to `results.tsv` (never commit this file).
7. If val_rmse improved → `git add train.py && git commit` directly to master, then `git push`.
8. If val_rmse equal or worse → `git checkout train.py` to revert.

**Crashes**: Fix obvious bugs and re-run. If the idea is fundamentally broken, log "crash" and move on.

**NEVER STOP**: Loop until manually interrupted. If you run out of ideas, try harder — feature engineering, target transformations, regularization, ensembling, model stacking, etc.
