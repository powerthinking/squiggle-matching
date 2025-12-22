# Squiggle Matching — PLAN (Epic-level)

## Purpose of this document

This `PLAN.md` is the structural guide for building out the Squiggle Matching research program.

It is written at **Epic-level** (outcomes, deliverables, contracts, and dependencies). We will later drill down into Task-level plans per repo.

## North Star / Research Thesis

Treat learning as **geometric change over time** inside representation spaces.

Primary object:

- A **squiggle**: a temporally ordered trajectory of basis-invariant geometric state changes, compressed into a matchable signature.

Primary method:

- **Squiggle Matching**: detect and compare co-similar patterns of geometric change across time/layers/models/tasks/seeds by matching event structure + dynamic profiles.

Key outputs (first-class artifacts):

- Training-trajectory datasets (“lake once → cubes forever”)
- Reusable tooling (schemas, validators, CLIs)
- Reports + summaries that remain useful even for null results

## Workspace / Repo map (what lives where)

Main repo root: `squiggle-matching/`

- `squiggle-matching/`
  - Research program overview, repo roles, and documentation for data contracts
  - Canonical guideposts: `docs/parquet_schemas.md`, `docs/artifacts.md`

Supporting repos (in this workspace as sibling folders):

- `squiggle-core/`
  - Canonical paths and dataset schemas/validators
  - Stable interface between training ↔ analysis ↔ reporting

- `squiggle-instrumentation/`
  - Capture hooks / logging harnesses (currently minimal in codebase)

- `squiggle-experiments/`
  - Experiment runners/configs (Scout + future Research runs)
  - Produces run artifacts (scalars, captures/samples, meta)

- `squiggle-analysis/`
  - Post-training analysis pipeline
  - Consumes captured tensors and writes geometry/events/report artifacts

- `squiggle-datasets/`
  - Dataset build scripts/manifests (planned)

- `aimo-math-corpus-proof-families/`
  - Public release repo for proof-family datasets (Math Corpus Prize focus)

## System-level data model (Three-layer squiggle structure)

- **Layer 1 — Geometric State**
  - Basis-invariant descriptors at a time slice (rank/spectrum, anisotropy, alignment, etc.)

- **Layer 2 — Dynamics**
  - Temporal change of geometry (drift/velocity/volatility, rotation-like change, stability vs churn)

- **Layer 3 — Events / Signatures**
  - Semi-discrete transitions suitable for matching
  - Event tables + signature tables + match tables

## Canonical artifacts (contract)

This project depends on *strictly defined* artifacts and schemas.

High-level artifact families:

- **Run metadata** (`meta.json`)
- **Scalar metrics** (loss, lr, grad_norm, probe metrics)
- **Captured tensors** (selected steps/layers/modules; not “log everything”)
- **Derived tables**
  - `geometry_state`
  - `geometry_dynamics`
  - `events`
  - `signatures`
  - `matches`
- **Reports** (`report.md`, plots)

## Open Questions / Doc inconsistencies to resolve (blocking contracts)

These were the inconsistencies I found between the READMEs/docs and the current code. The following decisions are now locked as the contract for Epic 0.

### Locked decisions (contract)

- **Canonical storage layout**: hybrid
  - Run-scoped artifacts: `data/runs/runs/<run_id>/...`
  - Dataset/cube Parquets: `data/runs/<dataset>/<run_id>.parquet`
- **Canonical capture folder name**: `captures/`
- **Geometry table format**: both
  - Long-form (iteration/debug): `geometry_state_long/` (and optionally `geometry_dynamics_long/`)
  - Wide-form (release/strict): `geometry_state/` (and `geometry_dynamics/`) following the richer schema contracts
  - Implementation detail: long vs wide are stored in **separate folders** (not filename suffixes)
- **Events interval column names**: `start_step` / `end_step` (with optional canonical `step` for convenience)
- **Scalar metrics storage**: both
  - Wide-form (convenient): `metrics_wide/`
  - Long-form (canonical contract): `metrics_scalar/` with `metric_name`/`value`
  - Implementation detail: wide vs long are stored in **separate folders** (not filename suffixes)
- **DDAR**: “Deductive Database + Algebraic Reasoning” — leveraging an outside logic system (AlphaGeometry-inspired) to generate or support reasoning traces and data distributions

## Roadmap (Epics)

### Epic 0 — Lock the data contract (schemas + paths)

Goal:

- Establish a single, unambiguous on-disk layout and schema policy so all repos agree.

Scope:

- Canonicalize run root, capture folder naming, and table formats
- Align validators to the chosen schema policy

Deliverables:

- An authoritative artifact-layout spec using the hybrid layout:
  - `data/runs/runs/<run_id>/...` for run-scoped artifacts
  - `data/runs/<dataset>/<run_id>.parquet` for dataset/cube Parquets
