# AIMO_PLAN.md
**Squiggle-Based Corpus Construction for AIMO Math Corpus Prize**

> Goal: Establish *credible, reproducible evidence* that **squiggle-based geometric events** correlate with learning in math-reasoning LLMs, then use that evidence to construct a **corpus of items that measurably induce learning events**.

---

## 0. Guiding Principles (Non-Negotiable)

- **Not a toy model**: architectures, datasets, and training regimes must resemble real LLM practice.
- **Seed-invariant evidence > single-run anecdotes**: All claims require consensus across seeds.
- **trace_enabled: true required for all experiments**: Every run must capture sample traces for attribution.
- **Separation of concerns**:
  - Training ≠ Analysis
  - Probes ≠ Interpretation
  - Events ≠ Consensus
- **Every artifact traceable** to: run_id, dataset slice, seed, curriculum, config hash
- **Immutability**: Runs are facts. Analyses are versioned interpretations.

**Definition of Done (Global):**
Another researcher can reproduce *at least one* squiggle-learning claim using only the repo + data instructions.

---

## 1. Infrastructure (Complete)

All core infrastructure is implemented and validated.

| Component | Status | Key Features |
|-----------|--------|--------------|
| **Training** | ✅ | Step-based budgets, extension ladder (`step_multiplier`), loss milestones |
| **Test Runner** | ✅ | Multi-seed orchestration, per-seed analysis, consensus extraction |
| **Curriculum** | ✅ | 4 presets: iid, balanced, blocked, ramp (see `curriculum.md`) |
| **Event Detection** | ✅ | Adaptive scoring (DIS), seed-invariant consensus |
| **Dataset** | ✅ | 5000 items, 6 target + 11 control families, split_v1_s42 |

### Key References
- Training: `squiggle-experiments/src/squiggle_experiments/research/trainer.py`
- Test Runner: `squiggle-experiments/src/squiggle_experiments/test_runner/`
- Curriculum: `squiggle-matching/docs/curriculum.md`
- Event Scoring: `squiggle-core/src/squiggle_core/scoring/SCORING.md`
- Dataset: `data/candidate_pool/splits/split_v1_s42/`

---

## 2. The Corpus Pipeline

### 2.1 Meta-Pattern: Trace → Attribution → Corpus

The path from training runs to corpus submission:

1. **Run a Test** (multi-seed) with `trace_enabled: true`
2. **Detect seed-invariant events** via consensus scoring
3. **Use trace windows** to compute event-trigger candidates
4. **Validate with counterfactual replays** (remove/replace trigger items)

This transforms "cool analysis" into a defensible corpus claim.

### 2.2 Key Enablers

| Artifact | Purpose |
|----------|---------|
| `sample_trace.jsonl` | Item IDs per training step (attribution source) |
| `curriculum_manifest.json` | Phase resolution and reproducibility |
| `consensus_events.parquet` | Seed-invariant events with DIS scores |
| `events_candidates/<run_id>.parquet` | Per-run candidate events |

### 2.3 Attribution Flow

```
For each consensus event:
  1. Get event step window (onset to peak)
  2. Extract item_ids from sample_trace in that window
  3. Score items by frequency across seeds
  4. Top-K items = candidate triggers
```

---

## 3. Experiments

All experiments follow the formal structure defined in `design_and_contracts.md` Section 6:

```
experiments/<EXP-ID>/
  README.md                    # Hypothesis, invariants, outcomes
  spec/
    experiment.yaml            # Formal experiment spec
    <arm>_test.yaml            # Per-arm test configs
  outputs/
    compare.md                 # Cross-arm comparison
```

### 3.1 Experiment 1: Curriculum A/B (Event Yield)

**Goal**: Find curricula that produce more/clearer seed-invariant events.

**Hypothesis**: Blocked and ramp curricula yield more consensus events than iid/balanced baselines.

**Design**: 4 arms (Tests), 3 seeds each, identical `total_steps`:

