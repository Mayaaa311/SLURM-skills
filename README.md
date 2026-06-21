# pace-skill

A Claude Code skill for end-to-end SLURM job management on GPU/HPC clusters.

Handles the full lifecycle: assess cluster resources → plan allocation → draft sbatch → submit → monitor logs → debug failures → report results.

## What it does

- **Assesses** your cluster before writing anything (`scontrol`, `sacctmgr`, `sinfo`)
- **Plans** GPU/CPU allocation and gives an ETA estimate before you submit
- **Drafts** sbatch scripts with a validated template and pre-submission checklist
- **Submits** jobs, captures IDs, handles parallel and chained (`--dependency`) submissions
- **Health-checks** within 90 seconds to catch immediate crashes before you walk away
- **Monitors** iteratively at intervals matched to job duration (minutes for short jobs, hours for overnight runs)
- **Debugs** common SLURM/GPU failures and resubmits automatically
- **Reports** a final summary table with status, duration, and output paths

## Install

```bash
# Clone somewhere stable
git clone https://github.com/<your-username>/pace-skill.git ~/pace-skill

# Install the skill globally
mkdir -p ~/.claude/skills
ln -s ~/pace-skill/pace ~/.claude/skills/pace

# Install the slash command globally
mkdir -p ~/.claude/commands
cp ~/pace-skill/commands/pace.md ~/.claude/commands/pace.md
```

The skill must be at `.claude/skills/pace/SKILL.md` for Claude Code to discover it.

## Usage

Just describe what you want — Claude picks up the skill automatically:

```
"Submit a training job for my model, 8192 steps on the full dataset"
"Check on my jobs"
"My job 5420485 crashed"
"How long will 16384 training steps take on an H100?"
```

Or invoke directly with the slash command:

```
/pace
```

## What's in the box

```
pace/
  SKILL.md          — the skill definition (7-phase pipeline)
commands/
  pace.md           — /pace slash command
```

No multi-agent ensemble needed — a single well-structured skill handles this workflow.

## Cluster coverage

Built and tested on **Georgia Tech PACE ICE** (H100, `coe-gpu` partition, `coe-ice` QOS).

The skill includes a generic checklist for any new cluster and notes which values to verify with `scontrol`/`sacctmgr` at the start of each session.

## Common failures handled

| Error | Fix encoded in skill |
|-------|---------------------|
| `FileNotFoundError` for YAML/config paths | Copy config locally, fix stale absolute paths |
| `Requested time limit is invalid` | Check partition MaxTime before setting `--time` |
| `CUDA error: device busy` | Transient node collision — resubmit |
| `negative stride in numpy array` | Add `.copy()` after `np.fliplr()` / `np.flipud()` |
| Missing checkpoint after training | Fix `find` glob path for checkpoint discovery |

## License

MIT
