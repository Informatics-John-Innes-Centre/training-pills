# Conda Basics

## Installing software into your own environments on the HPC

**Time:** about 30–40 minutes
**Level:** Beginner — you should be comfortable with the [Linux](linux-basics.md) command line

By the end of this tutorial, you will be able to:

- Explain what conda is and why it uses separate environments
- Set up conda so that it only uses the free channels allowed at JIC
- Install conda yourself if you don't have it yet
- Create an environment, install tools into it and activate it
- Use an environment inside a Slurm job script
- Save an environment to a file so someone else can recreate it
- Keep your home folder from filling up with conda packages

The main idea to take away:

> **One environment per project or tool, created on `software23` from the free channels, and activated wherever you need it, including inside your job scripts.**

---

## 1. What conda is

Conda is a package manager: it downloads and installs software, plus everything that software depends on, into a folder you own. You don't need admin rights, and you can have different versions of the same tool side by side.

It does this with **environments**: separate folders, each with its own set of tools.

```text
~/miniforge3/envs/
├── nf-env/        nextflow 26.04 + java
├── qc-env/        fastqc, multiqc
└── py-analysis/   python 3.12, pandas, matplotlib
```

Activating an environment puts its tools first on your `PATH`; deactivating it removes them. Environments can't break each other, so a tool that needs an old Python can't break your other analyses.

> **Conda or `ml`?** If the tool is already available as a module or in the HPC Software Catalogue (see the [Software](software-basics.md) pill), loading it that way is simpler. Use conda when it isn't there, when you need a different version, or when you want a set of tools pinned together for a project.

## 2. Two JIC-specific rules

**Rule 1: only `software23` has internet access.** Installing anything downloads packages, so do `conda create` and `conda install` there:

```bash
ssh software23
```

Activating and using an environment works on every node, because the environment is just files in your folder.

**Rule 2: only the free channels.** A *channel* is where conda downloads packages from. Anaconda's own channels (`defaults`, `main`, `r`) now need a paid licence for institutions like JIC, so they're blocked on the HPC to keep the Institute from being liable for unlicensed use. Use the free community channels instead, through the prefix.dev mirror. Put this in `~/.condarc`:

```yaml
channels:
  - bioconda
  - conda-forge
  - pytorch
  - nvidia

channel_alias: https://repo.prefix.dev
```

- If `~/.condarc` doesn't exist, create it with exactly this (e.g. `nano ~/.condarc`).
- If it already exists, **remove any existing `channels:` and `channel_alias:` entries first**, so the old, now-blocked Anaconda channels aren't still configured.
- `channel_alias` routes these channels through prefix.dev's free mirror instead of Anaconda's servers.

Check what conda will use:

```bash
conda config --show channels
conda config --show channel_alias
```

You should see `bioconda`, `conda-forge`, `pytorch` and `nvidia`, but no `defaults`.

## 3. Getting conda (if you don't have it yet)

Type `conda --version`. If you get "command not found", install **Miniforge**, a free conda distribution that uses conda-forge by default (on `software23`):

```bash
ssh software23
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh -b -p ~/miniforge3
~/miniforge3/bin/conda init bash
exit
```

Log out and back in, and `conda --version` should work. Then set up `~/.condarc` as in Section 2.

Stop conda from activating its `base` environment in every new shell, which slows logins and hides problems:

```bash
conda config --set auto_activate_base false
```

> **Don't install tools into `base`.** Keep `base` for conda itself and create a separate environment for everything else. A broken `base` breaks conda.

## 4. Creating and using an environment

```bash
ssh software23
conda create -n qc-env fastqc multiqc
```

Conda shows what it will install and asks `Proceed ([y]/n)?`. Type `y`.

```bash
conda activate qc-env
fastqc --version
conda deactivate
```

**Pin versions for anything you'll publish:**

```bash
conda create -n nf-env nextflow=26.04.1
```

**Add a tool to an existing environment:**

```bash
conda activate qc-env
conda install seqkit
```

**Find a package and its versions:**

```bash
conda search -c bioconda samtools
```

Or search on https://bioconda.github.io or https://prefix.dev.

> **Mamba:** `mamba` is a faster drop-in replacement for `conda` (`mamba create ...`, `mamba install ...`). It comes with Miniforge. Recent conda versions use the same fast solver anyway, so either command works.

## 5. Conda inside a Slurm job

A job starts with a fresh shell that doesn't know about conda. Load it explicitly in the script:

```bash
#!/bin/bash
#SBATCH --job-name=qc
#SBATCH --partition=jic-short
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=01:00:00

source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate qc-env

fastqc -t 4 *.fastq.gz
```

If `conda info --base` itself isn't found in the job, use the full path instead: `source ~/miniforge3/etc/profile.d/conda.sh`.

## 6. Sharing and recreating an environment

Save the environment to a file:

```bash
conda env export -n qc-env --no-builds > environment.yml
```

Keep `environment.yml` in your project's git repository. Anyone (including future you) can recreate the same environment on `software23`:

```bash
conda env create -f environment.yml
```

A hand-written file with just the tools you asked for is often easier to read and more portable across systems:

```yaml
name: qc-env
channels:
  - bioconda
  - conda-forge
dependencies:
  - fastqc=0.12.1
  - multiqc=1.25
```

## 7. Keeping your home folder small

Conda keeps a cache of every package it downloads, and environments can be several GB each. Your home folder has a quota.

**Clear the download cache** now and then:

```bash
conda clean --all
```

**Delete environments you no longer use:**

```bash
conda env list
conda env remove -n old-env
```

**Store environments and the cache outside home.** Add this to `~/.condarc` (pick a folder with space, e.g. your group's software area):

```yaml
envs_dirs:
  - /path/with/space/conda/envs
pkgs_dirs:
  - /path/with/space/conda/pkgs
```

## 8. When it goes wrong

- **`CondaHTTPError` / connection timeout:** you're not on `software23`.
- **Errors mentioning `repo.anaconda.com` or `defaults`:** an old channel is still configured. Check `conda config --show channels` and fix `~/.condarc` (Section 2).
- **`conda: command not found` in a job:** the script doesn't load conda (Section 5).
- **The solver runs forever or reports conflicts:** too many tools in one environment. Split them into smaller environments, or pin fewer versions.
- **`Disk quota exceeded`:** run `conda clean --all` and see Section 7.

## 9. Your conda survival kit

| Command | What it does |
| --- | --- |
| `ssh software23` | Go to the only node with internet, for installs |
| `conda config --show channels` | Check which channels conda uses |
| `conda create -n <env> <tool>=<version>` | Create an environment |
| `conda activate <env>` / `conda deactivate` | Switch an environment on/off |
| `conda install <tool>` | Add a tool to the active environment |
| `conda env list` | List your environments |
| `conda env export --no-builds > environment.yml` | Save an environment to a file |
| `conda env create -f environment.yml` | Recreate an environment from a file |
| `conda env remove -n <env>` | Delete an environment |
| `conda clean --all` | Free space used by the package cache |
| `source "$(conda info --base)/etc/profile.d/conda.sh"` | Make conda available in a job script |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Set up `~/.condarc` with the free channels and confirm `defaults` is not listed by `conda config --show channels`
- [ ] On `software23`, create an environment with a pinned version of `seqkit`
- [ ] Activate it on a login node (not `software23`) and run `seqkit version`
- [ ] Export it to `environment.yml`, delete it, and recreate it from the file
- [ ] Write a small Slurm job that activates the environment and runs `seqkit stats` on a FASTQ file
- [ ] Run `conda clean --all` and note how much space it frees
