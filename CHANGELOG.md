# Changelog

## v1.1.0 — 2026-06-30

- Added **Parallel Execution Patterns** section: independent parallel sbatch, job arrays with `%` throttling, parallel-then-chained pipelines, QoS-for-parallelism table, and guardrails for embarrassingly parallel workloads.
- Added **QoS Reference** table (account `cse`): `coe-ice` (16h, 500 queued, no concurrent cap), `coe-grade`/`pace-grade` (24h, 10 jobs), GPU inventory (H100 ×152, H200 ×144), and re-verify commands.

## v1.0.0 — 2026-06-21

Initial release.

- 7-phase SLURM lifecycle pipeline (assess → plan → draft → submit → health check → monitor → report)
- Pre-submission checklist with common failure modes and fixes
- Cluster reference notes for Georgia Tech PACE ICE (H100, `coe-gpu`, `coe-ice` QOS)
- Adaptive monitoring intervals based on job duration
- Error table covering: stale YAML paths, time limit violations, CUDA node collisions, numpy negative strides, missing checkpoints
- `/pace` slash command
