# Project Instructions

This document is the practical reference for setting up, running, testing, and
extending the MulCas Ethereum Ponzi detection project. Commands are written for
Windows PowerShell and should be run from the repository root.

For the detailed reproducibility protocol and phase-by-phase output inventory,
see [Running and Testing Guide](RUN_AND_TEST_GUIDE.md). For the research
decisions and completed work, see [Full Current Research Progress Report](FULL_CURRENT_RESEARCH_PROGRESS_REPORT.md).

## 1. Project Overview

The project reproduces and evaluates the MulCas multi-view cascade ensemble for
Ethereum smart-contract Ponzi detection. The workflow includes:

1. Raw dataset auditing and integrity checks.
2. Chronological, leakage-safe dataset splitting.
3. Static opcode feature reproduction.
4. Individual Ridge, SVM, and XGBoost baselines.
5. MulCas cascade and voting reconstruction.
6. Controlled Phase 6 and Phase 7 experiments.
7. Phase 8 representation and extension experiments.
8. Phase 9 protocol and creator-state analyses.

The retained reference model is the frozen Phase 5.5 paper-aligned MulCas
baseline. Later experiments must not overwrite its outputs.

## 2. Repository Layout

| Path | Purpose |
| --- | --- |
| `src/data/` | Dataset auditing, validation, and chronological splits |
| `src/features/` | Opcode, structural, operand, developer, and paper-aligned features |
| `src/models/` | Baselines, MulCas models, and model runners |
| `src/evaluation/` | Metrics and temporal evaluation helpers |
| `src/experiments/` | Phase 6 and Phase 7 experiments |
| `src/extension/` | Phase 8 extension and graph/sequence experiments |
| `src/phase9/` | Phase 9 protocol and creator-state analyses |
| `scripts/` | Standalone research scripts |
| `configs/` | Versioned YAML experiment configurations |
| `tests/` | Pytest test suite |
| `data/raw/` | Local raw dataset; ignored by Git |
| `data/processed/` | Generated datasets and features; ignored by Git |
| `results/` | Compact reports, metrics, traces, and experiment outputs |
| `results/artifacts/` | Large fitted models and transformers; ignored by Git |
| `notebooks/` | Exploratory and presentation notebooks |
| `docs/` | Protocols, reports, and project documentation |

## 3. Requirements

- Windows PowerShell.
- Python 3.11 or newer available as `python`.
- Sufficient disk space for the raw CSV and generated artifacts.
- The released dataset at `data/raw/PonziDataset_20221114.csv`.

The raw dataset is intentionally excluded from Git. It must remain byte-for-byte
unchanged. Do not edit, sort, deduplicate, re-save, or overwrite it.

## 4. Initial Setup

From the repository root:

```powershell
python -m venv .venv
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

The execution-policy command changes only the current PowerShell process. If
activation is already permitted, it can be skipped.

Verify the interpreter and imports:

```powershell
python -c "import sys; print(sys.executable); print(sys.version)"
python -c "import numpy, pandas, sklearn, xgboost, gensim; print('Imports OK')"
```

The interpreter path should end with `.venv\Scripts\python.exe`.

### Optional CUDA environment

The base requirements install the CPU-compatible dependency set. For supported
CUDA experiments, install the CUDA overlay after the base requirements:

```powershell
python -m pip install -r requirements-cuda.txt
```

Only use the CUDA overlay on a machine with a compatible NVIDIA driver and CUDA
runtime. Record the environment and dependency versions for any experiment that
uses it.

## 5. Dataset Setup and Verification

Place the released file at:

```text
data/raw/PonziDataset_20221114.csv
```

Check that it exists and record its digest:

```powershell
Test-Path -LiteralPath data\raw\PonziDataset_20221114.csv
Get-FileHash -Algorithm SHA256 data\raw\PonziDataset_20221114.csv
```

The expected schema is documented in [Dataset Description](dataset_description.md).
Run the project audit before generating derived data:

```powershell
python -m src.data.audit
```

This writes `results/baseline/dataset_audit.json`. The audit output is the
authoritative record of the local file's row count, schema, labels, ranges,
missing values, duplicates, and digest.

## 6. Reproduce the Frozen Baseline

Run the baseline stages in order because later stages consume earlier outputs.

### Stage 1: chronological split

```powershell
python -m src.data.split --config configs/data_split.yaml
```

Outputs:

- `results/baseline/temporal_split_membership.csv`
- `results/baseline/temporal_split_metadata.json`

### Stage 2: feature reproduction

```powershell
python -m src.features.reproduce --config configs/feature_reproduction.yaml
```

The compact summary is written to
`results/baseline/feature_view_summary.json`. Fitted vectorizers and
transformers are stored below the ignored `results/artifacts/` directory.

### Stage 3: individual baselines

```powershell
python -m src.models.run_baselines --config configs/baseline_models.yaml
```

Outputs:

- `results/baseline/individual_baseline_results.json`
- `results/baseline/individual_baseline_results.csv`

### Stage 4: MulCas reconstruction

```powershell
python -m src.models.run_mulcas --config configs/mulcas_reproduction.yaml
```

Outputs include:

- `results/baseline/mulcas_reproduction_results.json`
- `results/baseline/mulcas_traces/validation_trace.csv`
- `results/baseline/mulcas_traces/test_trace.csv`
- `docs/mulcas_baseline_report.md`

### Stage 5.5: paper-aligned correction

This is a separate frozen protocol and must not replace the original baseline:

```powershell
python -m src.data.paper_aligned_split --config configs/paper_aligned_split.yaml
python -m src.features.paper_aligned --config configs/paper_aligned_features.yaml
python -m src.models.run_baselines --config configs/paper_aligned_baseline_models.yaml
python -m src.models.run_mulcas --config configs/paper_aligned_mulcas.yaml
```

Its outputs are stored under `results/baseline/paper_aligned/`.

## 7. Run Tests

Run the full suite before accepting a change:

```powershell
python -m pytest -q
```

Useful focused test commands:

```powershell
python -m pytest tests/test_data_validation.py tests/test_temporal_split.py -q
python -m pytest tests/test_feature_leakage.py -q
python -m pytest tests/test_baseline_models.py tests/test_evaluation_metrics.py -q
python -m pytest tests/test_mulcas.py -q
python -m pytest tests/test_phase8_closure.py -q
```

The suite checks imports, deterministic behavior, data validation, temporal
ordering, leakage controls, feature and model construction, routing/voting,
phase manifests, and closure consistency.

## 8. Run Notebooks

Start Jupyter from the activated environment:

```powershell
python -m jupyter notebook
```

Open notebooks in numerical order. Notebooks are presentation and exploration
layers; reusable research logic belongs in `src/`.

Execute a notebook without replacing the source notebook:

```powershell
New-Item -ItemType Directory -Force results\temp | Out-Null
python -m jupyter nbconvert --execute --to notebook `
  --output-dir results\temp `
  --output mulcas_reproduction_executed.ipynb `
  notebooks\05_mulcas_reproduction.ipynb
```

