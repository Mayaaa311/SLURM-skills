---
name: pace
description: "SLURM job lifecycle agent for GPU/HPC clusters. 7-phase pipeline: assess cluster resources → plan GPU/CPU allocation + ETA → draft sbatch → submit → early health check → iterative monitoring → report results. Handles parallel jobs, chained dependencies, and common failure modes. Triggers on: submit job, sbatch, slurm, run training, schedule gpu job, queue experiment, check jobs, monitor jobs, job failed, debug slurm."
metadata:
  version: "1.0.0"
  last_updated: "2026-06-21"
  status: active
  task_type: hpc-job-management
---

# PACE — SLURM Job Lifecycle Agent

End-to-end SLURM job management for GPU/HPC clusters. Covers the full lifecycle from resource assessment to confirmed successful completion, including debugging and resubmission.

---

## Trigger Conditions

Activate when the user wants to:
- Submit, queue, or schedule any job on a SLURM cluster
- Check, monitor, or debug running or failed SLURM jobs
- Write or fix an sbatch script
- Estimate GPU time or plan resource allocation

**Trigger keywords:** sbatch, slurm, squeue, gpu job, hpc job, submit training, queue experiment, check my jobs, job failed, job crashed, monitor jobs

---

## 7-Phase Pipeline

### Phase 1 — Assess Cluster Resources

Before writing any script, gather current cluster state:

```bash
# List partitions and their max wall times
scontrol show partition | grep -E "PartitionName|MaxTime"

# Check QOS limits for the user's account
sacctmgr show qos format=name,maxwall,maxjobspu,maxsubmitpu

# Check current user jobs
squeue --me --format="%.10i %.22j %.8T %.10M %.10l %R"

# Check GPU node availability
sinfo -p <partition> -o "%n %G %C %t" | head -20
```

**Establish before proceeding:**
- Max wall time per partition (never exceed — sbatch will reject the script silently or with an unhelpful error)
- Available GPU types and node count
- User's QOS: max concurrent jobs, max simultaneous submit
- Whether the requested job fits in one submission or needs chaining

**If cluster is unknown**, ask the user:
1. What cluster / institution?
2. What partition and account do they typically use?
3. Have they submitted jobs before? (if yes, check an existing sbatch for headers)

---

### Phase 2 — Plan Allocation and Estimate ETA

Based on Phase 1 findings, decide resource allocation and estimate runtime:

**GPU selection:**
| Workload | Recommended |
|----------|-------------|
| Large model training (>500M params) | 1× H100 or A100 |
| Inference / eval only | 1× A100 or V100 sufficient |
| Multi-GPU needed? | Only if model doesn't fit on one GPU — prefer single-GPU for simplicity |
| CPU-only | Use CPU partition; don't waste GPU allocation |

**ETA estimation:**
```
estimated_hours = training_steps / steps_per_hour

Reference throughputs (single GPU, batch=1):
  H100: ~900 steps/h  (large diffusion model, local adapter fine-tuning)
  A100: ~600 steps/h  (same workload estimate)
  V100: ~350 steps/h  (same workload estimate)

For eval/inference:
  ~30 images/h per GPU at 100 DDIM steps, 512px resolution
```

Add 20% buffer. Never set `--time` above the partition's MaxTime.

**Job structuring decisions:**
| Situation | Action |
|-----------|--------|
| Job fits in max wall time | Single sbatch |
| Job exceeds max wall time | Chain two jobs: `--dependency=afterok:JOBID` |
| Multiple independent jobs | Submit in parallel (separate sbatch calls) |
| Job B depends on Job A output | Chain: submit A first, capture JOBID, submit B with dependency |

Report to user:
- Estimated training/eval time
- Whether it fits in one job
- Recommended `--time` value
- Any flags for checkpointing if the job is long

---

### Phase 3 — Draft sbatch Script

**Canonical template:**

```bash
#!/bin/bash
#SBATCH --job-name=<short_descriptive_name>
#SBATCH --account=<account>
#SBATCH --partition=<partition>
#SBATCH --qos=<qos>
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=90G
#SBATCH --gres=gpu:<type>:1
#SBATCH --time=<HH:MM:SS>               # MUST be ≤ partition MaxTime
#SBATCH --output=<abs_path>/logs/%x_%j.out
#SBATCH --error=<abs_path>/logs/%x_%j.err

set -euo pipefail
set -x

# Environment setup
module load <module_name>
conda activate <env_path>
export PATH="<env_path>/bin:$PATH"
export PYTHONDONTWRITEBYTECODE=1
export PYTHONPYCACHEPREFIX="/tmp/${USER}_pycache_${SLURM_JOB_ID}"
export OMP_NUM_THREADS=1
export MKL_NUM_THREADS=1

# Sanity check
python -c "import torch; print('ENV OK', torch.__version__, 'CUDA:', torch.cuda.is_available())"

mkdir -p <output_dir>

# --- Your job commands here ---
```

**Pre-submission checklist — verify every item before submitting:**
- [ ] `--time` ≤ partition MaxTime (run Phase 1 to confirm)
- [ ] `--output` and `--error` log dirs exist (or are created with `mkdir -p` inside the script)
- [ ] All file paths are **absolute** (not relative)
- [ ] Config/YAML `anno_path`, `data_path`, etc. point to the actual files — if dirs were reorganized, paths in old configs are stale
- [ ] Resume checkpoint exists if doing `--resume` or similar
- [ ] Output dirs created before any write commands run

---

### Phase 4 — Submit

