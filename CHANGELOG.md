# Changelog

## v1.0.0 — 2026-06-21

Initial release.

- 7-phase SLURM lifecycle pipeline (assess → plan → draft → submit → health check → monitor → report)
- Pre-submission checklist with common failure modes and fixes
- Cluster reference notes for Georgia Tech PACE ICE (H100, `coe-gpu`, `coe-ice` QOS)
- Adaptive monitoring intervals based on job duration
- Error table covering: stale YAML paths, time limit violations, CUDA node collisions, numpy negative strides, missing checkpoints
- `/pace` slash command
