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

## 3. Seed Invariance & Event Validity

### 3.1 Seed Policy

* All primary claims must be based on **events that appear consistently across seeds**
* Events that do not replicate are downgraded or marked exploratory

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
