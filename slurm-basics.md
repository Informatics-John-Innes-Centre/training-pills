# Slurm Basics

A short introduction to running jobs on an HPC cluster with Slurm.

## What is Slurm?

Slurm is the **job scheduler** used on the cluster. Instead of running your
analysis directly on a login node (shared by everyone, and not meant for
heavy work), you submit it as a **job**, and Slurm queues it and runs it on
one of the compute nodes when resources are free.

## Key concepts

- **Job** — a piece of work you submit (a script, or a single command).
- **Partition** — a named group of nodes with its own limits (e.g. maximum
  walltime). You choose one when submitting.
- **Node** — a physical machine in the cluster.

## The commands you'll actually use

| Command | What it does |
| --- | --- |
| `sbatch script.sh` | Submit a job script to the queue |
| `squeue -u $USER` | See your jobs and their status (pending/running) |
| `scancel <jobid>` | Cancel a job |
| `sacct -j <jobid>` | See how a finished job used its resources |
| `srun --pty bash` | Start an interactive session on a compute node |

## A minimal job script

```bash
#!/bin/bash
#SBATCH --job-name=my-analysis
#SBATCH --partition=short
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=01:00:00
#SBATCH --output=my-analysis-%j.log

echo "Running on $(hostname)"
python my_script.py
```

Submit it with:

```bash
sbatch script.sh
```

The `#SBATCH` lines are resource requests, not comments Slurm ignores —
they tell the scheduler what your job needs before it runs.

## Checking on a job

```bash
squeue -u $USER          # is it queued or running?
tail -f my-analysis-%j.log   # watch the output live
```

Once it finishes, `sacct -j <jobid> --format=JobID,Elapsed,MaxRSS,State`
shows you how long it took and how much memory it actually used — handy for
sizing your next request more accurately.

## A few habits worth building

- **Ask for what you need, not more.** Over-requesting CPUs/memory just
  means your job waits longer in the queue for resources it won't use.
- **Give your job a sensible `--time` limit.** Too short and it gets killed
  before finishing; too long and it may queue longer than necessary.
- **Check `sacct` after a job finishes**, not just whether it completed —
  it tells you whether your resource request actually matched what you used.
