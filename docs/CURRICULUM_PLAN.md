# CURRICULUM_PLAN.md

This document defines a flexible, reproducible way to implement **curriculum-driven training** over an already-classified problem pool. It is designed to be passed to an LLM in an IDE to guide implementation.

**Non-goals**
- This plan does not assume anything about the current training code, repo structure, or experiment runner beyond what is explicitly described here.
- This plan does not require pre-materializing full "ordered datasets" unless you choose to for debugging/replay.

---

## 1) Data Contract

### 1.1 Classified item schema (JSONL)

Assume a JSONL file (e.g., `items_fully_classified.jsonl`) where each line has:

```json
{
  "item_id": "omr:cot:1",
  "family_id": "bijection_counting",
  "content": {
    "problem": "...",
    "final_answer": "..."
  },
  "hashes": {
    "content_sha256": "..."
  },
  "provenance": {
    "generated_solution": "..."
  }
}
```

### 1.2 Required fields (minimum)

- `item_id`: stable identifier
- `family_id`: classifier output (string)
- `content.problem`: prompt/problem text
- `hashes.content_sha256`: stable content hash (recommended for dedup / replay)

### 1.3 Optional-but-strongly-recommended derived fields

Compute once during ingestion/post-process and store alongside each item:

- `token_len`: tokenized length (or char length if tokenizer not available at ingestion time)
- `symbol_density`: simple proxy (count of non-alnum symbols / token_len)
- `difficulty_proxy`: bucketed value computed from heuristics (see §6)

Store these either:
- embedded into JSONL records, or
- in a sidecar table keyed by `item_id` or `content_sha256`.

---

## 2) Why "Curriculum = Sampling Policy" (core design)

A curriculum should be implemented as a **time-varying sampling policy**, not as a static reordered file. Concretely:

> A curriculum maps training time `t` (step or epoch fraction) → a distribution over families and/or items.

This makes curricula:
- easy to A/B test
- easy to version
- reproducible via manifests
- compatible with large pools (streaming, sharding, etc.)

---

## 3) Artifacts and File Layout (recommended)

### 3.1 Candidate pool (immutable)

```
data/candidate_pool/items_fully_classified.jsonl
data/candidate_pool/index/ (generated once)
  - families.json (counts, family list)
  - ids_by_family_train.json (or parquet)
  - ids_by_family_val_random.json
  - ids_by_family_val_family.json
  - optional: difficulty_bins.json (ids by bin)
  - optional: dedup_map.json (hash → canonical id)
```

### 3.2 Splits (immutable)

```
data/candidate_pool/splits/split_vX_seedY/
  - train_ids.txt
  - val_random_ids.txt
  - val_family_ids.txt
  - split_manifest.json (seed, rules, family holdouts)
```

### 3.3 Curricula (versioned)

```
curricula/
  - iid_balanced.yaml
  - family_blocked.yaml
  - family_ramp.yaml
  - difficulty_ramp.yaml
  - custom_*.yaml
```

### 3.4 Run manifests (written per run)

```
runs/<run_id>/curriculum_manifest.json
runs/<run_id>/sample_trace.parquet (or jsonl)
  columns: step, epoch, phase, family_id, item_id, content_sha256 (optional)
```

---

## 4) Curriculum DSL (YAML Spec)

### 4.1 Schema

A curriculum config should define:

- **global defaults**: sampling mode, replacement, weights normalization, etc.
- **time_unit**: `steps` or `epochs`
- **phases**: ordered list with:
  - time span
  - allowed families
  - weights / mixing
  - optional filters (difficulty/length)
  - transition (hard / ramp)

### 4.2 Example DSL (illustrative)