```bash
# Single job — capture ID for monitoring
JOB_ID=$(sbatch --parsable path/to/run.sbatch)
echo "Submitted: $JOB_ID"

# Multiple parallel jobs
JOB_A=$(sbatch --parsable job_a.sbatch) && echo "A: $JOB_A"
JOB_B=$(sbatch --parsable job_b.sbatch) && echo "B: $JOB_B"

# Chained jobs (B only starts if A succeeds)
JOB_A=$(sbatch --parsable job_a.sbatch)
JOB_B=$(sbatch --parsable --dependency=afterok:${JOB_A} job_b.sbatch)
echo "Chained: A=$JOB_A → B=$JOB_B"
```

Record all job IDs before moving on.

---

### Phase 5 — Early Health Check (60–120 seconds after submit)

After ~90 seconds, confirm the jobs aren't immediately crashing:

```bash
# Check queue state — all submitted IDs should show RUNNING or PENDING
squeue --me --format="%.10i %.22j %.8T %.10M %.10l %R"

# Tail error log for crash signals
tail -20 <log_dir>/<jobname>_<jobid>.err

# Tail output log for progress signals
tail -10 <log_dir>/<jobname>_<jobid>.out
```

**Green signals (job is healthy):**
- State shows `RUNNING` or `PENDING` in squeue
- `.err` has only deprecation warnings, no tracebacks
- `.out` shows environment setup lines (`ENV OK`, module loads), then job-specific output beginning

**Red signals (immediate failure):**
- Job disappears from queue within 2 minutes
- `.err` ends with a Python/Bash traceback

**Common failure modes and fixes:**

| Error message | Root cause | Fix |
|---------------|-----------|-----|
| `FileNotFoundError: <config_path>` | Config YAML has stale path from before a directory reorganization | Copy YAML locally to experiment dir, update all absolute paths, point sbatch CONFIG var to the new copy |
| `Requested time limit is invalid` | `--time` exceeds partition MaxTime | Run `scontrol show partition` to find actual limit, lower `--time` |
| `CUDA error: device busy or unavailable` | Multiple jobs landed on same single-GPU node simultaneously | Transient — resubmit; SLURM will assign a different node |
| `negative stride in numpy array` | `np.fliplr()` / `np.flipud()` returns a view with negative strides; PyTorch rejects it | Add `.copy()` after the flip operation: `np.fliplr(arr).copy()` |
| `No such file or directory: last.ckpt` | Training was killed before a checkpoint saved, or `find` glob path is wrong | Check checkpoint dir manually; fix the find command; resubmit |
| `sbatch: invalid partition name` | Partition name typo or wrong cluster | Run `scontrol show partition` to list valid names |
| `sbatch: Job violates accounting/QOS policy` | Over QOS limit (too many jobs, too much time) | Check `sacctmgr show qos`; reduce concurrent submissions or `--time` |

If any job fails, fix the issue and go back to Phase 4 to resubmit that job only.

---

### Phase 6 — Iterative Monitoring

Choose check interval based on estimated job duration:

| Estimated duration | Check interval |
|--------------------|---------------|
| < 30 min | Every 5–10 min |
| 30 min – 2h | Every 15–20 min |
| 2h – 8h | Every 30–60 min |
| 8h+ / overnight | Check before sleeping, check in the morning |

**Per-check routine:**

```bash
# 1. Queue overview
squeue --me --format="%.10i %.22j %.8T %.10M %.10l %R"

# 2. For RUNNING jobs — tail progress output
tail -5 <log_dir>/<jobname>_<jobid>.out

# 3. For jobs that left queue — check outcome
tail -30 <log_dir>/<jobname>_<jobid>.out
tail -20 <log_dir>/<jobname>_<jobid>.err
```

**Success indicators in `.out`:**
- Training: loss values appearing at each logged step, checkpoint save message near the end
- Eval/inference: `"done — N images written"` or equivalent completion line
- Script: clean exit, no error keywords

**Failure indicators:**
- Job left queue in < 30% of its wall time → almost certainly crashed, not completed
- `.err` ends with a traceback
- `.out` ends mid-run without a completion message

On failure: diagnose from Phase 5 error table, fix, resubmit.

---

### Phase 7 — Report Results

Once all jobs complete, produce a summary table:

```
| Job ID | Name               | Status  | Duration | Key Output |
|--------|--------------------|---------|----------|------------|
| 12345  | 1A_full12k_16k     | SUCCESS | 13.2h    | eval_staged/generated/ (30 imgs) |
| 12346  | 2A_10var_cont8k    | SUCCESS | 4.1h     | eval_staged/generated/ (30 imgs) |
| 12347  | 1B_flip            | FAILED  | 0.1h     | numpy stride bug → fixed, resubmitted as 12350 |
| 12350  | 1B_flip (retry)    | RUNNING | —        | — |
```

Point user to output directories and any generated artifacts (images, checkpoints, contact sheets, logs).

---

## Cluster Reference Notes

These values should be verified with `scontrol`/`sacctmgr` at the start of each session — cluster configs change.

**Georgia Tech PACE ICE (as of 2026-06):**
- Partition `coe-gpu`: MaxTime = 16:00:00, H100 GPUs
- QOS `coe-ice`: used with `coe-gpu`
- GPU request: `--gres=gpu:h100:1`
- Safe step budget per 16h job: ≤ 12,288 steps (leaves ~15% buffer)
- Conda env: `/home/hice1/yyuan394/scratch/diffusion`
- Checkpoint storage: `/storage/ice-shared/bmed-sp-wang/yining/checkpoints/`
- YAML config trap: if experiment dirs were moved into a dated subfolder, YAML `anno_path` fields still point to the old flat path — always copy the YAML into the new experiment's `configs/` dir and fix all absolute paths

**Generic checklist for a new cluster:**
1. `scontrol show partition` — find partition names and MaxTime
2. `sacctmgr show qos` — find your QOS and its limits
3. `sinfo` — find GPU types available
4. Check an existing sbatch from a colleague for account/partition/qos headers
