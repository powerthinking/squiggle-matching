# Probe Harness Design – v1

## 1. Purpose

The probe harness measures **how training data changes a model internally**, rather than only how it affects loss or accuracy.

It is explicitly **not** an optimization tool.

Its role is:
- Comparative measurement
- Repeatable selection signal
- Lightweight instrumentation

---

## 2. Core Concept

Each probe run answers:

> *“What geometric / representational changes does this training data induce?”*

The output is a **Dynamics Signature** and a **Dynamics Impact Score (DIS)**.

---

## 3. Probe Protocol

For each problem family:

1. Load fixed base checkpoint
2. Snapshot internal metrics (pre)
3. Micro-fine-tune on family batch
4. Snapshot internal metrics (post)
5. Compute deltas
6. Store signature + score

All runs use:
- Fixed step count
- Fixed optimizer
- Fixed learning rate
- 1–2 seeds (for stability estimation)

---

## 4. Dynamics Signature (v1 – Frozen)

### 4.1 Layerwise Metrics

Computed for selected layers (e.g., every N layers):

- Effective rank of activations
- Singular value spectrum slope
- Attention entropy
- MLP activation sparsity proxy
- Representation drift norm (Δ activation)

### 4.2 Global Metrics

- Training loss delta
- Gradient norm statistics
- Parameter update norm
- Layerwise drift distribution

### 4.3 Stability Metrics

- Variance across seeds
- Consistency of affected layers

---

## 5. Dynamics Impact Score (DIS)

### 5.1 Components

**Magnitude**
- Normed size of representational change

**Coherence**
- Concentration of change in consistent layers/modules

**Novelty**
- Distance from existing signatures (cosine / L2)

### 5.2 Output

Each probe emits:

```json
{
  "schema_version": "probe_summary@2.0",

  "identity": {
    "family_id": "...",
    "model_id": "...",
    "base_checkpoint": "...",
    "run_mode": "probe_micro_finetune",
    "steps": 100,
    "seed": 123,
    "dtype": "bf16",
    "device": "cuda",
    "timestamp_utc": "2026-01-12T16:30:00Z"
  },

  "capture_ref": {
    "run_id": "...",
    "probe_name": "layerwise_geometry_v1",
    "probe_config_hash": "...",
    "captures_manifest_path": "runs/<run_id>/probes/layerwise_geometry_v1/manifest.json",
    "captures_index_path": "runs/<run_id>/probes/layerwise_geometry_v1/index.parquet",
    "capture_steps_used": [0, 50, 100],
    "notes": {
      "dropped_steps": [],
      "warnings": []
    }
  },

  "layer_A_state": {
    "description": "Basis-invariant geometric state descriptors (snapshot geometry).",
    "layers_covered": [0, 4, 8, 12, 16, 20, 24],
    "metrics": {
      "effective_rank": {
        "by_layer": [/* float per layer */],
        "pre": [/* float per layer */],
        "post": [/* float per layer */],
        "delta": [/* post - pre */]
      },
      "sv_entropy": {
        "pre": [/* per layer */],
        "post": [/* per layer */],
        "delta": [/* per layer */]
      },
      "sparsity_proxy": {
        "pre": [/* per layer */],
        "post": [/* per layer */],
        "delta": [/* per layer */]
      },
      "principal_angle_to_pre": {
        "post_vs_pre": [/* per layer angle or similarity */]
      }
    },
    "aggregations": {
      "summary_pre": {
        "mean_abs": 0.0,
        "median_abs": 0.0,
        "p95_abs": 0.0
      },
      "summary_delta": {
        "mean_abs": 0.0,
        "median_abs": 0.0,
        "p95_abs": 0.0
      }
    }
  },

  "layer_B_dynamics": {
    "description": "Temporal descriptors of change in geometric state (velocity/acceleration/volatility).",
    "windowing": {
      "type": "steps",
      "dt": 1,
      "smoothing": "ema",
      "ema_alpha": 0.2
    },
    "metrics": {
      "drift_velocity": {
        "by_layer": [/* float per layer */],
        "global": 0.0
      },
      "drift_acceleration": {
        "by_layer": [/* float per layer */],
        "global": 0.0
      },
      "volatility": {
        "by_layer": [/* float per layer */],
        "global": 0.0
      },
      "alignment_velocity": {
        "by_layer": [/* float per layer */],
        "global": 0.0
      }
    },
    "affected_layers": {
      "method": "topk_by_abs(drift_velocity)",
      "k": 6,
      "layers": [12, 13, 14]
    }
  },

  "layer_C_event_candidates": {
    "enabled": true,
    "detector_version": "events@0.1",
    "timewarp_tolerance_steps": 25,
    "candidates": [
      {
        "event_id": "e1",
        "type": "rank_collapse",
        "t_step": 73,
        "layers": [12, 13],
        "strength": 0.62,
        "supporting_signals": {
          "effective_rank_delta": -0.31,
          "volatility": 0.44
        }
      }
    ],
    "notes": "Candidates only; consensus and matching occur post-run across seeds/tests."
  },

  "signature": {
    "signature_version": "sig@2.0",
    "construction": {
      "basis_invariant": true,
      "inputs": ["layer_A_state.metrics", "layer_B_dynamics.metrics"],
      "includes_layer_C": "optional"
    },
    "vector": [/* floats */],
    "vector_semantics": [
      "A:eff_rank_delta_mean",
      "A:sv_entropy_delta_mean",
      "B:drift_velocity_global",
      "B:volatility_global"
    ],
    "vector_norm": 0.0
  },

  "ranking": {
    "score_version": "dis@1.0",
    "formula": "Magnitude × Coherence × Novelty",
    "components": {
      "magnitude": 0.73,
      "coherence": 0.81,
      "novelty": 0.64
    },
    "total": 0.38,
    "interpretation": "Operational ranking only; not part of squiggle identity or matching."
  }
}


```

---
## 6. Directory Layout
```
probe/
├── configs/
│   └── probe.yaml
├── runs/
│   └── family_id/
│       ├── metrics_pre.json
│       ├── metrics_post.json
│       ├── signature.json
│       └── logs.txt
├── src/
│   ├── runner.py
│   ├── metrics.py
│   ├── scoring.py
│   └── utils.py
└── README.md
```

---
## 7. Design Constraints (Intentional)

- No full-training runs
- No benchmark chasing
- No adaptive hyperparameters
- No metric additions mid-cycle

These constraints protect comparability.

---
## 8. Expected Failure Modes

- Noisy metrics → increase batch size or steps slightly
- Over-correlated signatures → improve novelty pressure
- Low signal → adjust layer sampling

Failures are logged, not hidden.

---
## 9. Outputs Used Downstream

- Dataset selection
- Ablation experiments
- Dataset card evidence
- Research analysis

The probe harness is deliberately minimal so it can be trusted.
