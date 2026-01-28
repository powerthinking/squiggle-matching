# Curriculum-Driven Training

This document describes the curriculum system for controlling how training samples are selected over time based on problem family membership.

## Overview

A **curriculum** is a time-varying sampling policy over classified problem families. Rather than shuffling all training data uniformly, curricula allow you to:

- Focus early training on specific problem families
- Gradually introduce new families over time
- Control the balance between families with different item counts
- Study how training order affects representation learning

The key insight is that curricula are implemented as **sampling policies**, not static reorderings. This makes them:
- Easy to A/B test
- Reproducible via manifests
- Compatible with large datasets (streaming, sharding)

## Configuration

Enable curriculum training in your research config YAML:

```yaml
curriculum:
  enabled: true
  spec_path: "curricula/family_ramp.yaml"
  trace_enabled: false        # Write sample trace for exact replay
  log_distribution_every: 100 # Log family distribution every N steps (0 = disabled)
```

### Config Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | false | Enable curriculum-driven training |
| `spec_path` | string | null | Path to curriculum YAML specification |
| `trace_enabled` | bool | false | Write sample trace for deterministic replay |
| `log_distribution_every` | int | 100 | Log family weights every N steps (0 = disabled) |

## Curriculum YAML Specification

Curricula are defined in YAML files with the following structure:

```yaml
version: 1
name: my_curriculum
time_unit: steps  # "steps" or "epochs"

defaults:
  sampling:
    mode: balanced_family
    replacement: true

phases:
  - name: phase_1
    start: 0.0      # Fraction of total training (0.0 to 1.0)
    end: 0.5
    families:
      include: [family_a, family_b]
      exclude: []   # Optional
    weights:
      type: uniform # or "proportional", "explicit", "ramp"
    sampling:
      mode: balanced_family

  - name: phase_2
    start: 0.5
    end: 1.0
    families:
      include: "*"  # All families
    weights:
      type: proportional
    sampling:
      mode: proportional_family
```

### Phase Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Identifier for the phase |
| `start` | Yes | Start fraction (0.0-1.0) of total training |
| `end` | Yes | End fraction (0.0-1.0) of total training |
| `families.include` | Yes | List of family IDs or `"*"` for all |
| `families.exclude` | No | List of family IDs to exclude |
| `weights.type` | No | Weight distribution type (default: "uniform") |
| `sampling.mode` | No | Sampling mode override for this phase |

### Weight Types

**uniform**: All included families have equal weight.

```yaml
weights:
  type: uniform
```

**proportional**: Families weighted by their item count in the dataset.

```yaml
weights:
  type: proportional
```

**explicit**: Manually specify weight per family.

```yaml
weights:
  type: explicit
  explicit:
    family_a: 2.0
    family_b: 1.0
```

**ramp**: Linearly interpolate weights over the phase duration.

```yaml
weights:
  type: ramp
  ramp:
    from:
      family_a: 1.0
      family_b: 0.0
    to:
      family_a: 1.0
      family_b: 1.0
```

## Sampling Modes

Three sampling modes control how batches are constructed:

### balanced_family

1. Choose a family uniformly from included families
2. Choose an item uniformly within that family

**Use when**: You want equal representation per family regardless of item counts. Prevents large families from dominating.

### proportional_family

1. Choose a family proportional to its item count
2. Choose an item uniformly within that family

**Use when**: You want the training distribution to match the underlying dataset distribution.

### uniform_item

1. Pool all items from included families
2. Choose uniformly from the pool

**Use when**: You want simple uniform sampling ignoring family structure.

## Available Curricula

Four preset curricula are provided in `squiggle-experiments/curricula/`:

### iid_baseline.yaml

Single phase with proportional sampling. Matches the underlying data distribution.

```
Phase 1 [0-100%]: All families, proportional weights
```

### balanced_baseline.yaml

Single phase with balanced sampling. All families get equal representation.

```
Phase 1 [0-100%]: All families, uniform weights
```

### family_blocked.yaml

Sequential phases, one target family at a time, then full mix.

```
Phase 1 [0-10%]:   functional_equations only
Phase 2 [10-20%]:  bijection_counting only
Phase 3 [20-30%]:  modular_arithmetic only
Phase 4 [30-40%]:  graph_recurrence only
Phase 5 [40-50%]:  invariant_monovariant only
Phase 6 [50-60%]:  cyclic_groups only
Phase 7 [60-100%]: All families, balanced
```

