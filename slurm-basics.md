# Slurm Basics for Bioinformatics

## Running jobs on a shared HPC cluster

**Time:** about 40–50 minutes
**Level:** Beginner — no prior Slurm experience needed

By the end of this tutorial, you will be able to:

- Explain why jobs are submitted through Slurm instead of run directly
- Load the software your job needs before it runs
- Write and submit a job script
- Check on a running job and read its output
- Start an interactive session for testing and debugging
- Process many samples at once with a job array
- Chain pipeline steps together with job dependencies
- Check how much CPU/memory a finished job actually used
- Diagnose the most common reasons a job fails

The main idea to take away:

> **The login node is for editing and submitting; the compute nodes are for the actual work. Slurm is what connects the two, queuing your job until the resources you asked for are free.**

---

## 1. Why Slurm?

The login node is shared by everyone at once. Running a real analysis directly on it slows the whole cluster down for other people, and it isn't sized for heavy work anyway.

Instead, you describe what your job needs (CPUs, memory, time) and hand it to Slurm, which queues it and runs it on a compute node once those resources are free:

```text
You                     Slurm                    Compute nodes
 │                        │                            │
 │   sbatch script.sh     │                            │
 ├───────────────────────►│                            │
 │                        │   queued until resources    │
 │                        │   are free, then runs it   │
 │                        ├───────────────────────────►│
 │                        │                            │  job runs here
 │   squeue -u $USER      │                            │
 ├───────────────────────►│                            │
```

## 2. Key concepts

- **Job** — a piece of work you submit: a script, or a single command.
- **Partition** — a named group of nodes with its own limits (e.g. maximum walltime, node type). You choose one when submitting.
- **Node** — a physical machine in the cluster.
- **Task/array index** — when you submit a *job array* (Section 7), each individual run is one task, identified by an array index.

## 3. Loading the software your job needs

A compute node starts with a bare environment — it doesn't automatically have the same software loaded as your interactive login shell. Load whatever your job depends on inside the script itself, not just in your terminal beforehand.

**Environment modules** (common on many HPCs)

```bash
module load samtools/1.19
module load python/3.11
```

**Or a conda/mamba environment**

```bash
source ~/miniconda3/etc/profile.d/conda.sh
conda activate rnaseq-env
```

> **Why this matters:** a job that works when you test a command interactively can still fail under `sbatch` if the script itself never loads the right module or environment — the compute node doesn't inherit it automatically.

## 4. A minimal job script

```bash
#!/bin/bash
#SBATCH --job-name=my-analysis
#SBATCH --partition=short
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=01:00:00
#SBATCH --output=my-analysis-%j.log

module load python/3.11

echo "Running on $(hostname)"
python my_script.py
```

The `#SBATCH` lines are resource requests, not comments Slurm ignores — they tell the scheduler what your job needs before it runs. `%j` in the output filename is replaced with the job ID, so repeated runs don't overwrite each other's logs.

**Submit it**

```bash
sbatch script.sh
```

```text
Submitted batch job 123456
```

## 5. Checking on a job

```bash
squeue -u $USER              # is it queued (PD) or running (R)?
tail -f my-analysis-123456.log   # watch the output live
```

`squeue` output includes a `ST` (state) column — `PD` means pending (still queued), `R` means running. If a job stays `PD` for a long time, `squeue --start -j <jobid>` gives Slurm's estimate of when it'll begin.

**Cancel a job**

```bash
scancel 123456
```

## 6. Interactive sessions for testing and debugging

Before wrapping something in an `sbatch` script, it's often faster to test it interactively on a compute node:

```bash
interactive
```

This puts you on a compute node with an interactive shell, so you can run commands one at a time, check that paths and modules are correct, and debug errors as they happen — rather than waiting for a queued batch job to fail and then reading the log.

> **Keep it alive:** an interactive session is tied to your terminal, so it dies if your connection drops. Run it inside `tmux` (or `screen` on a data mover node) so you can detach and reattach safely — see the `tmux-basics` pill.

## 7. Job arrays — one script, many samples

Bioinformatics work is usually "run the same thing on every sample." Rather than writing 96 nearly-identical `sbatch` scripts, submit one **job array**:

