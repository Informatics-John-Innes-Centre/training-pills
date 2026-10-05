# Singularity Basics

## Running tools from containers on the HPC

**Time:** about 30–40 minutes
**Level:** Beginner — you should be comfortable with the [Linux](linux-basics.md) and [Slurm](slurm-basics.md) basics

By the end of this tutorial, you will be able to:

- Explain what a container is and why it helps reproducibility
- Find a ready-made container image for a bioinformatics tool
- Download an image as a `.sif` file on `software23`, or build one from a definition file
- Run a tool from a container, interactively and in a Slurm job
- Make your data folders visible inside the container
- Use a GPU from inside a container
- Keep container downloads from filling your home folder

The main idea to take away:

> **A container image is a single file holding a tool *and* everything it needs. Download it once on `software23`, keep the `.sif` file, and every run, on any node and by anyone, uses exactly the same software.**

---

## 1. What a container is

A tool needs more than its own program to run: the right libraries, the right Python or Java version, sometimes other tools. A **container image** packs all of that into one file. Running a command "in" the container uses the software inside the image, not whatever is installed on the node.

```text
samtools-1.18.sif
├── samtools 1.18
├── htslib, zlib, libcurl ...   exact versions it was built with
└── a minimal Linux system

singularity exec samtools-1.18.sif samtools view ...
        └─ runs samtools from inside the image, on your files
```

Why this helps:

- **Reproducible:** the same `.sif` file gives the same results next year, on another cluster, or on a colleague's account.
- **Nothing to install:** no conda solving, no dependency conflicts.
- **Shareable:** you can put an image in a group folder and everyone uses the same one.

> **Do you need a container?** For a quick analysis, a module or Software Catalogue package is often simpler (see the [HPC Software](hpc-software-basics.md) pill). Containers shine when you need the exact same software everywhere, or a pipeline provides the images.

> **Singularity and Apptainer** are the same tool under two names (Apptainer is the newer, community-maintained branch). Commands are identical; replace `singularity` with `apptainer` where that's what is installed. Docker is the other well-known container tool, but it isn't available on the HPC. Singularity can run Docker images, though.

## 2. Where images come from

You rarely build images yourself. Most bioinformatics tools already have one:

