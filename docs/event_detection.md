# Event Detection

This document describes the event detection system in squiggle-analysis, which identifies significant geometric changes during training.

## Overview

Event detection transforms continuous geometry trajectories into discrete, interpretable change events. The system uses adaptive thresholding, peak selection with suppression, and warmup-aware budgeting to produce meaningful events.

## Pipeline

```
Geometry State → Delta Computation → Adaptive Thresholding → Peak Selection → Composite Formation → Events
```

### 1. Delta Computation

For each (layer, metric) series, compute step-over-step deltas:

```
delta[i] = |value[i+1] - value[i]|
```

This captures the magnitude of change between consecutive capture points.

### 2. Adaptive Thresholding

Rather than fixed thresholds, the system computes per-series thresholds using robust statistics:

```
threshold = median(deltas) + k × MAD(deltas)
```

Where:
- `median(deltas)` = median absolute delta for this series
- `MAD(deltas)` = Median Absolute Deviation (robust variance estimate)
- `k` = sensitivity multiplier (default: 2.5)

**Why adaptive?**
- Different metrics have different scales (effective_rank vs sv_entropy)
- Different layers may have different volatility
- Avoids manually tuning thresholds per metric

**Parameters:**
- `adaptive_k: float = 2.5` — Higher = fewer events (stricter)
- `adaptive_min_threshold: float = 0.01` — Floor to prevent too-low thresholds

### 3. Peak Selection with Suppression

Candidates (deltas exceeding threshold) undergo non-maximum suppression:

1. Sort candidates by delta magnitude (descending)
2. Select top candidate
3. Suppress candidates within `suppression_radius` steps
4. Repeat until budget exhausted

**Parameters:**
- `peak_suppression_radius: int = 15` — Step distance for suppression
- `max_events_per_series: int = 5` — Total budget per series

### 4. Warmup-Aware Budgeting

Training warmup often produces large but uninformative transients. The system uses **separate budgets** for pre-warmup and post-warmup candidates:

```
max_peaks_post = max_events_per_series - max_pre_warmup
max_peaks_pre = max_pre_warmup
```

**Key property:** Pre-warmup and post-warmup candidates never compete for the same slots.

**Warmup end detection:**
1. Check `meta.json` for `scheduler.warmup_steps`
2. Fall back to `warmup_fraction × total_steps`

**Parameters:**
- `warmup_fraction: float = 0.1` — Fallback warmup fraction
- `max_pre_warmup: int = 1` — Pre-warmup budget (default: 1 event max)

### 5. Composite Events

Events occurring at the same step across multiple metrics are grouped into composites:

- Type: `change_point_composite`
- Metric: `__composite__`
- Contains list of constituent metrics

Composites indicate coordinated geometric changes across multiple descriptors.

## Detection Summary

The system produces a `detection_summary/<run_id>.parquet` with per-series retention statistics:

| Column | Description |
|--------|-------------|
| `n_candidates_raw` | Candidates above threshold |
| `n_candidates_pre` | Pre-warmup candidates |
| `n_candidates_post` | Post-warmup candidates |
| `n_selected_final` | Final selected events |
| `n_selected_pre` | Selected from pre-warmup |
| `n_selected_post` | Selected from post-warmup |
| `n_skipped_suppression` | Skipped by non-max suppression |
| `n_skipped_suppression_pre` | Suppression skips in pre-warmup |
| `n_skipped_suppression_post` | Suppression skips in post-warmup |
| `n_skipped_topk_post` | Budget overflow in post-warmup |
| `n_skipped_pre_warmup_cap` | Exceeded pre-warmup budget |
| `retention_ratio` | n_selected / n_candidates |

### Accounting Invariant

For each series:
```
n_selected + n_skipped_suppression + n_skipped_topk + n_skipped_pre_warmup_cap = n_candidates_raw
```

This invariant prevents silent counting bugs.

## Output Schema

Events are written to `events_candidates/<run_id>.parquet`:

| Column | Type | Description |
|--------|------|-------------|
| `run_id` | str | Run identifier |
| `layer` | int | Layer index |
| `metric` | str | Geometry metric name |
| `step` | int | Peak step |
| `start_step` | int | Window start |
| `end_step` | int | Window end |
| `delta` | float | Signed change magnitude |
| `event_type` | str | `change_point` or `change_point_composite` |
| `score` | float | Event importance score |
| `direction` | str | `increase` or `decrease` |