```bash
#!/bin/bash
#SBATCH --job-name=align
#SBATCH --partition=short
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=01:00:00
#SBATCH --array=1-96%20
#SBATCH --output=align-%A_%a.log

SAMPLE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" samples.txt)

module load bwa samtools
bwa mem ref.fa "${SAMPLE}_R1.fastq.gz" "${SAMPLE}_R2.fastq.gz" \
  | samtools sort -o "${SAMPLE}.bam"
```

- `--array=1-96%20` submits 96 tasks (indices 1–96), but runs at most 20 at a time — considerate to other users sharing the cluster.
- `$SLURM_ARRAY_TASK_ID` is the one thing that differs between tasks — here it's used to pick a sample name out of a `samples.txt` list, one per line.
- `%A_%a` in the output filename expands to the overall array job ID and the individual task index, so each sample gets its own log.

## 8. Chaining pipeline steps with dependencies

If step 2 needs step 1 to finish successfully first, tell Slurm about the dependency instead of babysitting the queue yourself:

```bash
jobid1=$(sbatch --parsable align.sh)
sbatch --dependency=afterok:$jobid1 variant_call.sh
```

`--parsable` makes `sbatch` print just the job ID, so you can capture it in a variable. `afterok` means "only start once the first job finished successfully" — if it fails, the dependent job is cancelled rather than running on incomplete input.

## 9. Checking resource usage after a job finishes

```bash
sacct -j 123456 --format=JobID,Elapsed,MaxRSS,State
```

shows how long the job took and how much memory it actually used. A quicker summary of the same idea:

```bash
seff 123456
```

```text
CPU Efficiency: 42.10% of 04:00:00 core-walltime
Memory Efficiency: 61.35% of 8.00 GB
```

Low CPU efficiency usually means you asked for more CPUs than the job actually used; low memory efficiency means you over-requested memory. Use this to size your *next* job's request more accurately — over-asking just makes it wait longer in the queue for resources it won't use.

## 10. When a job fails

**Check the log first**

```bash
tail -n 50 my-analysis-123456.log
```

**Common causes:**

- **Out of memory** — the log (or `sacct -j <jobid> --format=State`) shows `OUT_OF_MEMORY`, or the process is simply killed with no clear error. Re-run with a higher `--mem`, informed by what `seff` showed for a similar job.
- **Exceeded the time limit** — `State` shows `TIMEOUT`. Increase `--time`, or check whether the job is genuinely stuck rather than just slow.
- **Software not found** — the script ran, but a command failed immediately. Check that the right `module load`/`conda activate` is actually inside the script (Section 3), not just run in your interactive shell beforehand.

## 11. A few habits worth building

- **Ask for what you need, not more.** Over-requesting CPUs/memory just means your job waits longer in the queue for resources it won't use.
- **Give your job a sensible `--time` limit.** Too short and it gets killed before finishing; too long and it may queue longer than necessary.
- **Check `sacct`/`seff` after a job finishes**, not just whether it completed — it tells you whether your resource request actually matched what you used.
- **Test interactively before scaling up.** Get one sample working with `interactive` before submitting a 96-way array job.

## 12. Your Slurm survival kit

| Command | What it does |
| --- | --- |
| `sbatch script.sh` | Submit a job script to the queue |
| `squeue -u $USER` | See your jobs and their status (pending/running) |
| `scancel <jobid>` | Cancel a job |
| `sacct -j <jobid> --format=...` | See how a finished job used its resources |
| `seff <jobid>` | Quick CPU/memory efficiency summary |
| `interactive` | Start an interactive session on a compute node |
| `sinfo` | List partitions and node availability |
| `--array=1-N%C` | Submit N tasks, at most C running at once |
| `--dependency=afterok:<jobid>` | Only start once another job finishes successfully |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Write a minimal job script with sensible `--time`/`--mem`/`--cpus-per-task` values
- [ ] Submit it with `sbatch` and watch it with `squeue -u $USER`
- [ ] Tail the log file while the job runs
- [ ] Run `sacct` and `seff` once it finishes, and compare requested vs. actual resource use
- [ ] Start an `interactive` session (inside `tmux`) and run a command by hand before scripting it
- [ ] Turn a loop over 3–5 test files into a small `--array` job, using `$SLURM_ARRAY_TASK_ID` to pick each one
- [ ] Submit two scripts with a `--dependency=afterok:` link between them, and confirm the second only starts after the first succeeds
- [ ] Deliberately under-request memory for a small test job and confirm you can see the `OUT_OF_MEMORY` state in `sacct`
