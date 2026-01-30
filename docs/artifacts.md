data/runs/
  runs/<run_id>/
    meta.json
    checkpoints/ (optional in Scout)
    captures/
      step_000100/
        resid_layer_00.pt
        resid_layer_06.pt
        embed.pt
      ...
    reports/
      <analysis_id>/           # Versioned analysis outputs
        report.md
        llm_analysis.json      # Optional LLM analysis
      plots/

  metrics_scalar/<run_id>.parquet
  metrics_wide/<run_id>.parquet
  geometry_state/<run_id>.parquet
  events_candidates/<run_id>/<analysis_id>.parquet   # Versioned event detection
  detection_summary/<run_id>/<analysis_id>.parquet   # Detection diagnostics
  scoring_baselines/<baseline_id>.json

  # Derived/consensus artifacts (typically cross-seed)
  events/<test_id>.parquet     # Consensus events from multi-seed test
  signatures/<run_id>.parquet
  matches/<run_id>.parquet


# Test (Multi-Seed) Artifacts
# A "test" is a collection of runs that are identical except for seed.
# See design_and_contracts.md Section 2.5 for terminology.

data/tests/
  tests/<test_id>/
    test.yaml                  # Test manifest (seeds, config, status, summary)
    test.log                   # Execution log
    outputs/
      consensus_report.md      # Main comparison report (seed-invariant focus)
      consensus_report.llm_analysis.json  # Optional LLM analysis
      plots/
        effective_rank_layer_*.png
        raster_*.png

# Test Manifest Schema (test.yaml)
#
# test_id: test_scout_tiny_20250129_143000
# config_path: configs/scout_tiny.yaml
# config_hash: abc123...
# seeds: [42, 123, 456, 789, 1011]
# created_at: 2025-01-29T14:30:00Z
#
# detection_params:
#   warmup_fraction: 0.1
#   max_pre_warmup: 1
#   peak_suppression_radius: 15
#   max_events_per_series: 5
#   adaptive_k: 2.5
#   step_tolerance: 5
#
# runs:
#   - seed: 42
#     run_id: 20250129_143000_scout_tiny_s42
#     status: success
#     analysis_id: w10_p1_r15_e5_k25
#   - seed: 123
#     run_id: 20250129_143500_scout_tiny_s123
#     status: success
#     analysis_id: w10_p1_r15_e5_k25
#
# summary:
#   total_seeds: 5
#   successful: 5
#   failed: 0
#   consensus_events: 23
#   jaccard_similarity: 0.67
#   confidence: high
