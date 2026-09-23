# tmux Basics for Bioinformatics

## Keeping your interactive work alive on an HPC

**Time:** about 20–30 minutes
**Level:** Beginner — no prior tmux experience needed

By the end of this tutorial, you will be able to:

- Explain what tmux protects on a Slurm HPC — and what Slurm already protects on its own
- Start a tmux session
- Detach from a session without stopping what's running inside it
- Reattach to a session from anywhere
- List and manage multiple sessions

The main idea to take away:

> **tmux keeps a terminal session alive on the login node, even after you disconnect.**

`sbatch` jobs don't need this — Slurm already runs them on a compute node independently of your terminal. tmux matters for everything that *isn't* handed off to the scheduler.

---

## 1. Why tmux on a Slurm HPC?

You've probably had this happen:

```text
$ ssh hpc.nbi.ac.uk
$ interactive
$ python debug_pipeline.py
...
[connection lost — laptop went to sleep / VPN dropped / wifi died]
```

That interactive session dies the instant your SSH connection drops — because it was running directly in your terminal, not managed by Slurm.

Compare that with a batch job:

```bash
sbatch align_job.sh
```

This one is safe with or without tmux: Slurm hands it to a compute node and it keeps running whether you're connected or not.

So on an HPC, tmux is for the things Slurm *doesn't* look after:

- **Interactive sessions** — our `interactive` wrapper (or `salloc`), an interactive Python/R session on a compute node
- **Monitoring** — watching `squeue`, tailing a log file while you keep working elsewhere
- **File transfers** — moving data to/from the login node (see Section 7)
- **Prototyping** — testing a command interactively before wrapping it in an `sbatch` script

```text
Your laptop          Login node              Compute node
     │                    │                        │
     │   ssh + tmux       │                        │
     ├───────────────────►│                        │
     │                    │   interactive           │
     │                    ├───────────────────────►│
     │  connection drops  │  ── session kept alive ──►
     │                    │        by tmux           │
     │   ssh + reattach   │                        │
     ├───────────────────►│                        │
     │   (session still there)                     │
```

## 2. Start a session

Log into the login node as normal, then start a named tmux session:

```bash
tmux new -s monitor
```

Your terminal now looks almost the same, but you're inside a tmux session. You'll usually see a green status bar at the bottom showing the session name.

> **Tip:** Always name your sessions (`-s <name>`). It's much easier to find `monitor` again later than to guess which unnamed session is which.

## 3. Interactive sessions and monitoring

**An interactive session on a compute node**

```bash
interactive
```

Now you're on a compute node, working interactively — testing a script, debugging, running an interactive Python/R session. This is tied to your terminal, so it's exactly the kind of thing to protect with tmux.

**Monitoring a job you've already submitted with `sbatch`**

```bash
squeue -u $USER
tail -f slurm-12345.out
```

Leave this running in a tmux session so you can check progress from anywhere, without re-typing the command every time you log back in.

## 4. Detach — leave the job running

To step away without losing what's running inside it, **detach** from the session:

```text
Ctrl-b, then d
```

(Press `Ctrl` and `b` together, release, then press `d`.)

You'll land back in your normal terminal, and you'll see something like:

```text
[detached (from session monitor)]
```

Whatever is inside the session — your interactive `srun` job, your `tail -f`, your `squeue` — keeps running on the login/compute node. You can now close your laptop, lose wifi, or log out completely — it doesn't matter.

## 5. Reattach — check back in

Log back in (from anywhere — office, home, a different laptop) and reattach:

```bash
tmux attach -t monitor
```

You're back exactly where you left off — check the log output, see if your interactive session is still alive, or keep working.

## 6. Working with multiple sessions

You can have several sessions running at once — for example, one for monitoring jobs and one for an interactive debugging session.

**List all sessions**

```bash
tmux ls
```

```text
monitor: 1 windows (created ...) 
debug: 1 windows (created ...)
```

