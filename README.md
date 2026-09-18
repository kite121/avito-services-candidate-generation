# Avito Services Candidate Generation

High-recall candidate generation for Avito service search. The system returns
up to 50 service listing IDs for each query and is evaluated by macro
`Recall@50`.

The project starts with reproducible lexical baselines, then measures whether
historical, semantic and hybrid retrieval improve candidate coverage.

## Status

Current milestone: **M3 — zero-shot semantic retrieval**. Completed analysis notebooks:
`output/jupyter-notebook/m0_data_audit.ipynb` and
`output/jupyter-notebook/m1_lexical_retrieval.ipynb` and
`output/jupyter-notebook/m2_historical_query_retrieval.ipynb`.

The full roadmap, experiment hypotheses and ClearML tracking contract are in
[docs/EXPERIMENT_PLAN.md](docs/EXPERIMENT_PLAN.md).

The current lexical baseline is RRF over stemmed all-field BM25 and
control-normalized title char-TFIDF (`Recall@50 = 0.312988` on the frozen
proxy). Its reusable local text cache is
`artifacts/text_preprocessing/m1_lexical_best_v1/`; it is derived data and is
intentionally ignored by Git.

## Layout

```text
configs/                  Versioned experiment configurations
data/                     Local input data only; ignored by Git
artifacts/text_preprocessing/  Reusable derived text representations; ignored by Git
docs/                     Project documentation and experiment plan
output/jupyter-notebook/   Versioned executed analysis notebooks
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
