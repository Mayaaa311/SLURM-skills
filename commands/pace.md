---
description: PACE SLURM agent — full job lifecycle (assess → plan → draft → submit → monitor → debug → report)
---

Trigger the `pace` skill. Runs the 7-phase SLURM job lifecycle pipeline:
1. Assess cluster resources (partitions, QOS, GPU availability)
2. Plan allocation and estimate ETA
3. Draft sbatch script
4. Submit job(s)
5. Early health check (~90s after submit)
6. Iterative monitoring until completion
7. Report results

Skill entry: `pace/SKILL.md`
