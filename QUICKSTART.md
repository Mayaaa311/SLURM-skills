# Quick Start

## Step 1: Install

```bash
git clone https://github.com/<your-username>/pace-skill.git ~/pace-skill

mkdir -p ~/.claude/skills
ln -s ~/pace-skill/pace ~/.claude/skills/pace

mkdir -p ~/.claude/commands
cp ~/pace-skill/commands/pace.md ~/.claude/commands/pace.md
```

## Step 2: Launch

```bash
claude
```

## Step 3: Use it

Tell Claude what you want:

```
"Submit a training job — 8192 steps, full dataset"
```

```
"Check on my jobs"
```

```
"My job crashed, job ID 5420485"
```

Or trigger directly: `/pace`

## What happens

Claude runs the 7-phase pipeline:

1. Checks your cluster (partitions, QOS, GPU availability)
2. Plans resources and gives you an ETA
3. Drafts the sbatch script
4. Submits and captures job IDs
5. Health-checks after ~90 seconds
6. Monitors at intervals until done
7. Reports final status + output paths