### family_ramp.yaml

Three-phase curriculum with gradual introduction of control families.

```
Phase 1 [0-30%]:   Target families only (6 families, uniform)
Phase 2 [30-70%]:  Ramp in control families (weights 0→1)
Phase 3 [70-100%]: Full mix (proportional)
```

## Problem Families

The curriculum system is designed around the classified problem families from the OpenMathReasoning dataset:

**Target Families (6):**
- functional_equations
- bijection_counting
- modular_arithmetic
- graph_recurrence
- invariant_monovariant
- cyclic_groups

**Control Families (11):**
- geometry_euclidean
- calculus_analysis
- optimization
- algebra_polynomials
- misc_other
- sequences_series
- probability_expectation
- combinatorics_basic
- inequalities
- number_theory_basic
- logic_proof

## Artifacts

When curriculum training is enabled, the following artifacts are written:

### Curriculum Manifest

Written to `runs/<run_id>/curriculum_manifest.json`:

```json
{
  "curriculum_name": "family_ramp_v1",
  "curriculum_version": 1,
  "yaml_hash": "sha256:...",
  "total_steps": 50000,
  "seed": 42,
  "created_at_utc": "2024-01-15T10:30:00+00:00",
  "resolved_phases": [
    {
      "name": "phase_1_targets_only",
      "start_step": 0,
      "end_step": 15000,
      "families": ["functional_equations", "bijection_counting", ...],
      "weights": {"functional_equations": 0.166, ...},
      "sampling_mode": "balanced_family"
    }
  ]
}
```

### Sample Trace (Optional)

When `trace_enabled: true`, writes to `runs/<run_id>/sample_trace.jsonl`:

```json
{"step": 0, "phase": "phase_1", "family_id": "bijection_counting", "item_id": "omr:cot:123"}
{"step": 0, "phase": "phase_1", "family_id": "functional_equations", "item_id": "omr:cot:456"}
```

The sample trace enables exact replay of training even if the underlying dataset changes.

### Scalar Metrics

The `curriculum_phase` column is added to scalar logs, recording which phase was active at each step.

## Implementation Details

### CurriculumSpec

Parses and validates curriculum YAML. Key methods:

- `from_yaml(path)` - Load and parse a curriculum file
- `resolve_phases(total_steps, families, counts)` - Convert fractional boundaries to absolute steps
- `phase_at_step(step, ...)` - Get the active phase for a given step
- `get_weights_at_step(step, ...)` - Get interpolated weights for ramp phases
- `to_manifest(...)` - Generate reproducibility manifest

### CurriculumSampler

Step-aware batch sampler. Key methods:

- `set_step(step)` - Update current training step (call before each batch)
- `sample_batch()` - Return list of dataset indices for next batch

### Integration Points

The curriculum system integrates with the trainer at these points:

1. **Dataset loading**: `build_family_index()` creates the family→indices mapping
2. **Sampler creation**: `CurriculumSampler` initialized with spec and family index
3. **Training loop**: `set_step()` called each step, `sample_batch()` replaces DataLoader iteration
4. **Logging**: Phase name added to scalar rows, distribution logged periodically

## Validation

The curriculum system validates:

- All phases have non-empty eligible families
- Phase boundaries are contiguous (gaps are allowed but logged)
- Weights normalize correctly
- Ramp endpoints are valid

Validation errors are raised at training start, not during curriculum loading.

## Best Practices

1. **Use fractional boundaries**: Define phases as fractions (0.0-1.0) so the same curriculum works for different run lengths.

2. **Start simple**: Begin with `balanced_baseline.yaml` before testing complex curricula.

3. **Monitor distribution**: Set `log_distribution_every: 100` to verify families are sampled as expected.

4. **Enable manifests**: Always review `curriculum_manifest.json` to confirm phase resolution.

5. **Consider family sizes**: Use `balanced_family` when family sizes are highly skewed to prevent large families from dominating.

## Reproducibility

To exactly reproduce a curriculum run:

1. Use the same `seed` in config
2. Use the same curriculum YAML (verified by `yaml_hash` in manifest)
3. Use the same dataset (verified by family counts in manifest)

For absolute reproducibility across dataset changes, enable `trace_enabled: true` and replay from the sample trace.