| Arm | Curriculum | Description |
|-----|------------|-------------|
| iid | `iid_baseline` | Proportional sampling, no structure |
| balanced | `balanced_baseline` | Family-balanced sampling |
| blocked | `family_blocked` | One target family at a time |
| ramp | `family_ramp` | Targets-only → ramp controls → full mix |

**Primary Outcomes**:
- `consensus_event_count` per arm
- `event_yield_per_step` (events / total_steps)
- `jaccard_similarity` of trigger sets across arms

**Artifact**: Ranked `item_id`s that precede events, minimal trigger bundles.

---

### 3.2 Experiment 2: Trigger-Set Mining (Counterfactual) ⭐

**Goal**: Demonstrate causality - specific items *cause* events, not training noise.

**Hypothesis**: Removing candidate trigger items eliminates or delays the corresponding event.

**Design**:
1. Run baseline curriculum (e.g., ramp) with 3 seeds
2. Identify T-step window before each consensus event peak
3. Extract `item_id`s from trace windows, compute overlap across seeds
4. Counterfactual runs: replace trigger items with same-family alternatives
5. Compare event presence/timing in counterfactual vs baseline

**Artifact**:
```
(event_signature, trigger_items, family, effect_size)
```

Plus statistics: "removal kills event" / "replacement preserves event"

---

### 3.3 Experiment 3: Order-Matters Tests

**Goal**: Find items whose impact depends on *when* they appear.

**Hypothesis**: Some items trigger events only when preceded by specific exposure patterns.

**Design**: 3 arms, same families, different introduction schedules:

| Arm | Schedule |
|-----|----------|
| blocked_early | Blocked targets early, then mix |
| ramp | Targets-only → ramp controls |
| controls_first | Controls early, ramp targets late |

**Artifact**: Order-sensitive families/items - those that lose impact if introduced too early/late.

---

### 3.4 Experiment 4: Sampling-Mode Stress Test

**Goal**: Verify triggers aren't artifacts of family balancing.

**Hypothesis**: Robust triggers appear regardless of sampling mode.

**Design**: Same curriculum phases, vary sampling mode:
- `balanced_family` vs `proportional_family`
- optionally `uniform_item`

**Artifact**:
- **Robust triggers**: Appear across sampling modes
- **Distribution-sensitive triggers**: Depend on balanced vs proportional

For corpus submission, robust triggers are gold.

---

### 3.5 Experiment 5: Squiggle Impact Curriculum ⭐

**Goal**: Build a curated curriculum + dataset slice as the submission artifact.

**Design** (2 stages):

**Stage A (Discovery)**:
1. Run baseline curricula (Experiment 1)
2. Mine top trigger items per event type
3. Rank items by cross-seed trigger frequency

**Stage B (Construction)**:
1. Create curriculum YAML that starts with trigger set (explicit weights)
2. Ramp in remaining items later
3. Validate: consensus event lift vs baseline

**Artifact**:
- `impact_core.jsonl` - The trigger items
- `impact_ramp.yaml` - Curriculum spec to train on them
- Evidence: seed-invariant event lift vs baseline

This is the most direct "submission-shaped" deliverable.

---

### 3.6 Recommended Priority

Given time constraints: **Experiment 2 + Experiment 5 Stage B**

- Minimal new engineering (trace + manifests + curricula exist)
- Strongest claim (event disappears when items removed)
- Clean deliverables (ranked list / slice you can ship)

---

## 4. Submission Artifacts

### 4.1 Corpus Package

| File | Description |
|------|-------------|
| `impact_core.jsonl` | Items that measurably induce learning events |
| `impact_ramp.yaml` | Curriculum spec: how to train on them |
| `manifest.json` | Provenance, hashes, config versions |

### 4.2 Evidence Package

