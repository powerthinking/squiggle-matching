# Squiggle Matching : Experiment Design & Execution Contract

This document is the **authoritative design and strategy handoff** for implementing, refactoring, and extending the Squiggle Matching system.

It is intended to be used by:

* a developer (human or LLM) working inside the codebase
* as the replacement for any prior PLAN.md or informal guidance

The focus is **operational clarity, scientific rigor, and long-term extensibility**, not narrative history.

---

## 1. Core Principles (Non‑Negotiable)

1. **Runs are immutable facts. Analyses are evolving interpretations.**
2. **All core artifacts are immutable once created.**
3. **Curriculum is a first‑class experimental variable, not a hyperparameter.**
4. **Events must be seed‑invariant (or explicitly downgraded).**
5. **Instrumentation, probing, and metrics are configurable : never experiment‑local code.**
6. **Everything required to reproduce a run must be captured in machine‑readable manifests.**

---

## 2. Artifact Taxonomy & Immutability Rules

### 2.1 Dataset (Immutable)

A dataset is a collection of training items with stable identifiers.

**Properties**:

* Immutable content (items never change in place)
* Each item has a stable `item_id`
* Dataset has a `dataset_id` derived from a canonical manifest hash

A dataset may be reused across many curricula and experiments.

---

### 2.2 Curriculum Template (Reusable Policy)

A curriculum template defines **how** a dataset is structured, not the final ordering.

Examples:

* ordering policy (low→high, high→low, interleaved)
* stratification rules
* noise injection policy
* pacing rules

Curriculum templates are reusable and parameterized.

---

### 2.3 Curriculum Instance (Immutable)

A curriculum instance is the **fully realized structure** used by training.

**Properties**:

* Concrete ordered sequence (or deterministically defined schedule) of `item_id`s
* Derived from a dataset + curriculum template
* Has a stable `curriculum_id` (hash of manifest)
* Immutable once created

**Key rule**:

* Same dataset + different ordering ⇒ different curriculum instance
* Same ordering logic + different dataset ⇒ different curriculum instance

---

### 2.4 Run (Immutable)

A run is a single executed training job.

Defined by:

* dataset_id
* curriculum_id
* model config
* training config
* instrumentation config
* probe config
* metrics config
* controller config(s)
* seed
* code versions

Each run produces immutable artifacts and a `meta.json` manifest.

### 2.4.1 Probe (Config + Capture Stream)

A **probe** is a configured mechanism for capturing **primitive observations** from the model during a run, in support of downstream geometry/state/dynamics/event analysis.

A probe is defined by **probe_config** (immutable per run) and produces a **probe capture stream** (immutable artifacts).

**Probe scope** (non-negotiable):
- Probes capture **primitive observations only** (e.g., activations, attention summaries, gradient norms, representation sketches).
- Probes do **not** perform event detection, matching, or scoring online.
- Probes may compute lightweight, local reductions needed to make capture feasible (e.g., random projections, layer sampling, chunked stats), but these reductions must be:
  - explicitly specified in probe_config
  - versioned
  - reproducible

**Probe ≠ Analysis**:
- Probe outputs are **raw-ish evidence**.
- Geometry/state computations, event detection, matching, and scoring are **post-run analysis** consumers of probe artifacts.

**Immutability rule**:
- Probe artifacts are immutable once created.
- If a probe definition changes, it must produce a new run_id (via probe_config hash).

---

### 2.5 Test (Replicate Set)

A test is a **collection of runs** that are identical except for seed.

Purpose:

* Measure seed stability
* Enable consensus‑based event detection

---

### 2.6 Variant

A variant is a **single controlled change** that does **not** redefine the curriculum instance.

Allowed as variants:

* model architecture changes
* optimizer / hyperparameters
* probe schedules
* instrumentation levels

Not allowed as variants:

* any change to curriculum definition or realized ordering

---

### 2.7 Experiment

An experiment is a structured comparison of **two or more Tests**.

Typical uses:

* A/B comparisons
* What‑if analyses
* Controlled sweeps

**Rule**:

* Any change to curriculum ⇒ new Test (not a buried variant)

---
## 2.8 Probe Artifacts (Immutable)

Each run that enables probes must produce probe artifacts in a standardized location, e.g.:

```
runs/<run_id>/
meta.json
logs/
probes/
<probe_name>/
manifest.json
captures/
step_000100.*
step_000200.*
index.*
stats.json
```

**Probe manifest must include**:
- probe_config hash
- schema_version
- code commit
- capture schedule actually executed
- any dropped samples and why (OOM, truncation, etc.)
- file inventory + checksums (optional but preferred)

**Storage rule**:
Probe artifacts may be sharded/chunked, but must be reconstructable deterministically via index + manifest.

---
## 3. Seed Invariance & Event Validity

### 3.1 Seed Policy

* All primary claims must be based on **events that appear consistently across seeds**
* Events that do not replicate are downgraded or marked exploratory
* Event consensus operates over post-run derived Layer C events and their supporting Layer A/B dynamics; probes only supply the primitive evidence used to compute these layers.


