# Cross-Seed Invariance Analysis

This document describes the cross-seed analysis system in squiggle-analysis, which compares training runs with different random seeds to identify reproducible learning dynamics.

## Motivation

A fundamental question for any event detection system: **Are detected events real, or artifacts of randomness?**

Cross-seed analysis answers this by comparing runs that differ only in random seed. Events that appear consistently across seeds indicate genuine learning dynamics; events that appear in only one seed are likely noise.

## The Invariance Stack

The analysis computes invariance at multiple levels, forming a diagnostic hierarchy:

```
Level 1: Trajectory Correlation (Signal)
    ↓
Level 2: Neighbor Coverage (Neighborhoods)
    ↓
Level 3: Region Jaccard (Clustered Episodes)
    ↓
Level 4: Peak IoU Jaccard (Discrete Winners)
```

Each level answers a different question:

| Level | Metric | Question |
|-------|--------|----------|
| **Signal** | Trajectory Correlation | Do the geometry curves look the same? |
| **Neighborhood** | Neighbor Coverage | Do both seeds show activity in the same neighborhoods? |
| **Region** | Region Jaccard | Do clustered episodes match one-to-one? |
| **Peak** | IoU Jaccard | Do they select the same discrete event winners? |

### Interpreting the Stack

The most common pattern is **"stable energy landscape / unstable argmax"**:

- High trajectory correlation (~0.9)
- High neighbor coverage (~50%)
- Low peak IoU Jaccard (~30%)

This means: *Both seeds detect events in the same neighborhoods, but pick different winners within those neighborhoods.* The underlying signal is reproducible; only the final peak selection is sensitive to noise.

## Core Metrics

### 1. Trajectory Correlation

Pearson correlation of geometry metric trajectories between runs.

```
For each (layer, metric) pair:
    corr = pearson(trajectory_A, trajectory_B)
```

**Interpretation:**
- `> 0.9` — Strong signal invariance
- `0.8–0.9` — Moderate invariance
- `< 0.8` — Runs may have divergent dynamics

### 2. Neighbor Coverage

Measures whether events have neighbors in the other run within a radius (typically suppression_radius).

**Definition:**
- **Signature (matching key)**: (layer, metric, event_type)
- **Neighbor condition**: ∃ event in other run with |Δstep| ≤ radius_steps
- **Formula**: `(A_has_neighbor + B_has_neighbor) / (|A| + |B|)`

**Important:** This is NOT a true Jaccard. It's a symmetric coverage score that answers: *"What fraction of events have any neighbor in the other run?"*

**Output includes:**
- `neighbor_coverage` — Overall coverage rate
- `coverage_a`, `coverage_b` — Per-run coverage rates
- `events_a_with_neighbor`, `events_b_with_neighbor` — Counts
- `winner_stability` — Fraction where winners match exactly
- `winner_close_stability` — Fraction where winners are within half-radius
- `nn_distance_stats` — Nearest-neighbor distance distribution

**Winner selection:**
- Winner = argmax by score within matched events
- Tiebreak: smallest step (deterministic)

### 3. Peak IoU Jaccard

Traditional event matching using window overlap.

**Definition:**
- Each event has a window: `[start_step, end_step]`
- IoU = Intersection / Union of windows
- Events match if IoU ≥ threshold (default: 0.2)
- Jaccard = |matched pairs| / |union of events|

**Window dilation:** The `window_dilation` parameter expands windows before comparison, compensating for grid discretization artifacts.

### 4. Region Jaccard (True Jaccard)

Clusters events into region-level entities, then computes true set Jaccard.

**Definition:**
- Cluster peaks within suppression_radius using single-linkage
- Each cluster becomes a "region" with centroid and spread
- Match regions one-to-one by greedy assignment
- Jaccard = |matched regions| / |union of regions|

## Diagnostic Outputs

### Nearest-Neighbor Distance Statistics

For each event, compute distance to nearest event in other run:

```
nn_distance_stats:
  median: 2.0          # Typical offset
  p90: 8.0             # 90th percentile
  p95: 12.0            # 95th percentile
  within_radius: 51%   # Fraction within suppression radius
  within_half_radius: 47%  # Fraction within half radius
```

These explain *why* coverage is what it is. Median of 2 steps means events are well-aligned; the IoU ceiling is due to short windows, not misalignment.

### Retention Comparison

Detection retention statistics help diagnose divergence sources:

| Metric | Description |
|--------|-------------|
| `n_candidates` | Raw candidates above threshold |
| `n_selected` | Final events after suppression |
| `retention_rate` | n_selected / n_candidates |
| `suppression_rate` | Fraction lost to suppression |
| `mean_candidates_per_series` | Candidate density |

**Key insight:** If one run has 1.5x the candidate density, it will have more suppression collisions, leading to different winner selection even with identical underlying signal.

### Center Distance Diagnostics

For IoU-matched event pairs, compute center-to-center distances:

```
Center Distance Statistics:
  Median: 2.0 steps
  p90: 4.0 steps
  Max: 8.0 steps
```

Small median distance + low IoU Jaccard indicates the IoU ceiling is due to **short event windows**, not event misalignment.

## Interactive Notebook

The `compare_runs_interactive.ipynb` notebook provides visualization and LLM-assisted analysis:

### Sections

