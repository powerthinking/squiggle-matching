# Squiggle Matching
### A geometry-first research program for understanding how models learn mathematics: by tracking representation change over time.

This repository is the home of a long-term research effort to study **how transformer models develop mathematical skill** by treating learning as **geometric change** inside representation spaces.

Most interpretability work asks: *“What’s inside the model at the end?”*  
This project asks: *“What changed, when did it change, and what caused it?”*

The central object of study here is the **squiggle**:

> A *squiggle* is a temporally ordered trajectory of geometric state changes in a neural representation space, expressed in a basis-invariant way and compressed into a matchable signature.  
> It represents how representations reorganize over time, not their final state.

And the core method:

> **Squiggle Matching** detects and compares co-similar geometric change patterns across time, layers, models, tasks, or seeds by matching event structure and dynamic profiles (not raw coordinates).  
> It is used both as:
> 1) a signal of important internal learning events (collapse, bifurcation, phase transition), and  
> 2) a correlation tool (correspondence is a special case of correlation).

---

## Why this exists

This work is intentionally optimized for:
- **understanding** how learning happens (and when it *doesn’t* happen),
- building reusable tooling and datasets for others,
- and producing research artifacts that are valuable even if they *don’t* maximize benchmark score.

A core part of the program is producing openly useful training-trajectory datasets and curated math curricula: including work intended for competitions like **Kaggle AIMO / Math Corpus Prize**: but the bigger goal is to build a new lens on learning dynamics that generalizes beyond a single contest.

---
## What this is not

This project does **not** attempt to:
- discover new mathematical theorems,
- replace formal theory with neural analogies,
- or treat internal representations as literal mathematical objects.

All claims are empirical and scoped to measurable changes inside trained models.

---

## The Three-Layer Squiggle Structure

Squiggle analysis is explicitly layered:

### 1) Geometric State Layer
Basis-invariant descriptors at a time slice, e.g.:
- effective rank / singular value spectrum
- subspace alignment metrics
- sparsity / anisotropy measures
- attention/residual distribution descriptors
- local curvature proxies / manifold-ish indicators (where applicable)

### 2) Dynamics Layer
Temporal changes of geometry:
- velocity / acceleration of descriptor changes
- volatility, drift, rotation-like changes
- stability vs churn indicators

### 3) Event / Signature Layer

Compressed, semi-discrete transitions suitable for matching:
- rank collapse → re-expansion
- bifurcation / specialization events
- phase transitions (“operation onset” events)
- regime shifts across tasks/curricula

Events are treated as heuristic markers for candidate learning transitions, not ground-truth semantic boundaries.

The system is built to support **event detection**, **signature extraction**, and **cross-run matching** as first-class concepts.

> **Implementation details:** See [docs/event_detection.md](docs/event_detection.md) for the event detection algorithm, warmup handling, and retention metrics.

---

## Two-model setup: Scout vs Research

This project uses two distinct model tracks:

### Scout Model (make it work)
A smaller, lower-cost model focused on:
- verifying training pipelines and data correctness
- validating logging, probes, and derived metrics
- testing event detection + squiggle signature extraction end-to-end
- running lots of cheap A/B perturbations

Scout is where we find bugs, confirm the instrumentation, and prove the pipeline.

### Research Model (make it real)
A larger but still tractable model focused on:
- deeper curricula and longer training trajectories
- controlled experiments at scale (seeds, curricula variants, tokenization variants, ordering variants)
- high-resolution logging for squiggle analysis
- producing “publication-grade” datasets + writeups

---

## Data strategy: “Over-collect, then compress into cubes”
Raw tensors are not the product. They’re the input.

The guiding rule is:
> **Use the lake once to build reusable cubes.**

The system logs enough to support geometry-first analysis (selected layers/steps/seeds), then compiles:
- tiny scalar tables (loss, accuracy, grads, etc.)
- medium derived geometry tables (per-layer descriptors over time)
- event tables (detected transitions)
- signature tables (matchable representations of events/dynamics)

This is designed to keep the *analysis* lightweight even when the *capture* is rich.

---

## AIMO / Math Corpus Prize: proof families as a geometry probe
For the Math Corpus Prize, the strategy is intentionally research-driven:

- We curate **problem families** (especially “proof families” and structured variations)
- We train (often on **R1-Zero style models**) to reduce contamination from pre-trained “general reasoning” patterns
- We evaluate not only performance but the **geometric impact** of each family:
  - Which families cause large internal reorganizations?
  - Which ones do almost nothing?
  - Which ones produce consistent event signatures across seeds?

A “no change” result is still a result: it tells us that the family did not induce representational reorganization under those conditions.

Where appropriate, we also compare against publicly released datasets from other competitors and identify families that appear to generate similar (or contrasting) squiggle behaviors: not as a takedown, but as a way to map the space of “what induces learning events.”

---

## Sub-Repos

### [`squiggle-core`](https://github.com/powerthinking/squiggle-core)
The reusable library
- geometry descriptor extraction
- event detection algorithms
- signature formats and matching logic
- schemas (Parquet table specs), validation, versioning
- evaluation utilities for “match quality”

### [`squiggle-experiments`](https://github.com/powerthinking/squiggle-experiments)
Experiment configs and runners:
- curriculum variants
- tokenization/order perturbations
- logging resolution settings
- seed matrices

### [`squiggle-instrumentation`](https://github.com/powerthinking/squiggle-instrumentation)
Reusable logging + probe harnesses:
- hooks for embeddings/residual streams/selected activations
- optional attention logging (kept selective for cost)
- checkpointing strategy
- derived metric compilation

### [`squiggle-analysis`](https://github.com/powerthinking/squiggle-analysis)
Squiggle pipeline:
- geometry descriptor extraction
- dynamics + event detection
- signature extraction
- matching + correlation tooling
- reporting

### [`squiggle-datasets`](https://github.com/powerthinking/squiggle-datasets)
Dataset build scripts + manifests:
- proof-family generators
- train/val/test split logic
- dataset cards and licensing
- dataset versioning and reproducibility

---

## Principles (the “don’t make me regret this later” section)

- **Basis-invariance first:** avoid coordinate-level artifacts as much as possible.
- **Multiple seeds always:** if a squiggle doesn’t survive seeds, it’s probably a mirage.
- **Instrumentation should be boring:** failures should be easy to detect and explain.
- **Releases matter:** datasets, configs, and summaries are first-class outputs.
- **No heroics:** prefer repeatable pipelines over clever one-offs.

---

## Documentation

Detailed specifications in `docs/`:

- [event_detection.md](docs/event_detection.md) — Event detection algorithm, warmup handling, retention metrics, LLM analysis
- [squiggle_architecture.md](docs/squiggle_architecture.md) — Frozen ontology vs evolving measurement
- [probe_harness_design.md](docs/probe_harness_design.md) — Probe protocol, DIS scoring
- [artifacts.md](docs/artifacts.md) — Directory layout and artifact paths
- [parquet_schemas.md](docs/parquet_schemas.md) — Schema definitions

---

## Current status

Active development with current focus on:
1) Finalizing Scout pipeline end-to-end
2) Validating descriptor and event detection on controlled curricula
3) Scaling into Research runs and preparing the first trajectory dataset + writeup
4) Packaging Math Corpus Prize datasets with accompanying analysis

---

## Contact / collaboration
If you’re working on similar “learning dynamics” or training-trajectory interpretability, I’d love to compare notes, methods, and results.