**Attach to a specific one**

```bash
tmux attach -t debug
```

**Close a session once you're done with it**

```bash
tmux kill-session -t monitor
```

> **Important:** A session only disappears when you kill it or the server reboots — simply detaching does not stop it. If you don't need the session anymore, kill it to keep things tidy (and free up the interactive allocation if it was an `srun` session).

## 7. Moving large files

Copying sequencing data or results between servers runs on the login node, not through Slurm — so it's just as vulnerable to a dropped connection as any other interactive command. A multi-GB transfer can easily take hours. Run it in tmux the same way you'd run any other interactive task.

**Start a session for the transfer**

```bash
tmux new -s transfer
```

**Copy the data with `rsync`**

```bash
rsync -avP /data/raw_reads/ user@hpc.nbi.ac.uk:/data/project/raw_reads/
```

`-P` shows a progress bar and, importantly, allows the transfer to **resume** if it gets interrupted, instead of starting again from scratch.

**Detach, and check on it later**

```text
Ctrl-b, then d
```

```bash
tmux attach -t transfer
```

> **Tip:** Prefer `rsync -avP` over `scp` for large or unreliable transfers — if the connection drops, re-running the same `rsync` command only copies what's missing, rather than the whole file again.

> **Note:** Large transfers are often run from a dedicated data mover node rather than the login node. Some data movers don't have `tmux` installed — if so, use `screen` instead (`screen -S <name>` to start, `Ctrl-a d` to detach, `screen -r <name>` to reattach). See the `rsync-basics` pill for a full rsync walkthrough, including this mapping and other gotchas (trailing slashes, compression, symlinks vs hard links).

## 8. A typical bioinformatics workflow

```text
ssh hpc.nbi.ac.uk
sbatch assembly.sh              (job runs on a compute node, managed by Slurm —
  ↓                              this part survives disconnects on its own)
tmux new -s monitor
  ↓
squeue -u $USER
tail -f slurm-<jobid>.out
  ↓
Ctrl-b d                 (detach, log off, go home)
  ↓
... hours later ...
  ↓
ssh hpc.nbi.ac.uk
tmux attach -t monitor    (check progress / see if it finished)
  ↓
tmux kill-session -t monitor   (once finished)
```

## 9. Your tmux survival kit

These are the commands to remember after this tutorial:

| Command | What it does |
| --- | --- |
| `tmux new -s <name>` | Start a new named session |
| `Ctrl-b d` | Detach from the current session |
| `tmux ls` | List all sessions |
| `tmux attach -t <name>` | Reattach to a session |
| `tmux kill-session -t <name>` | Close a session |
| `rsync -avP <src> <dest>` | Copy files, resumable if interrupted |

**The workflow to remember:**

```text
       ┌──────────────┐
       │ ssh to server│
       └──────┬───────┘
              ↓
      tmux new -s <name>
              ↓
      run your job
              ↓
        Ctrl-b d   (safe to disconnect)
              ↓
   ... later, from anywhere ...
              ↓
   tmux attach -t <name>
```

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] SSH into the login node
- [ ] Start a new named tmux session
- [ ] Start an interactive session inside it with `interactive`
- [ ] Detach with `Ctrl-b d`
- [ ] Confirm you're back in your normal terminal on the login node
- [ ] List sessions with `tmux ls`
- [ ] Log out completely, then log back in
- [ ] Reattach to your session with `tmux attach -t <name>` and confirm your interactive session is still alive
- [ ] Submit a real job with `sbatch` (outside of tmux) and confirm it keeps running after you detach/disconnect
- [ ] Start a separate tmux session and use it to `tail -f` that job's log or watch `squeue -u $USER`
- [ ] Kill each session once you're done with it
- [ ] Start a new session and run an `rsync -avP` transfer inside it
- [ ] Detach, then reattach and confirm the transfer is progressing or complete
