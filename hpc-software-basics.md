# HPC Software Basics

## Finding and loading software on the HPC

**Time:** about 20–30 minutes
**Level:** Beginner — you should be comfortable with the [Linux](linux-basics.md) command line

By the end of this tutorial, you will be able to:

- Explain why most tools aren't available until you load them
- Find and load software with Lmod modules (`ml`)
- Find and load software from the HPC Software Catalogue
- Load software inside a Slurm job script
- Choose between modules, the catalogue, conda and containers
- Ask for software that isn't available yet

The main idea to take away:

> **Typing a tool's name only works after you load it. Search for it (`ml av`, `catalogue --search`), load a specific version, and put that same load command in every job script that uses it.**

---

## 1. Why tools are "not found"

The cluster serves hundreds of users who need different tools, and often different versions of the same tool. Installing everything into one shared place would make versions clash. So most software is installed side by side and kept *out* of your environment until you ask for it:

```bash
bcftools --version
```

```text
bash: bcftools: command not found
```

That doesn't mean bcftools isn't on the cluster; it just isn't **loaded**. Loading a tool adds it to your `PATH` for the current shell (or job) only.

There are four ways to get a tool. Try them in this order:

| Way | Use it when |
| --- | --- |
| **Lmod module** (`ml`) | The tool is installed as a module. Quickest. |
| **Software Catalogue** (`source package`) | The tool is in the HPC Software Catalogue. |
| **Conda** | Neither has it, or you need a version they don't have. See the [conda](conda-basics.md) pill. |
| **Container** (`singularity exec`) | You want exactly the same software everywhere, or a pipeline already provides an image. See the [Singularity](singularity-basics.md) pill. |

## 2. Lmod modules

**Lmod** is the module system. `ml` is short for `module load`, and works for most module commands.

**Search**

```bash
ml av                 # every available module (long!)
ml av bcftools        # only modules whose name contains "bcftools"
ml spider bcftools    # description and all versions
ml keyword paftools   # find the module that provides a command
```

**Load and check**

```bash
ml bcftools/1.21      # load a specific version
bcftools --version
ml                    # list what's loaded now
```

Tab completion works for module names and versions: type `ml bcft` and press Tab.

**Unload**

```bash
ml unload bcftools    # one module
ml purge              # everything
```

> **Always give the version** (`ml bcftools/1.21`, not just `ml bcftools`). Without it you get the *default* version, which can change when a newer one is installed, and then your results can change without you noticing.

**Save a set you use often**

```bash
ml samtools/1.18 bcftools/1.21
ml save variant-tools     # save the current set under a name
ml restore variant-tools  # reload it later, e.g. in a new session
```

## 3. The HPC Software Catalogue

The catalogue holds more packages, and you can also browse it on the web at https://software.hpc.nbi.ac.uk.

```bash
catalogue --search bcftools     # search; note the package ID it gives
catalogue -i <ID>               # details for one package
source package <ID>             # load it
bcftools --version
```

Older software lives in the legacy areas:

```bash
catalogue --legacy              # search the legacy production/testing areas
source package <name>           # legacy packages are loaded by name
```

On the website you can also **vote** for packages you'd like added or updated.

## 4. Loading software in a job script

A Slurm job starts with a clean environment: modules you loaded in your terminal are **not** carried over. Put the load commands inside the script:

```bash
#!/bin/bash
#SBATCH -p jic-short
#SBATCH -c 4
#SBATCH --mem=8G
#SBATCH -t 0-01:00

ml samtools/1.18
source package <ID>      # e.g. a catalogue package

samtools flagstat -@ 4 sample.bam > sample.flagstat.txt
```

> **Why this matters:** a command that works in your terminal can fail with `command not found` in a job if the script itself doesn't load the software. It's the most common reason a job fails within seconds. The [Slurm](slurm-basics.md) pill covers job scripts in more detail.

## 5. When the tool isn't there

1. Check both places: `ml spider <tool>` **and** `catalogue --search <tool>`. Names can differ, e.g. `blast+` or `ncbi-blast`.
2. Search by command instead of package name: `ml keyword <command>`.
3. Install it yourself with conda ([conda](conda-basics.md) pill), or use a ready-made container image ([Singularity](singularity-basics.md) pill).
4. If many people would use it, request or vote for it on https://software.hpc.nbi.ac.uk.

## 6. A few habits worth building

- **Pin versions** in every `ml` command you put in a script.
- **Record versions in your methods.** `ml` shows what's loaded; most tools also report it with `--version`.
- **Load only what you need** in each job. Loaded modules can override each other's libraries.
- **Start job scripts with `ml purge`** if you see odd library errors. It makes the job independent of whatever your shell had loaded.

## 7. Your software survival kit

| Command | What it does |
| --- | --- |
| `ml av <name>` | Search modules by name |
| `ml spider <name>` | Describe a module and list its versions |
| `ml keyword <command>` | Find the module that provides a command |
| `ml <name>/<version>` | Load a specific version |
| `ml` | List loaded modules |
| `ml unload <name>` / `ml purge` | Unload one / all modules |
| `ml save <set>` / `ml restore <set>` | Save / reload a set of modules |
| `catalogue --search <term>` | Search the Software Catalogue |
| `catalogue -i <ID>` | Details for a catalogue package |
| `source package <ID>` | Load a catalogue package |
| `catalogue --legacy` | Search the legacy software areas |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Run `bcftools --version` and see that it's not found
- [ ] Find bcftools with `ml av bcftools` and `ml spider bcftools`, and load a specific version
- [ ] Run `ml` to confirm it's loaded, then `ml purge` and confirm it's gone
- [ ] Find samtools with `catalogue --search samtools` and load it with `source package <ID>`
- [ ] Use `ml keyword` to find which module provides a command you use
- [ ] Write a Slurm job that loads a pinned module version and prints its `--version` into the log
- [ ] Save two modules as a collection with `ml save`, then `ml purge` and `ml restore` them