1. **Event Raster Plots** — Visual comparison of event locations
2. **Common Events** — Strictly matching events (±5 steps)
3. **IoU Invariance** — Jaccard curves vs threshold
4. **Grid Cadence Analysis** — Window/suppression scale mismatch diagnosis
5. **Center Distance** — Event alignment histogram
6. **Window Dilation Experiment** — IoU ceiling investigation
7. **Neighbor Coverage** — The "middle layer" metric
8. **Candidate vs Selected** — Pre/post suppression comparison
9. **IoU Distribution** — Histogram of match qualities
10. **Trajectory Comparison** — Per-layer correlation plots
11. **Composite Events** — Multi-metric event invariance
12. **Phase Distribution** — Temporal event distribution
13. **Retention Analysis** — Detection pipeline comparison

### LLM Analysis

Each section can be analyzed by an LLM (GPT-4o-mini by default) that provides:
- Interpretation of the data
- Key observations
- Follow-up questions

An overall synthesis combines section analyses into a comprehensive assessment.

## CLI Usage

### Basic Comparison

```bash
python -m squiggle_analysis --compare RUN_ID1 RUN_ID2 --output comparison.md
```

### With Specific Analysis Versions

```bash
python -m squiggle_analysis --compare RUN_ID1 RUN_ID2 \
    --compare-analysis-ids w10_p1_s15 w10_p1_s15 \
    --output comparison.md
```

### With LLM Analysis

```bash
python -m squiggle_analysis --compare RUN_ID1 RUN_ID2 \
    --output comparison.md \
    --llm-analysis \
    --llm-question "What explains the low event Jaccard despite high trajectory correlation?"
```

### N-Way Comparison

```bash
python -m squiggle_analysis --compare RUN1 RUN2 RUN3 RUN4 --output multi_seed.md
```

## Interpreting Results

### Case 1: Stable Energy Landscape / Unstable Argmax

```
Trajectory Correlation: 0.91
Neighbor Coverage: 51%
Peak IoU Jaccard: 29%
Winner Close Stability: 95%
```

**Interpretation:** Both seeds detect the same learning dynamics. The underlying signal is highly reproducible. Low IoU Jaccard is due to:
1. Short event windows (3 capture points)
2. Suppression radius >> window width
3. Different winner selection within matched neighborhoods

**Recommendation:** Report neighbor coverage as the primary invariance metric. Peak IoU Jaccard understates reproducibility due to window discretization.

### Case 2: Genuinely Different Dynamics

```
Trajectory Correlation: 0.72
Neighbor Coverage: 23%
Peak IoU Jaccard: 8%
```

**Interpretation:** Runs have divergent learning trajectories. Events are not reproducible across seeds. This may indicate:
- High learning rate sensitivity
- Chaotic training dynamics
- Configuration differences (check config fingerprints)

### Case 3: Detection Sensitivity Issues

```
Trajectory Correlation: 0.94
Neighbor Coverage: 31%
Peak IoU Jaccard: 15%
Retention rate A: 35%
Retention rate B: 62%
```

**Interpretation:** High trajectory correlation but asymmetric retention suggests detection sensitivity issues, not signal differences. One run may have different noise characteristics affecting threshold crossings.

## Best Practices

### 1. Always Check Config Fingerprints

Before comparing runs, verify they used identical detection parameters:

```
Config fingerprint: abc123 (both runs)
```

Mismatched configs make comparison meaningless.

### 2. Report Multiple Invariance Levels

Don't report just IoU Jaccard. Report the full stack:

> Trajectory correlation: 0.91 (signal level)
> Neighbor coverage: 51% (neighborhood level)
> Peak IoU Jaccard: 29% (discrete selection level)
> Winner stability (±7 steps): 95%

This tells the full story.

### 3. Use Nearest-Neighbor Stats for Context

Include NN distance stats to explain coverage numbers:

> Median NN distance: 2.0 steps
> 47% of events within ±7 steps of a neighbor

This makes "95% winner close stability at ±7 steps" interpretable.

### 4. Consider Window Scale

If using small event windows (radius=1 capture point) with wide suppression (15+ steps), expect:
- Low IoU even for well-aligned events
- IoU ceiling around 0.5 for 1-tick offset

Use `window_dilation` to diagnose discretization effects.

## API Reference

### `compute_neighbor_coverage(runs, radius_steps=15)`

Compute bidirectional neighbor coverage.

**Returns:**
```python
{
    "neighbor_coverage": float,      # Overall coverage rate
    "coverage_a": float,             # A's coverage rate
    "coverage_b": float,             # B's coverage rate
    "events_a_with_neighbor": int,
    "events_b_with_neighbor": int,
    "total_events_a": int,
    "total_events_b": int,
    "n_signatures_with_overlap": int,
    "winner_stability": float,       # Exact match rate
    "winner_close_stability": float, # Within half-radius rate
    "nn_distance_stats": {
        "median": float,
        "p90": float,
        "p95": float,
        "max": float,
        "within_radius": float,
        "within_half_radius": float,
    },
    "matched_neighborhoods": [...],  # First 20 details
    "interpretation": str,           # Human-readable assessment
}
```

### `compute_window_overlap_invariance(runs, iou_threshold=0.2, window_dilation=0)`

Compute IoU-based event matching.

### `compute_trajectory_correlation(runs, metric, layers)`

Compute pairwise trajectory correlation matrix.

### `compute_region_invariance(runs, cluster_radius_steps=15)`

Compute true region-level Jaccard (clusters as first-class entities).

### `extract_region_events(run, cluster_radius_steps=15)`

Extract region-level events by clustering peaks.

## Future Directions

1. **N>2 Run Support** — Aggregate invariance across all pairs, compute consensus events
2. **Temporal Stability** — Track how invariance changes through training
3. **Per-Layer Analysis** — Identify layers with higher/lower reproducibility
4. **Causal Analysis** — Link invariance patterns to training hyperparameters