```yaml
version: 1
name: family_ramp_v1
time_unit: steps

defaults:
  sampling:
    mode: balanced_family          # options: balanced_family, proportional_family, uniform_item
    replacement: true
    family_weight_floor: 0.0       # avoid zeroing unless explicitly excluded
  filters:
    token_len:
      min: null
      max: null
    difficulty_proxy:
      allowed: null

phases:
  - name: phase_1_targets_only
    start: 0
    end: 0.30                     # fraction of total steps if end <= 1.0
    families:
      include: [functional_equations, bijection_counting, modular_arithmetic, graph_recurrence, invariant_monovariant, cyclic_groups]
    weights:
      type: uniform               # uniform or explicit
    sampling:
      mode: balanced_family

  - name: phase_2_add_controls_ramp
    start: 0.30
    end: 0.70
    families:
      include: [functional_equations, bijection_counting, modular_arithmetic, graph_recurrence, invariant_monovariant, cyclic_groups,
                geometry_euclidean, calculus_analysis, optimization, algebra_polynomials, misc_other, sequences_series,
                probability_expectation, combinatorics_basic, inequalities, number_theory_basic, logic_proof]
    weights:
      type: ramp
      ramp:
        from:
          functional_equations: 1.0
          bijection_counting: 1.0
          modular_arithmetic: 1.0
          graph_recurrence: 1.0
          invariant_monovariant: 1.0
          cyclic_groups: 1.0
          geometry_euclidean: 0.0
          calculus_analysis: 0.0
          optimization: 0.0
          algebra_polynomials: 0.0
          misc_other: 0.0
          sequences_series: 0.0
          probability_expectation: 0.0
          combinatorics_basic: 0.0
          inequalities: 0.0
          number_theory_basic: 0.0
          logic_proof: 0.0
        to:
          geometry_euclidean: 1.0
          calculus_analysis: 1.0
          optimization: 1.0
          algebra_polynomials: 1.0
          misc_other: 1.0
          sequences_series: 1.0
          probability_expectation: 1.0
          combinatorics_basic: 1.0
          inequalities: 1.0
          number_theory_basic: 1.0
          logic_proof: 1.0

  - name: phase_3_full_mix
    start: 0.70
    end: 1.0
    families:
      include: "*"
    sampling:
      mode: proportional_family
```

**Notes:**
- Use fractional phase boundaries (0–1) so the same curriculum spec works for different run lengths.
- `sampling.mode` can be overridden per phase.

---

## 5) Sampler Architecture

### 5.1 Core interfaces

Implement the following components:

**CurriculumSpec**
- Parse/validate YAML
- Convert phase boundaries into absolute step spans given `total_steps`
- Provide `phase_at_step(step)`

**CurriculumSampler**
- Inputs:
  - `spec`: CurriculumSpec
  - `split_ids`: set[item_id] (train)
  - `index_by_family`: dict[family_id, list[item_id]] (restricted to split)
  - `rng_seed`: int
- Method: `sample(step, batch_size) -> list[item_id]`
  1. Determine current phase
  2. Compute family weights (uniform/proportional/ramp/custom)
  3. Draw families (multinomial)
  4. Draw items from each family list (uniform among items within family, unless filtering)
  5. Return item ids

**TraceWriter** (optional but recommended)
- Writes sample_trace with deterministic replay: `(step, phase, family_id, item_id, sha256)`
- When enabled, the run becomes exactly reproducible even if later indexing changes.

### 5.2 Sampling modes

Provide these modes initially:

- **balanced_family**: Choose family uniformly among included families, then choose an item uniformly within that family. Prevents large families from dominating.
- **proportional_family**: Choose family proportional to available items (within split), then choose item. Matches underlying distribution.
- **uniform_item**: Choose uniformly from all items in allowed set (ignores family balancing).

### 5.3 Replacement behavior

- `replacement=true` recommended for simplicity and stable mixing.
- If `replacement=false`, the sampler becomes epoch-aware and must manage per-family queues.

---

## 6) Difficulty & Other Filters (optional, high value)

Add low-cost proxies; do NOT block implementation on perfect "difficulty."

### 6.1 Suggested proxies (cheap)