### 3.2 Event Consensus Requirements

Event detection must compute:

* replicate support (% of seeds)
* temporal alignment tolerance (events may shift in time)
* shape similarity (squiggle signature similarity)

Derived outputs:

* consensus events
* confidence scores
* seed‑outlier flags

---

## 4. Training‑Time Decisions (Online Logic)

Online decisions are allowed but must be **explicit, versioned, and auditable**.

Examples:

* early stopping
* dynamic probe scheduling
* adaptive logging density

### 4.1 Design Rule

Training code must log **primitive observations only**.

All derived logic must operate via a **Feature Window API**, e.g.:

* last value
* rolling mean / slope
* windowed z‑scores
* event‑triggered features

Controller decisions must be logged as their own event stream.

---
## 4.2 Probes vs Controllers (Online Boundary)

Probes and controllers may both run during training, but they have different contracts:

### Probes (Capture)
- Output: primitive observation streams
- Allowed: scheduled sampling, compression, selection of layers/heads/tokens
- Not allowed: declaring events, declaring matches, declaring causal impact
- Must log: what was captured, when, and with what reduction settings

### Controllers (Decision-making)
- Output: decision events + rationale features
- May use: Feature Window API over *already-logged primitives*
- Must log: inputs, computed features, decision, version, thresholds

**Design rule**:
Controllers may reference probe-derived primitives only through the Feature Window API, never through ad-hoc logic in training code.

---

## 5. Data Pipeline (Conceptual)

1. Dataset generation / import
2. Curriculum instance generation
3. Training run execution
4. Primitive logging (scalars, probes, captures)
5. Post‑run calculations / aggregations
6. Geometry state computation
7. Event detection
8. Reporting and export

Training artifacts are never overwritten.
Analysis artifacts are versioned and additive.

---
## 5.1 Probe & Analysis Split (Pipeline Contract)

The pipeline is intentionally split into:

### Training-time (Immutable evidence)
1. Dataset generation / import
2. Curriculum instance generation
3. Training run execution
4. Primitive logging (scalars)
5. **Probe captures** (configured observation streams)
6. Controller event stream (if any)

### Post-run (Versioned interpretation)
7. Post-run calculations / aggregations
8. Geometry state computation (Layer A)
9. Dynamics computation (Layer B)
10. Event extraction / encoding (Layer C)
11. Consensus (seed invariance)
12. Matching (event topology + dynamic profile + invariance class)
13. Scoring / ranking (Magnitude × Coherence × Novelty)
14. Reporting and export

**Rule**:
- Training-time produces facts.
- Post-run produces interpretations.
- Interpretations are versioned and additive.

---

## 6. Experiment Structure & Documentation

Each experiment is a durable research record.

### 6.1 Required Files

```
experiments/<EXP-ID>/
  README.md
  spec/
    experiment.yaml
    A_test.yaml
    B_test.yaml
  outputs/
    compare.md
```

### 6.2 Experiment README Must Contain

* hypothesis
* invariants
* differences between arms
* seed plan
* primary outcomes
* event‑consensus rules
* produced run_ids
* links to reports

This README is both:

* external documentation
* future self‑memory

---

## 7. Run Identity & Versioning

### 7.1 Canonical Run Spec

A run ID must be derived from a canonical JSON representation of:

* dataset_id
* curriculum_id
* all configs (hashed)
* seed
* code commit(s)

`run_id = hash(canonical_run_spec)`

### 7.2 Artifact Versioning

Each artifact records:

* semantic version (where applicable)
* schema version
* git commit

Breaking schema changes increment MAJOR versions.

---
## 7.3 Probe Config Canonicalization (Required)

A run’s identity must include a canonical representation (and hash) of `probe_config`, including at minimum:

- target modules (layers, blocks, heads)
- capture types (activations, attention summaries, gradients, etc.)
- sampling schedule (steps / intervals / triggers)
- token/sequence sampling rules (which tokens, how many)
- reductions/compressions (projection dims, PCA/RSVD params, sketch type)
- precision/dtypes and device placement (fp16/bf16, cpu/gpu)
- schema_version + probe_version
- random seeds used *inside the probe* (if any)

**Rule**:
Any change to probe_config ⇒ new run_id.

--- 

## 8. Repository Responsibilities

* **squiggle‑core**: schemas, run layout, validation
* **squiggle‑experiments**: declarative experiment definitions
* **squiggle‑instrumentation**: probes, hooks, logging profiles
* **squiggle‑analysis**: geometry, events, reports
* **squiggle‑datasets**: dataset generation and manifests

No experiment should require bespoke code changes in other repos.

---

## 9. Non‑Goals (Explicitly Out of Scope)

* leaderboard optimization
* adaptive logic without audit trails
* silent schema drift
* single‑seed claims
* experiment‑local instrumentation hacks

---

## 10. Intended Use of This Document

This document is the **single source of truth** for:

* refactoring existing code
* adding new experiments
* onboarding collaborators (human or LLM)
* preventing architectural drift

Any future change to experimental philosophy must update this document.
