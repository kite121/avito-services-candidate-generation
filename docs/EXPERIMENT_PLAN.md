# Avito Services Candidate Generation — experiment plan

## Goal

For every query from `benchmark_queries.parquet`, return at most 50 unique
`item_id` values from `benchmark_items.parquet`. The primary objective is the
official macro `Recall@50`:

```text
mean_query(|predicted_top50 ∩ relevant_items| / |relevant_items|)
```

The project is candidate generation, not final relevance ranking: an item that
is absent from the 50 submitted candidates cannot be recovered downstream.

## Current status

- **Project state:** M0 complete; M1 lexical baseline is next.
- **Current milestone:** `M1 — lexical retrieval`.
- **Rule:** do not claim an improvement until it was measured on the frozen
  local validation protocol below.

## Project stages at a glance

| Stage | Experiments | Purpose | Exit result | Status |
| --- | --- | --- | --- | --- |
| M0 — Data audit | — | Verify files, overlaps, labels and evaluation protocol | Frozen corpus/split, data report, metric and submission validator | Complete |
| M1 — Lexical retrieval | E01–E03 | Establish and strengthen BM25-based search | Best lexical baseline with measured Recall@50 | Next |
| M2 — Historical signal | E04 | Test general query-to-clicked-item retrieval from train | Accepted or rejected history source with documented coverage | Planned |
| M3 — Zero-shot dense | E05 | Benchmark semantic bi-encoders | One or two complementary dense retrievers | Planned |
| M4 — Domain adaptation | E06–E07 | Fine-tune the selected bi-encoder with mined negatives | Confirmed fine-tuned checkpoint or explicit rejection | Planned |
| M5 — Hybrid candidates | E08–E09 | Measure source complementarity and fuse candidates | Frozen candidate pool and fusion baseline | Planned |
| M6 — Final-50 selector | E10–E11 | Select the best 50 from the hybrid pool | Simplest selector that improves Recall@50 | Planned |
| M7 — Advanced retrieval | E12–E14 | Test only error-motivated complex methods | Retained improvement with a documented error class | Deferred |
| M8 — Finalization | — | Retrain, validate artifact, document and submit | Valid `answer.csv` and reproducible solution package | Planned |

## Non-negotiable rules

1. `query_id` is an output identifier only; it must never be a model feature or
   an input to hand-written candidate rules.
2. Keep all rows of the same logical query in the same fold. A random row split
   leaks repeated query texts and inflates retrieval metrics.
3. Deduplicate both gold `item_id` values and predicted `item_id` values per
   query before calculating recall.
4. Every submitted ID must be present in `benchmark_items.parquet`; output
   ordering is irrelevant to the official metric, but selection of the final 50
   is not.
5. For a query with a valid `search_category`, every retrieval source searches
   only the partition where `item_category_id == search_category`. If the query
   category is missing or invalid, use the full corpus as a fallback.
6. Change one meaningful factor per experiment. A result without config,
   validation predictions, code version and runtime is not evidence.
7. CSV evaluations are confirmation points, not a hyperparameter search loop.
   We have only seven attempts.

## Experiment tracking contract

### Sources of truth

| Artifact | Purpose |
| --- | --- |
| This file | Roadmap, hypotheses, decisions and current stage |
| ClearML project `avito-retrieval` | Per-run parameters, live logs, metrics, plots and selected artifacts |
| `results/experiment_registry.csv` | Compact local comparison table and backup of key scores |
| `artifacts/<experiment_id>/` | Config, predictions, models/index metadata and validation report |
| Git commit | Exact code state used by a run |

ClearML tracking is development-only. The final inference pipeline must run
locally without any external API. If a remote ClearML server is not configured,
use offline sessions and retain the local artifacts.

### ClearML naming

```text
Project:  avito-retrieval
Task:     E04__bm25__all_fields__s42
Tags:     [stage=lexical, split=v1, corpus=proxy-v1, seed=42]
```

For every measured run log:

- code commit, random seed, Python/package versions and device;
- hashes, row counts and schemas of all input Parquet files;
- text representation, preprocessing and full parameter config;
- macro `Recall@50`, diagnostic recall metrics and candidate-source coverage;
- elapsed time, peak RAM/VRAM and index size;
- validation predictions and the resolved configuration;
- checkpoint/index only when its size is practical.