## Interpreting Retention Metrics

The retention summary helps diagnose detection behavior:

**High suppression rate (>30%):**
Candidates cluster heavily. Suppression is collapsing near-duplicates.

**High pre-warmup cap rate:**
Many early candidates exceed the pre-warmup budget. Warmup is noisy.

**Pre-retention << Post-retention:**
Warmup produces more noise than signal. The warmup gate is working.

**Suppression mostly post-warmup:**
Post-warmup candidates cluster (local bursts during training).

## CLI Options

```bash
python -m squiggle_analysis --run-id <RUN_ID> \
    --suppression-radius 15 \
    --max-events-per-series 5 \
    --warmup-fraction 0.1 \
    --max-pre-warmup 1 \
    --analysis-id custom_analysis \
    --force
```

| Option | Default | Description |
|--------|---------|-------------|
| `--run-id` | required | Run ID to analyze |
| `--force` | off | Recompute even if exists |
| `--analysis-id` | auto | Analysis version identifier |
| `--suppression-radius` | 15 | Step distance for suppression |
| `--max-events-per-series` | 5 | Budget per (layer, metric) |
| `--warmup-fraction` | 0.1 | Fallback warmup fraction |
| `--max-pre-warmup` | 1 | Pre-warmup event budget |
| `--overwrite` | off | Overwrite existing analysis |

## Tuning Guidelines

| Symptom | Adjustment |
|---------|------------|
| Too many events | Increase `adaptive_k` (e.g., 3.0) |
| Events cluster at same step | Increase `suppression_radius` |
| Missing early events | Increase `max_pre_warmup` |
| Missing late events | Decrease `adaptive_k` |
| Warmup dominates | Decrease `max_pre_warmup` to 0 |

## Analysis Versioning

When running analysis with different detection parameters, outputs are versioned by `analysis_id` to prevent overwrites and enable comparison.

### Auto-generated Analysis ID

If not specified, `analysis_id` is auto-generated from detection parameters:

```
w{warmup%}_p{pre_warmup}_r{radius}_e{events}_k{k*10}
```

Example: `w10_p1_r15_e5_k25` means:
- `warmup_fraction=0.1` (10%)
- `max_pre_warmup=1`
- `peak_suppression_radius=15`
- `max_events_per_series=5`
- `adaptive_k=2.5`

### Versioned Output Paths

With analysis_id, outputs are stored in subdirectories:

```
events_candidates/<run_id>/<analysis_id>.parquet
detection_summary/<run_id>/<analysis_id>.parquet
runs/<run_id>/reports/<analysis_id>/report.md
runs/<run_id>/reports/<analysis_id>/llm_analysis.json
```

### CLI Usage

```bash
# Run with auto-generated analysis_id (from parameters)
python -m squiggle_analysis --run-id <RUN_ID> --warmup-fraction 0.2 --max-pre-warmup 0

# Run with explicit analysis_id
python -m squiggle_analysis --run-id <RUN_ID> --analysis-id my_custom_analysis

# List available analysis versions for a run
python -m squiggle_analysis --list-analyses <RUN_ID>
```

### Comparing Specific Analyses

When comparing runs, you can specify which analysis version to use:

```bash
python -m squiggle_analysis --compare RUN_ID1 RUN_ID2 \
    --compare-analysis-ids w20_p0_r15_e5_k25 w20_p0_r15_e5_k25
```

If not specified, the latest analysis for each run is auto-selected.

## Cross-Run Comparison

Use `compare_runs.py` to compare event detection across seeds:

```bash
python -m squiggle_analysis.compare_runs RUN_ID1 RUN_ID2 --output comparison.md
```

### Report Sections

**Config Fingerprint:**
Verifies runs used identical detection parameters. Displays hash for quick comparison.

**Invariance Metrics (Jaccard):**
- Jaccard similarity: `|intersection| / |union|`
- Precision A→B: What fraction of A's events appear in B
- Recall B→A: What fraction of B's events appear in A

**Event Raster Plots:**
Visual comparison showing step (x-axis) vs layer (y-axis), colored by metric.
Includes intersection raster showing events common to both runs.