- A single authoritative schema spec per dataset, with explicit v0/v1 boundaries
- A deliberate “dual format” strategy where needed:
  - Geometry: long-form for iteration + wide-form for release/strictness
  - Scalars: wide-form for convenience + long-form as canonical
- Versioning strategy (how schema changes are introduced)

Definition of done:

- A Scout run + analysis run produces artifacts matching the hybrid contract
- Run-scoped artifacts land under `data/runs/runs/<run_id>/...`
- Capture tensors land under `data/runs/runs/<run_id>/captures/step_*/...`
- Both long + wide geometry artifacts can be produced (at least for the MVP metric set)
- Scalars are exported to long-form `metrics_scalar/` even if training logs wide-form
- Validators pass (and fail loudly on drift)

Dependencies:

- None (contract decisions are locked)

---

### Epic 1 — Scout pipeline MVP (end-to-end, boring, reliable)

Goal:

- A small-cost training loop that consistently produces:
  - scalars
  - deterministic capture slices
  - metadata
  - “first useful” geometry/events/report outputs

Scope:

- Keep model small and fast
- Focus on correctness, determinism, and failure visibility

Deliverables:

- `squiggle-experiments`: Scout runner with config(s)
- `squiggle-analysis`: one-command analysis
- A small “golden run” fixture (known run_id) for regression checking

Definition of done:

- `run_and_analyze` style flow works on a clean machine with documented steps
- Outputs are diffable and reproducible (seeded)

Dependencies:

- Epic 0 contract decisions

---

### Epic 2 — Geometry descriptors v1 (basis-invariance-first)

Goal:

- Expand geometry extraction beyond v0 rank/topk-mass into a coherent v1 set.

Scope:

- Define “module” capture points (embeddings/resid_pre/resid_post/etc.)
- Implement and validate a small curated metric set

Deliverables:

- `squiggle-core`: geometry metric primitives
- `squiggle-analysis`: geometry computation producing a stable table

Definition of done:

- Geometry outputs are stable across reruns
- Metric definitions are documented and unit-tested

---

### Epic 3 — Dynamics layer v1 (drift/velocity/volatility)

Goal:

- Compute temporal deltas and drift features suitable for event detection and signatures.

Deliverables:

- `geometry_dynamics` derived dataset
- Basic diagnostics (e.g., drift vs loss improvements)

Definition of done:

- Dynamics table is generated for a run and validated

---

### Epic 4 — Event detection v1 (heuristic but falsifiable)

Goal:

- Detect candidate learning transitions with simple, explainable detectors.

Scope:

- Change-point detectors on selected metrics
- Layer-aware thresholds and score semantics

Deliverables:

- `events` dataset with stable schema
- Event detector versioning (`detector_version` or equivalent)

Definition of done:

- Events are produced for baseline runs
- Null results are handled cleanly (empty events table with correct schema)

---

### Epic 5 — Signature extraction v1 (matchable squiggles)

Goal:

- Compress state/dynamics/events into matchable signature objects.

Deliverables:

- `signatures` dataset
- Signature hashing/versioning
- Reference signature builder for baseline curricula

Definition of done:

- Signatures are produced deterministically for the same run

---

### Epic 6 — Matching + correspondence (prototype → robust)

Goal:

- Implement squiggle matching across runs (seed variations, curriculum variants).

Deliverables:

- `matches` dataset
- Match quality evaluation utilities (precision/recall-like diagnostics)

Definition of done:

- Can recover correspondences between baseline and near-baseline runs

---

### Epic 7 — Experiment suite (Paths 0–9)

Goal:

- Execute the controlled A/B experiment paths from `squiggle-experiments/README.md`.

Scope:

- Start with Path 0 + 1–2 high-value A/B paths
- Expand to tokenization/curriculum/proof-burst experiments

Deliverables:

- Versioned experiment configs
- Run matrix definitions (seeds, variants)
- A small “results index” table (what was run, where artifacts live)

Definition of done:

- For each path: baseline + variant runs exist and are analyzable end-to-end

---

### Epic 8 — Dataset releases (AIMO / Math Corpus Prize + general)

Goal:

- Produce reusable datasets and documentation that remain valuable outside this repo.

Deliverables:

- Proof-family generators + manifests
- Dataset cards/licensing
- Reproducible splits and versioning

Definition of done:

- A release artifact is consumable by a third party with minimal setup

---

### Epic 9 — Reporting + review workflow (diffable, durable)

Goal:

- Make analysis outputs easy to review, compare, and share.

Deliverables:

- Report templates
- Plot conventions
- “run comparison” reports (baseline vs variant)

Definition of done:

- A reviewer can answer: what changed, when, and how confidently?

## Immediate next step

- Resolve the Open Questions (Epic 0). Once the contract is locked, we can create task-level plans per repo without rework.