Do **not** upload raw Parquet data or raw announcement texts to a remote
tracker unless the dataset terms explicitly allow it. Log fingerprints and
aggregate statistics instead.

## Fixed evaluation protocol

### Query group

`query_key` is built from every available search-side field after conservative
normalization:

```text
search_query
search_location_id
search_is_delivery_search
search_infm_params_text
search_category
```

All positive rows with one `query_key` form one multi-label query. This matches
the intended retrieval problem better than treating every clicked pair as an
independent query.

### Validation corpus and splits

The audit determines which of these two protocols is viable; retain the one
that most closely matches inference and document the decision.

1. **Benchmark-aligned proxy.** Keep train positives whose `item_id` appears in
   `benchmark_items`, group-split by `query_key`, and retrieve only from
   `benchmark_items`.
2. **Train-corpus proxy.** Deduplicate train items into a corpus, group-split
   queries, and retrieve held-out query positives from that corpus.

Use a primary frozen validation fold for rapid iteration. Before accepting an
expensive or high-impact change, repeat it on a second group-disjoint fold.
The test benchmark and its hidden labels are never used to tune a method.

### M0 decision: frozen proxy v1

The audit selected the benchmark-aligned proxy. It uses all 189,212 rows of
`benchmark_items.parquet` as the retrieval corpus and only train positives
whose `item_id` is present in that corpus as labels. A logical query is the
normalized tuple of all five `search_*` fields; it is never split between
folds. `GroupShuffleSplit(test_size=0.20, random_state=42)` produces 21,239
train groups and 5,310 validation groups (33,010 surviving positive rows over
26,549 logical queries overall).

Only 5.26% of unique train item IDs occur in the benchmark corpus. Thus this
is a reproducible, corpus-aligned selection proxy rather than an estimate of
the hidden leaderboard score. Every later experiment must report macro
Recall@50 on this frozen fold and meaningful query slices.

### Metrics

| Metric | Role |
| --- | --- |
| Macro Recall@50 | Primary model-selection metric; exact implementation of the official metric |
| Recall@1, @5, @10, @20, @100, @200 | Diagnostics for candidate quality and candidate-budget allocation |
| HitRate@50 | Fraction of queries with at least one hit; diagnostic only |
| Source Recall@50 and union Recall@K | Measures complementarity of BM25, dense, history and other sources |
| Runtime, RAM/VRAM, index size | Reproducibility and feasibility constraints |

`NDCG`, `MRR` and AUC may be logged for diagnosis but do not replace macro
Recall@50: their query weighting and position sensitivity differ from the
official score.

## Milestones and experiment ladder

### M0 — data audit and reproducibility

**Purpose:** understand the real data before choosing models.

Tasks:

1. Validate the three schemas, dtypes, IDs, nulls and UTF-8 text fields.
2. Count unique raw and normalized queries, positives per query, item
   duplicates and text duplicates.
3. Measure train-to-benchmark item overlap and exact/normalized query overlap.
4. Inspect category, location, price, rating and filter distributions.
5. Build a strict `answer.csv` validator.
6. Create and freeze validation fold(s), corpus protocol and data fingerprints.

**Exit criteria:** one command produces the audit report; one unit-tested
function produces macro Recall@50; a dummy answer passes the format validator.

---

### M1 — lexical baseline

#### E01 — plain BM25

**Hypothesis:** lexical overlap is a meaningful baseline for short service
queries.

- Candidate partition: `item_category_id == search_category` (the M0 hard
  constraint); use the full corpus only for a missing or invalid query category.
- Document text: deterministic concatenation of title, item parameters and
  description.
- Query text: `search_query` only in the first pass.
- Output: top 50 and top 200 validation predictions.

**Decision:** establish the initial Recall@K, runtime and error slices. This
experiment is never removed from the comparison table.

#### E02 — query and document text ablations

**Hypothesis:** search filters/categories and structured item parameters add
useful evidence beyond title and description.

Test exactly one representation difference at a time:

- title only;
- title + item parameters;
- title + parameters + description;
- adding each search-side textual field to the query representation.