- **BioContainers** (`quay.io/biocontainers/...`): an image for every Bioconda package. Find the exact tag at https://quay.io/organization/biocontainers or https://biocontainers.pro.
- **Seqera Containers** (https://seqera.io/containers): pick one or more conda packages and get a ready-made image.
- **Vendor images:** e.g. NVIDIA Parabricks at `nvcr.io/nvidia/clara/clara-parabricks`.
- **Docker Hub** (`docker.io/...`): general software.

**Always use a specific tag, never `latest`.** `samtools:1.18--h50ea8bc_1` always means the same thing; `latest` changes over time.

## 3. Downloading an image (on `software23`)

Only `software23` has internet access, so download there:

```bash
ssh software23
mkdir -p /jic/scratch/groups/<your-group>/sif
cd /jic/scratch/groups/<your-group>/sif

singularity pull samtools-1.18.sif docker://quay.io/biocontainers/samtools:1.18--h50ea8bc_1
```

`singularity pull` downloads the Docker image and converts it into a single `.sif` file. You don't need admin rights for this. Once you have the `.sif`, it works on every node without internet.

Check it:

```bash
singularity exec samtools-1.18.sif samtools --version
```

> **Keep images in a group folder**, not your home, and give them clear names with the version (`samtools-1.18.sif`, not `samtools.sif`). Images range from tens of MB to several GB.

## 4. Running a tool from a container

**One command**

```bash
singularity exec samtools-1.18.sif samtools flagstat sample.bam
```

**A shell inside the container**, handy for exploring:

```bash
singularity shell samtools-1.18.sif
Singularity> samtools --version
Singularity> exit
```

**Inside a Slurm job**

```bash
#!/bin/bash
#SBATCH --job-name=flagstat
#SBATCH --partition=jic-short
#SBATCH --cpus-per-task=4
#SBATCH --mem=4G
#SBATCH --time=00:30:00

SIF=/jic/scratch/groups/<your-group>/sif/samtools-1.18.sif

singularity exec "$SIF" samtools flagstat -@ 4 sample.bam > sample.flagstat.txt
```

No `ml` or `conda activate` needed: the software is in the image.

## 5. Seeing your files from inside the container

By default the container can see your **home folder** and the **current directory**. Data elsewhere, such as scratch or archive paths, has to be **bound** (mounted) in with `-B`:

```bash
singularity exec -B /jic/scratch -B /jic/archive samtools-1.18.sif \
    samtools view -c /jic/scratch/groups/<your-group>/data/sample.bam
```

If a tool says "No such file or directory" for a file you can see outside the container, a missing `-B` is the usual cause. `-B /a/path:/inside/path` mounts a folder at a different location inside the container.

> Workflow managers such as Nextflow add the right `-B` options for you (`autoMounts`).

## 6. GPUs

GPU tools need the GPU drivers from the host. Add `--nv`:

```bash
singularity exec --nv parabricks-4.3.2-1.sif nvidia-smi
```

This only works on a GPU node, i.e. in a job submitted to a GPU partition with a GPU requested (`--gres=gpu:...`).

## 7. Building your own image

When no ready-made image fits, write a small **definition file** (`.def`):

```text
Bootstrap: docker
From: quay.io/biocontainers/bcftools:1.21--h8b25389_0

%post
    # commands that run once, while building
    echo "built for my-project" > /opt/README

%test
    bcftools --version
```

Building from a `.def` file needs root rights, which are available on `software23` for this purpose. So build there, like pulls:

```bash
ssh software23
singularity build bcftools-1.21.sif bcftools-1.21.def
```

For a plain image with no extra steps, `singularity pull` (Section 3) is enough and works anywhere with internet.

Keep `.def` files in git rather than `.sif` files. They are tiny text files that record exactly how each image was made.

## 8. Keeping your home folder small

Singularity caches every layer it downloads in `~/.singularity/cache`, which quickly fills your home quota. Move the cache, and the temporary build space, to scratch (add these to `~/.bashrc` on `software23`):

```bash
export SINGULARITY_CACHEDIR=/jic/scratch/groups/<your-group>/<you>/.singularity_cache
export SINGULARITY_TMPDIR=/jic/scratch/groups/<your-group>/<you>/.singularity_tmp
mkdir -p "$SINGULARITY_CACHEDIR" "$SINGULARITY_TMPDIR"
```

Clear the cache when you no longer need it:

```bash
singularity cache clean
```

## 9. When it goes wrong

- **Pull fails with a network error or timeout:** you're not on `software23`.
- **`No such file or directory` for a file that exists:** the folder isn't bound into the container (Section 5).
- **`FATAL: ... permission denied` when building:** building from a `.def` needs root rights, so build on `software23` (Section 7).
- **A GPU tool finds no GPU:** you forgot `--nv`, or the job isn't on a GPU node.
- **`Disk quota exceeded` while pulling:** move the cache (Section 8).
- **`WARNING: File mode (700) on ~/.singularity/remote.yaml needs to be 600`:** harmless; Singularity fixes it itself. `chmod 600 ~/.singularity/remote.yaml` silences it.

## 10. Your Singularity survival kit

| Command | What it does |
| --- | --- |
| `ssh software23` | Go to the only node with internet, for downloads |
| `singularity pull <name>.sif docker://<image>:<tag>` | Download an image as a `.sif` file |
| `singularity exec <sif> <command>` | Run one command inside the container |
| `singularity shell <sif>` | Open a shell inside the container |
| `singularity exec -B /jic/scratch <sif> ...` | Make a folder visible inside the container |
| `singularity exec --nv <sif> ...` | Use the node's GPU from inside the container |
| `singularity build <name>.sif <name>.def` | Build an image from a definition file |
| `singularity cache clean` | Clear downloaded layers |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Set `SINGULARITY_CACHEDIR` and `SINGULARITY_TMPDIR` to scratch on `software23`
- [ ] Find the BioContainers tag for `seqkit` and pull it as `seqkit-<version>.sif` into a group folder
- [ ] On a login node, run `singularity exec seqkit-<version>.sif seqkit version`
- [ ] Run `seqkit stats` on a FASTQ file in scratch, first without and then with `-B /jic/scratch`, and see the difference
- [ ] Write a Slurm job that runs the same command from the `.sif`
- [ ] Open `singularity shell` on the image and look around (`which seqkit`, `cat /etc/os-release`)
- [ ] Compare `seqkit version` from the container with a conda-installed `seqkit` (see the [conda](conda-basics.md) pill)