## 9. Phase 6, 7, 8, and 9 Work

These phases are controlled research workflows. Before running one, inspect its
configuration and frozen-manifest dependencies under `configs/`. Most modules
can be invoked with:

```powershell
python -m <module> --config <config-path>
```

Examples:

```powershell
python -m src.experiments.phase6a
python -m src.experiments.phase6b --config configs/experiments/phase6b/training_sampling.yaml
python -m src.experiments.phase7a
python -m src.extension.phase8_closure
python -m src.phase9.phase9a
python -m src.phase9.phase9c
```

Not every extension module is intended for a routine full rerun. Some require
large generated artifacts, external data, network access, rate-limited
providers, or a sealed evaluation protocol. Read the corresponding phase
documentation and module `--help` output first:

```powershell
python -m src.extension.phase8r2a --help
python -m src.extension.phase8r5b0 --help
```

The standalone static-random comparison script can be inspected with:

```powershell
python scripts\static_random_mulcas.py --help
```

Do not run external-provider enrichment against the final evaluation set unless
the relevant protocol explicitly permits it.

## 10. Configuration and Reproducibility

- Use YAML files under `configs/` instead of hard-coding experiment changes.
- Preserve frozen manifests and record any intentional change.
- Use `src.utils.paths` for repository-relative paths.
- Use `src.utils.seed.set_random_seed` for randomized processing.
- Keep train, validation, and test partitions separate.
- Fit vectorizers, IDF statistics, Word2Vec, scalers, and models on training
  data only.
- Do not use final-test labels to select thresholds, viewer order, or model
  parameters.
- Record the configuration, seed, package versions, dataset digest, and
  observations in `docs/experiment_log.md`.

Before reporting regenerated results:

```powershell
python -m pytest -q
git diff -- data/raw/PonziDataset_20221114.csv
git status --short
python -m pip freeze
```

## 11. API Keys and Secrets

Never commit API keys, RPC URLs containing credentials, `.env` files, or local
secret notes. The repository ignores `Alchemy.txt`, but any exposed key should
still be revoked and replaced.

For local provider-based experiments, use session-only environment variables
with values supplied privately:

```powershell
$env:ETH_RPC_URL = Read-Host "Ethereum RPC URL"
$env:BLOCKSCOUT_API_URL = "https://api.blockscout.com/1/api/v2"
$env:BLOCKSCOUT_API_KEY = Read-Host "Blockscout API key"
```

Do not place real values in this document, source files, YAML configuration,
shell history, or committed notebooks.

## 12. Git and Change Management

Check the worktree before and after an experiment:

```powershell
git status --short --branch
git diff --stat
```

Do not commit:

- `data/raw/`, `data/interim/`, or `data/processed/` datasets.
- `results/artifacts/`, fitted models, or large feature matrices.
- `.venv/`, caches, notebook checkpoints, or generated temporary files.
- Credentials or other secrets.

Keep compact metrics, reports, and traces in their designated `results/` paths.
Use a separate configuration and output namespace for new experiments rather
than overwriting frozen reference results.

## 13. Troubleshooting

### `ModuleNotFoundError`

Run from the repository root, activate the virtual environment, and reinstall:

```powershell
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

### Missing split or feature artifacts

Run the stages in order:

```powershell
python -m src.data.split
python -m src.features.reproduce
```

### Unexpected metric changes

Check the dataset digest, interpreter, dependency versions, configuration
changes, split metadata, and seed:

```powershell
Get-FileHash -Algorithm SHA256 data\raw\PonziDataset_20221114.csv
python -c "import sys; print(sys.executable)"
python -m pip freeze
git diff -- configs results/baseline
```

Regenerate results through the relevant module after identifying the cause;
never edit result files manually to force a metric.

### PowerShell cannot activate `.venv`

Allow scripts only for the current process:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
.\.venv\Scripts\Activate.ps1
```

Alternatively, invoke the environment's interpreter directly:

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

## 14. Research Safety Rules

- Do not modify the raw dataset.
- Do not introduce temporal or feature leakage.
- Do not tune against final-test labels.
- Do not confuse reproduced metrics with the original paper's published
  metrics.
- Do not overwrite frozen baselines with exploratory results.
- Do not evaluate sealed data during development.
- Do not expose or commit credentials.