#### E03 — strong lexical retrieval

**Hypothesis:** a field-aware lexical system improves the plain baseline.

Compare:

- BM25F or equivalent separately weighted fields;
- Russian-safe normalization (case, punctuation and whitespace), with stemming
  or lemmatization only if it helps on validation;
- word BM25 versus character n-gram TF-IDF/cosine;
- union/fusion of complementary lexical lists.

**Exit criteria for M1:** retain the strongest reproducible lexical source and
record its Recall@50, union coverage and resource cost.

---

### M2 — signals from historical positives

#### E04 — query-history retrieval

**Hypothesis:** train interactions provide candidates for repeated or
semantically-near requests without any per-`query_id` hardcoding.

Sources to test separately:

- exact normalized `query_key` lookup;
- relaxed lookup by query text plus category, then query text alone;
- nearest historical query in lexical/dense query space, propagating only its
  clicked items that exist in the inference corpus.

Measure coverage by overlap class: exact-repeat, near-repeat and unseen query.
Do not accept a rule that only improves repeated queries while harming the
overall frozen validation score.

**Exit criteria for M2:** either retain a general historical candidate source
with a documented quota, or explicitly reject it.

---

### M3 — zero-shot semantic retrieval

#### E05 — embedding model benchmark

**Hypothesis:** dense semantic retrieval recovers relevant services that do not
share words with the short query.

Compare candidate bi-encoders on identical texts, corpus and evaluation:

- a multilingual E5 baseline;
- `BAAI/bge-m3` dense representation;
- a practical Qwen3-Embedding size;
- a Russian-focused candidate if it can be reproduced locally.

For each model, record model revision, query instruction, max sequence length,
normalization, embedding dimension, exact/ANN index parameters, Recall@K and
index cost. With only about 189k benchmark items, test exact inner-product
search before accepting an approximate index that can lose recall.

**Exit criteria for M3:** choose at most one or two dense sources that provide
measurable complementary recall relative to M1/M2.

---

### M4 — domain-adapted bi-encoder

#### E06 — positive-pair fine-tuning

**Hypothesis:** the train query–chosen-item pairs adapt a selected bi-encoder
to Russian services better than zero-shot embeddings.

- Build consistent query and item prompts from the selected fields.
- Aggregate all known positives per logical query.
- Start with contrastive/in-batch-negative training using batches without
  duplicate queries or positive items.
- Compare frozen zero-shot checkpoint to the fine-tuned checkpoint.

#### E07 — hard-negative mining

**Hypothesis:** lexically and semantically close non-positive candidates are
more informative than random negatives.

For each train query, mine candidates from BM25 and the current dense model;
exclude all known positives for that query. Mix hard and random negatives and
record the ratio. Because unlabeled items may be plausible services, treat them
as weak negatives and inspect sampled pairs manually only for quality control,
not to recover benchmark labels.

**Exit criteria for M4:** retain domain adaptation only if it improves frozen
Recall@50 and does not collapse unseen-query performance.

---

### M5 — hybrid candidate generation

#### E08 — source coverage and candidate budget

**Hypothesis:** lexical, dense and historical sources miss different positives.

For every retained source, generate a configurable top-K list (initially test
50, 100, 200 and 300). Measure:

- per-source Recall@50 and Recall@K;
- union Recall before truncation;
- marginal recall gained when adding each source;
- duplicate rate and source overlap.

#### E09 — fusion

**Hypothesis:** rank fusion makes a better final 50 than any source alone.

Test in sequence:

1. rank union with a fixed source-priority policy;
2. Reciprocal Rank Fusion with a small, documented grid over `k` and source
   weights;
3. score normalization only if it beats rank-based fusion on frozen validation.

**Exit criteria for M5:** a frozen hybrid candidate pool and fusion baseline
with source quotas justified by measured marginal coverage.

---

### M6 — selecting the final 50

#### E10 — CatBoost ranker

**Hypothesis:** a supervised ranker can choose a better 50 from the hybrid pool
than RRF alone.