| File | Description |
|------|-------------|
| `consensus_events.parquet` | Seed-invariant events with DIS scores |
| `trigger_attribution.parquet` | Items → events mapping with scores |
| `counterfactual_results.parquet` | Removal/replacement effect sizes (if Exp 2 run) |
| `reproducibility/` | Seeds, config hashes, trace files |

---

## 5. Probes & Reports Status

### 5.1 Probes - What Exists ✅

| Component | Status | Location |
|-----------|--------|----------|
| Probe harness | ✅ | `squiggle-instrumentation/src/squiggle_instrumentation/probe_runner.py` |
| HF adapter (LoRA) | ✅ | `squiggle-instrumentation/src/squiggle_instrumentation/hf_adapter.py` |
| Layer A metrics | ✅ | effective_rank, sv_entropy, sparsity, principal_angle |
| Layer B metrics | ✅ | drift_velocity, alignment_velocity, volatility |
| Probe aggregation | ✅ | `squiggle-analysis/src/squiggle_analysis/io/write_probe_parquet_per_experiment.py` |
| CLI entry point | ✅ | `squiggle-probe --exp-id X --adapter hf` |

### 5.2 Reports - What Exists ✅

| Component | Status | Location |
|-----------|--------|----------|
| Single-run report | ✅ | `squiggle-analysis/src/squiggle_analysis/reporting/report_md.py` |
| Events summary | ✅ | Top-30 events table, detection params |
| LLM analysis framework | ✅ | `squiggle-analysis/src/squiggle_analysis/llm_analysis/` |
| Multi-run comparison | 🟡 | `squiggle-analysis/src/squiggle_analysis/compare_runs.py` (framework exists) |
| Trajectory plots | 🟡 | `squiggle-analysis/src/squiggle_analysis/trajectories.py` (not in reports) |

### 5.3 Gaps to Address

| Gap | Priority | Description |
|-----|----------|-------------|
| Multi-seed consensus DIS | HIGH | `find_common_events()` exists but no weighted DIS consensus |
| Probe ↔ Event correlation | HIGH | No temporal alignment of probe changes with events |
| Visualization in reports | MEDIUM | Plots exist but not embedded in report.md |
| Evidence bundling | MEDIUM | No automated packaging of configs + reports + artifacts |
| Probe A vs B separation | LOW | Currently single snapshot; need train vs holdout probes |

### 5.4 Required Work for Experiments

**For Experiment 2 (Trigger Mining):**
- ❌ Need: Trace window extraction (`item_ids` from `sample_trace.jsonl`)
- ❌ Need: Event attribution scoring (which items precede events)
- ✅ Have: Consensus event detection, trace artifacts

**For Experiment 5 (Impact Curriculum):**
- ❌ Need: Trigger ranking algorithm (item → event lift)
- ❌ Need: Curriculum generator from trigger set
- ✅ Have: Curriculum YAML spec, manifest generation

---

## 6. Terminology Alignment

Per `design_and_contracts.md`:

| Term | Definition |
|------|------------|
| **Run** | Single training execution, immutable artifacts |
| **Test** | Collection of runs identical except seed (replicate set) |
| **Experiment** | Structured comparison of 2+ Tests |
| **Consensus Events** | Events appearing consistently across all seeds in a Test |
| **Curriculum Instance** | Fully realized training structure (immutable) |
| **Curriculum Template** | Reusable policy (ordering, mixing, pacing rules) |

---

## 7. Non-Goals

- ❌ Winning the leaderboard immediately
- ❌ Full consensus events across dozens of seeds
- ❌ Cross-model matching at scale
- ❌ Interpretability claims beyond geometry + correlation
- ❌ Single-seed claims without consensus
- ❌ Timeline/deadline commitments

---

## Success Criteria

This project is **successful** if we can say:

> "We identified a set of training items that reliably induce seed-invariant geometric events in a non-toy model. When these items are removed or replaced, the events disappear. When they are prioritized in curriculum, the events appear earlier and stronger."

That's a corpus claim backed by evidence, not just a bag of hard problems.
