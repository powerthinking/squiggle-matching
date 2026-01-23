# Parquet Schemas (v2)

This document defines the **Parquet dataset schemas** for the Squiggle Matching project.

These tables support:
- **Immutable run artifacts** (facts)
- **Versioned analysis artifacts** (interpretations)
- **Probe summaries** for ranking/selection (e.g., Math Corpus pipeline)

Primary layers:
- **State (Layer A)**: basis-invariant geometric descriptors
- **Dynamics (Layer B)**: temporal change descriptors
- **Event candidates (Layer C)**: derived discrete transitions (optional / may be weak early)
- **Signatures**: compressed matchable squiggle representations
- **Matches**: correspondence results between signatures

Recommended Parquet compression: **zstd**.

---

## Key design conventions

### Common spine keys (appear in most datasets)
- `run_id` (string): stable ID for a training run (immutable facts)
- `analysis_id` (string, nullable): ID for a versioned analysis output (interpretations)
- `schema_version` (string): table schema version identifier
- `created_at_utc` (timestamp[us, tz=UTC]): creation time for this artifact

### Optional spine keys (by table)
- `step` (int64): optimizer step
- `seed` (int32): run seed
- `sample_id` (string, nullable): stable eval sample ID
- `sample_set` (string, nullable): fixed probe set name (e.g., `fixed_v1`)
- `split` (string, nullable): `train` | `val` | `probe`
- `layer` (int16, nullable): transformer layer index
- `module` (string, nullable): canonical module location
- `family_id` (string, nullable): probe-family identifier (Math Corpus selection unit)
- `probe_name` (string, nullable)
- `probe_config_hash` (string, nullable)

### Partitioning guidance
Avoid partitioning by `step`. Prefer:
- Always: `run_id`
- Often: `analysis_id` (for interpretation tables), `probe_name`, `sample_set`
- Optional: `module`, sometimes `event_type` or `metric_name`
- Optional: `layer` only if low-cardinality and helpful

---

## Dataset index

### 00 — `runs/` (run metadata, immutable)
Run configuration + environment + model info.

### 01 — `checkpoints/` (checkpoint index, immutable)
One row per checkpoint or logged step summary.

### 02 — `samples/` (eval sample catalog, immutable)
Stable sample IDs and optional payloads.

### 03 — `families/` (problem-family catalog, immutable)
Stable family IDs (for Math Corpus selection / probe grouping).

### 10 — `metrics_scalar/` (scalar metrics, immutable facts)
Generic time-series metrics in long form.

---

## Geometry and dynamics (analysis artifacts)

### 20 — `geometry_state/` (Layer A, interpretation)
Descriptors for `(run_id, analysis_id, step, sample_id, layer, module)`.

### 21 — `geometry_dynamics/` (Layer B, interpretation)
Deltas/drift for same keys as geometry_state.

### 22 — `events_candidates/` (Layer C candidates, interpretation)
Candidate event windows detected from `geometry_state`.

Notes:
- `events_candidates` are per-run candidates (not cross-seed consensus).
- Windows are represented by `start_step` and `end_step`.
- Supports both:
  - `event_type = change_point` (single-metric)
  - `event_type = change_point_composite` (multi-metric; `metric = __composite__`)

---

## Probe outputs (facts + interpretation)

### 12 — `probe_captures_index/` (probe capture inventory, immutable facts)
Index of probe capture shards/files produced by a run.

### 13 — `probe_summaries/` (probe_summary@2.x, interpretation)
One row per `(run_id, family_id, seed, probe_name)` summarizing A/B (+ optional C) and ranking.

### 14 — `probe_events_candidates/` (Layer C candidates, interpretation)
Optional event-candidate rows derived from probe summaries or from geometry tables.

---

## Events/signatures/matching (analysis artifacts)

### 30 — `events/` (events, interpretation)
Consensus events (typically across seeds/tests). May reference source candidates.

### 31 — `signatures/` (signatures, interpretation)
Compressed vectors + event sequences.

### 32 — `matches/` (match results, interpretation)
Correspondence / similarity results across signatures.

---

## Optional heavy datasets (use sparingly)
### 40 — `embeddings/`
Selected raw vectors only.
### 41 — `attention_summary/`
Head-level summaries only.

---

## Canonical enums (recommended)
- `split`: `train`, `val`, `probe`
- `module`: `resid_pre`, `attn_out`, `mlp_out`, `resid_post`, `logits`
- `run_mode`: `probe_micro_finetune`, `train_full`, `eval_only`

---

# Table Schemas

## 03 — `families/`
Purpose: stable registry of probe grouping units.

Columns:
- `family_id` string (PK)
- `family_version` string
- `source` string (e.g., dataset name, generator)
- `tags` list<string>
- `notes` string (nullable)
- `created_at_utc` timestamp[us, tz=UTC]

