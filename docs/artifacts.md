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
      report.md
      plots/

  metrics_scalar/<run_id>.parquet
  metrics_wide/<run_id>.parquet
  geometry_state/<run_id>.parquet
  events_candidates/<run_id>.parquet
  scoring_baselines/<baseline_id>.json

  # Derived/consensus artifacts (typically cross-seed)
  events/<test_id>.parquet
  signatures/<run_id>.parquet
  matches/<run_id>.parquet
