# Avito Services Candidate Generation

High-recall candidate generation for Avito service search. The system returns
up to 50 service listing IDs for each query and is evaluated by macro
`Recall@50`.

The project starts with reproducible lexical baselines, then measures whether
historical, semantic and hybrid retrieval improve candidate coverage.

## Status

Current milestone: **M0 — data audit and evaluation contract**.

The full roadmap, experiment hypotheses and ClearML tracking contract are in
[docs/EXPERIMENT_PLAN.md](docs/EXPERIMENT_PLAN.md).

## Layout

```text
configs/                  Versioned experiment configurations
data/                     Local input data only; ignored by Git
docs/                     Project documentation and experiment plan
src/avito_retrieval/      Source package
```

## Local setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

Place the supplied Parquet files under `data/`. They are deliberately excluded
from version control. ClearML is used for development-time experiment tracking;
the final inference pipeline will remain runnable without external APIs.
