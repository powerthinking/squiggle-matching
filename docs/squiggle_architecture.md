# Squiggle Architecture
## A Basis-Invariant Framework for Representational Dynamics and Matching

---

## 0. Purpose of This Document

This document defines the **conceptual architecture** of the Squiggle framework.

Its goal is to:
- Prevent conceptual drift as individual components evolve
- Separate **what exists** from **how it is measured** and **how it is used**
- Provide a stable reference for both:
  - the **Math Corpus Data Prize**
  - the **long-term Squiggle Matching research program**

This is a **design contract**, not an implementation guide.

---

## 1. Canonical Definition (Ontology – Frozen)

### 1.1 Squiggle (Canonical Definition)

A **squiggle** is a *compressed, basis-invariant representation of the temporal evolution of geometric descriptors of a neural representation*, capturing **state**, **dynamics**, and **events**, summarized as a compact signature suitable for comparison and matching.

Key properties:
- Temporal
- Geometric (not coordinate-based)
- Basis-invariant
- Comparable across contexts

A squiggle represents **how representations reorganize**, not their final state.

---

## 2. Squiggle Ontology: Three-Layer Structure (Frozen)

A squiggle is composed of three logically distinct layers.

---

### 2.1 Layer A — Geometric State (Snapshot Geometry)

Describes the structure of the representation at a given time `t`.

#### A1. Subspace Structure
- Effective rank
- Singular value spectrum / entropy
- Activation sparsity
- Principal angles (e.g., SVCCA, CKA, local variants)

#### A2. Orientation and Alignment
- Subspace alignment across time
- Cross-layer similarity
- Rotation angle relative to previous window

#### A3. Shape and Complexity
- Subspace curvature
- Local condition number
- Spectral concentration / diagonalization

Layer A is **descriptive**, not evaluative.

---

### 2.2 Layer B — Dynamics (Change Over Time)

Describes *how* geometry evolves.

#### B1. Velocity (1st Derivative)
- Drift velocity
- Alignment velocity

#### B2. Acceleration (2nd Derivative)
- Drift acceleration
- Curvature change

#### B3. Volatility
- Magnitude volatility
- Velocity volatility
- Curvature volatility

Layer B captures **motion**, not interpretation.

---

### 2.3 Layer C — Event Encoding (Discrete Structure)

Layer C compresses continuous geometry into **symbolic events**.

Canonical event types include:
- Rank collapse
- Sudden alignment
- Curvature sign change
- Volatility spike
- Subspace migration
- Phase transition (regime change)

Events are **derived**, not primitive.
They are the bridge from geometry to comparison.

---

### 🔒 Ontology Freeze Rule
Layers A, B, and C define *what a squiggle is*.  
They **must not change** for the Math Corpus submission.

---

## 3. Measurement Layer (Implementation – Evolving)

This layer defines **how ontology is instantiated numerically**.

Examples:
- How effective rank is computed
- Which layers are sampled
- Window sizes and smoothing
- Approximate vs exact SVD
- Proxy metrics for sparsity or curvature

Measurement choices:
- May evolve
- Must preserve semantic meaning of Layers A/B/C
- Must be documented when changed

Measurement ≠ ontology.

---

## 4. Event Extraction Layer (Rules – Evolving)

Event extraction converts A+B into C.

Examples:
- Threshold-based detection
- Change-point detection
- Spectral ratio crossings
- Volatility envelope breaches

Event extraction rules:
- Are heuristic
- Are revisable
- Must preserve **event semantics**, not numeric thresholds

---

## 5. Squiggle Matching Semantics (Frozen)

Squiggle Matching defines **when two squiggles are considered similar**.

Matching is semantic, not numeric.

---

### 5.1 Matching Axes

#### Axis 1 — Event Topology
- Same ordering of events
- Same qualitative transitions
- Timing differences allowed

Example:
collapse → stabilization → re-expansion


---

#### Axis 2 — Dynamic Profile
- Similar velocity / acceleration patterns
- Similar volatility envelopes
- Similar curvature signatures

---

#### Axis 3 — Invariance Class
Similarity must survive:
- Basis change
- Projection
- Mild noise

Similarity does *not* require:
- Same layer
- Same task
- Same modality

---

### 🔒 Matching Freeze Rule
Matching semantics define **what similarity means** and must remain stable.

---

## 6. Discovery Pipeline: Candidate Gates (Procedural – Evolving)

Gates define **how squiggles are discovered and filtered**, not what they are.

---

### Gate 1 — Identification (Fast, Noisy)
Purpose: detect potential events.

Signals:
- Rank or spectral energy shifts
- Singular value ratio changes
- Alignment deltas
- Drift magnitude
- Volatility spikes
- Regime shifts

Output: candidate windows.

---

### Gate 2 — Supporting Signal (Stability)
Purpose: reject noise.

Checks:
- Repeatability across seeds (time-warp tolerant)
- Synchronization across layers
- Task or curriculum linkage

---

### Gate 3 — Prototype Filters (Typing)
Purpose: classify event type.

Examples:
- Collapse-like
- Re-expansion-like
- Migration-like
- Bifurcation-like
- Volatility-only (likely noise)

---

### Gate 4 — Condition Correspondence & Causal Impact
Purpose: connect events to learning.

Questions:
- Does the event depend on a condition?
- Presence vs absence
- Timing shifts
- Stability differences
- Downstream performance impact

---

## 7. Scoring and Ranking (Operational – Evolving)

Scoring is **not part of squiggle identity or matching**.

It is used to **prioritize attention**.

---

### 7.1 Candidate Ranking Score

Score = Magnitude × Coherence × Novelty


- **Magnitude**: size of geometric change
- **Coherence**: consistency across seeds/layers/samples
- **Novelty**: distance from prior signatures

Scoring ranks **instances**, not **types**.

---

### 🔒 Scoring Boundary Rule
Scoring must not redefine:
- Events
- Matching
- Ontology

---

## 8. Application Profiles

### 8.1 Math Corpus Data Prize

Required:
- Coarse Layer A/B metrics
- Subset of Layer C events
- Gates 1–2
- Scoring for selection

Not required:
- Full invariance analysis
- Cross-task matching
- Event grammars

Focus: **data selection via representational impact**.

---

### 8.2 Core Squiggle Matching Research

Future extensions:
- Event grammars
- Cross-task and cross-model matching
- Archetypal squiggle libraries
- Causal graphs of learning events

Focus: **theory of learning dynamics**.

---

## 9. Architectural Summary

| Layer | Role | Stability |
|-----|-----|-----|
| Ontology (A/B/C) | What exists | Frozen |
| Measurement | How it’s computed | Evolving |
| Event Extraction | How events are detected | Evolving |
| Matching Semantics | What similarity means | Frozen |
| Gates | Discovery workflow | Evolving |
| Scoring | Prioritization | Evolving |

---

## 10. Guiding Principle

> **Squiggles are about structure, not scores.  
Scores help us decide where to look.**

This separation is what allows the framework to scale without collapsing.