- `token_len` (or char length)
- `symbol_density`
- `rare_token_rate` (if tokenizer available)
- `numeric_density` (#digits / token_len)
- `operator_count` (e.g., + - * / ^ = count)

### 6.2 Difficulty bins

Compute:
- `difficulty_proxy` ∈ {easy, medium, hard} (or 0..N bins)

Example rule:
- **easy**: bottom 33% by (token_len + 2*symbol_density)
- **medium**: middle
- **hard**: top 33%

Use in curricula:
- Phase 1: easy only
- Phase 2: easy + medium
- Phase 3: all

---

## 7) Curricula to Implement First (minimal set)

Implement these four specs; they cover most research questions with minimal complexity:

1. **IID baseline**
   - Single phase, `proportional_family` (or `balanced_family`)

2. **Balanced baseline**
   - Single phase, `balanced_family`
   - Useful given skewed family distributions

3. **Family-blocked**
   - Phases per family: train on one family at a time (hard switch)
   - Useful for testing "family-specific reorganization"

4. **Family-ramp**
   - Start with subset (e.g., "target families"), then ramp in "controls," then full mix
   - Useful for measuring phase transitions and squiggle topology changes

**Optional 5th:**
- Difficulty-ramp (using proxy bins)

---

## 8) Handling Skewed Family Distributions

Family distributions can be highly imbalanced (some families large, some small). Support skew explicitly:

- Use `balanced_family` when you want structure-learning comparisons and to prevent dominant families from controlling training.
- Use `proportional_family` when you want realistic mixture matching the pool.
- Add optional safeguards:
  - `min_items_per_family` threshold for inclusion (otherwise exclude or downweight)
  - `max_sampling_fraction_per_family` cap

---

## 9) Reproducibility and Versioning

### 9.1 Split manifest

Write `split_manifest.json` containing:
- split version, seed
- how `val_family` families were chosen
- counts per family in each split
- hashes for the ID lists

### 9.2 Curriculum manifest

Write `curriculum_manifest.json` containing:
- curriculum YAML (embedded or referenced by git hash)
- resolved absolute phase boundaries (steps)
- effective family weights per phase (resolved)
- RNG seeds used

### 9.3 Sample trace (optional but ideal)

Write `sample_trace.parquet` if you need exact replay:
- allows you to reproduce identical training stream even if the pool grows later

---

## 10) Integration Plan (step-by-step)

1. **Index build**
   - Read JSONL once
   - Build:
     - `family -> [item_id]`
     - `item_id -> sha256`
     - optional: difficulty proxies

2. **Split**
   - Produce split ID lists (train / val_random / val_family)
   - Store as `split_vX_seedY/`

3. **Sampler**
   - Load `split/train_ids`
   - Restrict `index_by_family` to only those ids
   - Initialize `CurriculumSampler(spec, seed)`

4. **Training loop integration**
   - Instead of "random batch from train," call:
     ```python
     batch_ids = sampler.sample(global_step, batch_size)
     ```
   - map IDs → content (problem text) → tokenizer

5. **Manifests**
   - Write curriculum manifest at run start
   - Optionally write sample trace continuously

---

## 11) Validation Checks

**Before training:**
- Verify each phase has non-empty eligible items
- Verify family weights sum and normalize as expected
- Print per-phase "effective distribution":
  - family counts available
  - target weights vs realized weights

**During training (optional logging):**
- Every N steps, log:
  - realized family histogram in last window
  - realized difficulty histogram (if used)

**After training:**
- Save realized histograms and attach to run report for auditability.

---

## 12) Notes on Streaming from HuggingFace

This curriculum design works whether you:
- store locally first (recommended for reproducibility), or
- stream on-demand

If you remain streaming-only:
- you must still maintain a stable `item_id` list and per-family indexing
- exact replay becomes harder unless you store sample traces and content hashes

**Recommendation:**
- Keep local candidate pool as the stable "inventory"
- Use HF streaming only for acquisition + classification expansion

---

## 13) Implementation Checklist (for the IDE LLM)

- [ ] Create `CurriculumSpec` with YAML parsing + phase resolution
- [ ] Create `CurriculumSampler` supporting balanced/proportional/uniform sampling
- [ ] Add optional filters (family include/exclude, token_len, difficulty_proxy)
- [ ] Add `split_manifest.json` writer
- [ ] Add `curriculum_manifest.json` writer
- [ ] Add optional `sample_trace.parquet` writer
- [ ] Add per-phase distribution sanity output
- [ ] Add 4 initial YAML curricula in `curricula/`

---

## 14) Minimal "Done" Definition

The curriculum system is considered complete for initial research when:

1. A run can be configured with `--curriculum curricula/family_ramp.yaml`
2. The sampler produces batches that follow phase rules
3. A curriculum manifest is written and can be replayed deterministically (at least by seed, ideally by sample trace)
4. Training logs can report realized family distributions per phase
