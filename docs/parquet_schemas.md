# Parquet Schemas (Starter)

This document defines the **starter Parquet dataset schemas** for the Squiggle Matching project.  
These tables support a full training-trajectory workflow with a focus on:

- **State**: basis-invariant geometric descriptors of representations
- **Dynamics**: temporal deltas and drift between checkpoints
- **Events**: detected transitions (collapse, re-expansion, bifurcation, etc.)
- **Signatures**: compressed, matchable squiggle representations
- **Matches**: correspondence/cosimilarity results between signatures

## Key design conventions

### Common spine keys (appear in most datasets)
- `run_id` (string): stable ID for a training run
- `step` (int64): optimizer step (global)
- `sample_id` (string): stable ID for an eval sample (nullable in “global” events)
- `sample_set` (string): fixed probe set name (e.g., `fixed_v1`)
- `split` (string): `train` | `val` | `probe`
- `layer` (int16): transformer layer index
- `module` (string): canonical module location (e.g., `resid_pre`, `attn_out`, `mlp_out`, `resid_post`, `logits`)

### Partitioning guidance (recommended)
Avoid partitioning by `step` (too many partitions). Prefer:
- Always: `run_id`
- Often: `sample_set`, `module`, sometimes `event_type` or `metric_name`
- Optional: `layer` (only if it stays low-cardinality and helps filtering)

Recommended Parquet compression: **zstd**.

---

## Dataset index

### 00 — `runs/` (run metadata)
Stores run configuration, environment, and model info.

### 01 — `checkpoints/` (checkpoint index)
One row per checkpoint or logged step summary.

### 02 — `samples/` (eval sample catalog)
Stable sample IDs and (optional) text/token payloads.

### 10 — `metrics_scalar/` (scalar metrics, long-form)
Generic time-series metrics (`loss`, `grad_norm`, etc.) in long form.

### 20 — `geometry_state/` (Geometric State Layer)
Compact descriptors for each `(run_id, step, sample_id, layer, module)`.

### 21 — `geometry_dynamics/` (Dynamics Layer)
Derived deltas/drift between steps for the same keys.

### 30 — `events/` (Event / Signature Layer — events)
Discrete or semi-discrete detected events from state/dynamics.

### 31 — `signatures/` (Event / Signature Layer — squiggle signatures)
Compressed matchable representations (vectors + event sequences).

### 32 — `matches/` (signature match results)
Stores correspondence / similarity results across signatures.

---

## Optional heavy datasets (use sparingly)

### 40 — `embeddings/`
Selected raw vectors only (probe sets / key steps). Avoid logging everything.

### 41 — `attention_summary/`
Head-level summaries (entropy, top-k mass, diagonal/band mass), not full matrices.

---

## Notes on nullable columns
Some datasets allow “global” objects:
- `events.sample_id`, `events.layer`, `events.module` may be null for run-level events
- `signatures.layer`, `signatures.module`, `signatures.sample_id` may be null for aggregate signatures

---

## Canonical enums (recommended)
- `split`: `train`, `val`, `probe`
- `module`: `resid_pre`, `attn_out`, `mlp_out`, `resid_post`, `logits`
- `event_type`: `rank_collapse`, `reexpansion`, `bifurcation`, `phase_shift`, `stability_plateau` (extend over time)

---

## Practical MVP subset
If you want the smallest useful slice that still supports Squiggle Matching end-to-end:
- `runs/`, `checkpoints/`, `samples/`
- `metrics_scalar/`
- `geometry_state/`, `geometry_dynamics/`
- `events/`, `signatures/`, `matches/`

Then optionally add `embeddings/` and `attention_summary/` once detectors are validated.