---

## 12 — `probe_captures_index/` (facts)
Purpose: inventory of immutable probe capture files.

Columns:
- `run_id` string
- `probe_name` string
- `probe_config_hash` string
- `schema_version` string
- `capture_type` string (e.g., `activations_sketch`, `attn_summary`, `grad_norms`)
- `step` int64 (nullable)
- `shard_id` string
- `path` string
- `bytes` int64
- `checksum` string (nullable)
- `created_at_utc` timestamp[us, tz=UTC]

Primary key (recommended):
- (`run_id`, `probe_name`, `probe_config_hash`, `shard_id`)

---

## 13 — `probe_summaries/` (probe_summary@2.x)
Purpose: indexable summaries used for ranking/selection.

Columns:
- `run_id` string
- `family_id` string
- `seed` int32
- `probe_name` string
- `probe_config_hash` string
- `schema_version` string
- `model_id` string
- `base_checkpoint` string
- `run_mode` string
- `steps` int32
- `capture_steps_used` list<int64>

Layer A (arrays aligned with `layers_covered`):
- `layers_covered` list<int16>
- `A_eff_rank_pre` list<float32>
- `A_eff_rank_post` list<float32>
- `A_eff_rank_delta` list<float32>
- `A_sv_entropy_pre` list<float32>
- `A_sv_entropy_post` list<float32>
- `A_sv_entropy_delta` list<float32>
- `A_sparsity_pre` list<float32>
- `A_sparsity_post` list<float32>
- `A_sparsity_delta` list<float32>
- `A_principal_angle_post_vs_pre` list<float32>  (or similarity)

Layer B (arrays aligned with `layers_covered` or derived `affected_layers`):
- `B_drift_velocity_by_layer` list<float32>
- `B_drift_accel_by_layer` list<float32>
- `B_volatility_by_layer` list<float32>
- `B_alignment_velocity_by_layer` list<float32>
- `B_drift_velocity_global` float32
- `B_drift_accel_global` float32
- `B_volatility_global` float32
- `B_alignment_velocity_global` float32
- `affected_layers` list<int16>

Signature + ranking:
- `signature_version` string
- `signature_vector` list<float32>
- `signature_norm` float32
- `score_version` string
- `magnitude` float32
- `coherence` float32
- `novelty` float32
- `DIS` float32

Traceability:
- `captures_manifest_path` string
- `captures_index_path` string
- `created_at_utc` timestamp[us, tz=UTC]

Primary key (recommended):
- (`run_id`, `family_id`, `seed`, `probe_name`, `probe_config_hash`)

---

## 14 — `probe_events_candidates/` (optional)
Purpose: derived Layer C candidate events (not consensus, not matching).

Columns:
- `run_id` string
- `family_id` string
- `seed` int32
- `probe_name` string
- `probe_config_hash` string
- `schema_version` string
- `detector_version` string
- `event_id` string
- `event_type` string
- `t_step` int64
- `layers` list<int16>
- `strength` float32
- `supporting_key` list<string> (optional)
- `supporting_val` list<float32> (optional)
- `timewarp_tolerance_steps` int32
- `created_at_utc` timestamp[us, tz=UTC]

Primary key (recommended):
- (`run_id`, `family_id`, `seed`, `probe_name`, `probe_config_hash`, `event_id`)

---

## 22 — `events_candidates/` (analysis)
Purpose: per-run candidate event windows detected from geometric time series.

Columns:
- `run_id` string
- `analysis_id` string
- `schema_version` string
- `created_at_utc` timestamp[us, tz=UTC]
- `event_id` string
- `layer` int16
- `metric` string
- `step` int64
- `start_step` int64 (nullable)
- `end_step` int64 (nullable)
- `event_type` string
- `score` float32

Scoring breakdown (nullable):
- `magnitude` float32
- `structure_modifier` float32
- `magnitude_eff` float32
- `coherence` float32
- `novelty` float32

Single-metric diagnostics (nullable; populated for `event_type = change_point`):
- `metric_size` float32
- `metric_z` float32
- `baseline_median` float32
- `baseline_mad` float32
- `volatility_event` float32
- `volatility_baseline` float32
- `volatility_ratio` float32
- `volatility_ratio_agg` float32

Composite per-metric diagnostics (nullable JSON strings; populated for `event_type = change_point_composite`):
- `metric_sizes_json` string
- `metric_z_json` string
- `baseline_median_json` string
- `baseline_mad_json` string
- `volatility_event_json` string
- `volatility_baseline_json` string
- `volatility_ratio_json` string

Primary key (recommended):
- (`run_id`, `analysis_id`, `event_id`)