**Retention Comparison:**
- Per-run retention statistics
- Density diagnostics (candidates/series)
- Interprets retention differences (density → suppression)

**Trajectory Correlation:**
Pairwise correlation of geometry trajectories across runs.

### Interpreting Results

| Metric | Strong Invariance | Weak Invariance |
|--------|-------------------|-----------------|
| Jaccard | > 50% | < 25% |
| Common Events | > 5 | 0 |
| Trajectory Corr | > 0.9 | < 0.8 |

Events appearing consistently across seeds indicate genuine learning dynamics rather than random fluctuations.

### Config Mismatch Warning

If detection configs differ between runs, the report will warn:
```
WARNING: Detection configs differ between runs! Results may not be comparable.
```

Always verify config fingerprints match before drawing conclusions.

## LLM Qualitative Analysis

Both single-run analysis and cross-run comparison support optional LLM-based qualitative analysis. This sends the generated report to an LLM (GPT-4o or Claude) for expert interpretation.

### Enabling LLM Analysis

```bash
# Single-run with LLM analysis
python -m squiggle_analysis --run-id <RUN_ID> --llm-analysis

# Comparison with LLM analysis
python -m squiggle_analysis --compare RUN_ID1 RUN_ID2 \
    --output comparison.md \
    --llm-analysis
```

### CLI Options

| Option | Default | Description |
|--------|---------|-------------|
| `--llm-analysis` | off | Enable LLM qualitative analysis |
| `--llm-backend` | openai | Backend: `openai` or `anthropic` |
| `--llm-model` | gpt-4o | Model to use for analysis |
| `--llm-question` | (none) | Specific question to ask the LLM |

### Environment Setup

```bash
# For OpenAI
export OPENAI_API_KEY=sk-...

# For Anthropic
export ANTHROPIC_API_KEY=sk-ant-...
```

### Output Format

LLM analysis is written to a separate JSON file:
- Single-run: `runs/<run_id>/reports/llm_analysis.json`
- Comparison: `<output>.llm_analysis.json`

The output follows a structured schema:

```json
{
  "analysis": {
    "headline_summary": ["...", "...", "..."],
    "overall_assessment": {
      "verdict": "strong|moderate|weak|inconclusive",
      "confidence": 0.0-1.0,
      "why": ["...", "...", "..."]
    },
    "key_findings": [
      {
        "title": "...",
        "evidence": ["..."],
        "interpretation": "...",
        "impact": "high|medium|low"
      }
    ],
    "hypotheses": [
      {
        "hypothesis": "...",
        "supporting_evidence": ["..."],
        "counter_evidence": ["..."],
        "confidence": 0.0-1.0,
        "tests": ["..."]
      }
    ],
    "recommended_next_actions": [
      {
        "action": "...",
        "goal": "...",
        "steps": ["..."],
        "effort": "low|medium|high",
        "expected_impact": "low|medium|high"
      }
    ],
    "inconsistencies_or_red_flags": [...],
    "suggested_report_improvements": [...],
    "questions_to_answer_next": [...]
  },
  "provenance": {
    "model_id": "gpt-4o-2024-...",
    "prompt_hash": "abc123...",
    "timestamp": "2024-01-28T12:00:00Z"
  }
}
```

### Context Provided to LLM

The LLM receives:
1. **Run context** — run IDs, seeds, config fingerprints, step ranges
2. **Report content** — the full markdown report
3. **Squiggle framework context** — definitions of metrics, event detection, and seed invariance concepts
4. **Artifacts manifest** — list of generated plots

### Asking Specific Questions

Use `--llm-question` to focus the analysis:

```bash
# Ask about seed invariance
python -m squiggle_analysis --compare RUN1 RUN2 \
    --llm-analysis \
    --llm-question "Are these runs seed-invariant post-warmup? What explains the retention difference?"

# Ask about anomalies
python -m squiggle_analysis --run-id <RUN_ID> \
    --llm-analysis \
    --llm-question "Are there any red flags in this run that suggest data or configuration issues?"
```

### Installation

LLM analysis requires the `llm` optional dependency:

```bash
pip install squiggle-core[llm]
pip install squiggle-analysis[llm]
```

This installs the `openai` and `anthropic` packages.