- Training group: `query_key`.
- Positives: known selected item IDs.
- Negatives: only retrieved candidates not known positive for that query.
- Candidate features: source flags/ranks/scores, field-level lexical scores,
  dense similarities, token overlap, category compatibility, location features
  and safely parsed filter compatibility.
- Objectives to compare: `PairLogit` and a groupwise ranking objective.

The ranker is evaluated by the project’s own macro Recall@50 implementation;
it is never evaluated only by its training loss or NDCG.

#### E11 — neural reranker

**Hypothesis:** a local multilingual cross-encoder/reranker improves final
selection from a small hybrid pool.

Run only on a bounded top-K pool, compare directly with E10, and retain it only
if its Recall@50 gain justifies inference cost.

**Exit criteria for M6:** choose the simplest selector that gives a confirmed
gain over hybrid RRF.

---

### M7 — advanced methods, only if motivated by errors

#### E12 — BGE-M3 sparse or multi-vector retrieval

Use only if error analysis shows an unresolved lexical/semantic gap after
M1–M6. Compare it as a separate source; do not replace a strong baseline
without an ablation.

#### E13 — late interaction retrieval

Test ColBERT-style retrieval only if its expected Recall gain justifies index
size and complexity.

#### E14 — controlled query expansion

Try deterministic, corpus- or train-derived expansion only after a strong
baseline. Evaluate by query slice because pseudo-relevance expansion can cause
query drift. No external online LLM/API may be required at inference.

**Exit criteria for M7:** retain only a method with a measurable, reproducible
gain and a clear error class it solves.

---

### M8 — final training, submission and report

1. Retrain the selected pipeline on all permitted train data.
2. Generate candidates for every benchmark query.
3. Run the strict `answer.csv` validator: exact columns, all query IDs once,
   0–50 unique item IDs per row, and every ID belongs to benchmark items.
4. Save the final config, model revision/checkpoint hashes, index metadata,
   environment lockfile and inference command.
5. Write the solution description: task formulation, data split, all accepted
   components, open-source models/libraries with versions, local metrics and
   reproducibility instructions.
6. Submit only a version that has passed the artifact-level validation above.

## Online submission budget

The seven allowed attempts are a limited confirmation resource. The tentative
allocation is revised only after local metrics exist.

| Attempt | Intended purpose |
| --- | --- |
| 1 | Format-valid plain BM25 or strongest validated lexical baseline |
| 2 | Strong lexical/historical combination |
| 3 | Best zero-shot hybrid |
| 4 | Fine-tuned dense hybrid |
| 5 | Best selector/fusion variant |
| 6 | One evidence-backed advanced improvement or robust ensemble |
| 7 | Final retrain after all local checks |

Do not submit an unvalidated idea merely to consume an attempt. Record every
submission ID, configuration hash and public score in the registry.

## Required experiment card

Copy this section before starting a measured experiment.

```md
## E## — short experiment name

Status: planned | running | accepted | rejected | blocked
Date:
Owner:
ClearML task:
Git commit:

### Hypothesis

### Single change versus control

### Fixed inputs
- split:
- corpus:
- candidate pool:
- seed:

### Configuration

### Results
| Metric | Control | Experiment | Delta |
| --- | ---: | ---: | ---: |
| Macro Recall@50 | | | |
| Recall@10 | | | |
| Recall@200 / union coverage | | | |
| Runtime | | | |
| Peak RAM/VRAM | | | |

### Error slices
- repeated / unseen query:
- short / long query:
- category:
- location/filter presence:

### Decision and rationale

### Artifacts
- config:
- predictions:
- logs / ClearML task:
```

## Decision log

| Date | Stage | Decision | Evidence | Status |
| --- | --- | --- | --- | --- |
| 2026-09-17 | Planning | Adopt ClearML plus local artifacts and this plan | Approved before data audit | Active |
| 2026-09-17 | M0 | Freeze `benchmark_aligned_proxy_v1`: benchmark corpus, group-disjoint 80/20 split, seed 42, macro Recall@50 | Executed audit notebook; 5,310 validation groups and submission validator smoke test | Active |
| 2026-09-17 | M0 | Make `item_category_id == search_category` a hard retrieval partition; fallback to all items only for an invalid category | 99.9894% category match among 33,010 benchmark-corpus-aligned train positives | Active |
