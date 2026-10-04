# GPU Variant Calling with Parabricks and Nextflow

## From FASTQ files to a cohort VCF with parabricks-germline-nf

**Time:** about 45–60 minutes (plus queue time for the GPU test)
**Level:** Intermediate — assumes you have done the [Slurm](slurm-basics.md) and [git](git-basics.md) pills and have used conda

By the end of this tutorial, you will be able to:

- Describe the biology behind it: reads, alignment, variants and genotypes
- Explain what the pipeline does, from FASTQ files to a multi-sample VCF
- Explain the few Nextflow ideas you need: processes, the work directory, profiles and `-resume`
- Set up Nextflow in a conda environment and clone the pipeline
- Get the container images, either pre-built or built from the recipes in the repo
- Write a samplesheet for your own samples
- Check your setup with a dry run, then with a small real GPU test
- Launch a full run on the cluster, follow its progress and resume it after a failure
- Find your results and know what the merged VCF does and does not contain

The main idea to take away:

> **You describe *what* to run (samplesheet, reference, profile) and Nextflow works out *how*: it submits every step as a Slurm job, runs it in a fixed container, and remembers what already finished so a failed run picks up where it stopped.**

---

## 1. The biology in brief

Every individual (a plant, an animal, a person) carries small differences in its DNA compared with a **reference genome**, the agreed "standard" sequence for that species. Most differences are **SNPs** (single-letter changes, e.g. `A` → `G`) or short **indels** (a few letters inserted or deleted). **Germline** variants are the ones the individual inherited and carries in every cell, as opposed to mutations that arise in one tissue, such as in a tumour.

Finding these differences across many individuals is the starting point for most population and breeding genetics: measuring diversity, building trees of relatedness, linking variants to traits (GWAS, QTL mapping) and designing markers.

The pipeline does this in three biological steps:

1. **Alignment: where does each read come from?** The sequencer produces millions of short reads (typically 2 × 150 bp) from random positions in the genome. Each read is placed at its best-matching position in the reference. Reads that are just PCR copies of the same DNA fragment are flagged as **duplicates**, so they aren't counted twice as evidence.
2. **Variant calling: where does this sample differ?** At each position, the caller (GATK HaplotypeCaller) looks at all the reads stacked on top of it. If they consistently show a different letter from the reference, that's a variant. It then decides the **genotype**:

   ```text
   Reference   ...ACGTTAGC...
   Reads       ...ACGTTAGC...   0/0  same as reference on both copies (homozygous reference)
               ...ACGCTAGC...
               ...ACGTTAGC...   0/1  one copy of each (heterozygous)
               ...ACGCTAGC...
               ...ACGCTAGC...   1/1  different on both copies (homozygous alternative)
   ```

   Inbred lines, such as most wheat landraces, are mostly `0/0` or `1/1`.
3. **Merging: one table for the whole cohort.** The per-sample results are combined into one **VCF** (Variant Call Format) file. It is essentially a table with one row per variant position and one column per sample, holding that sample's genotype. This matrix is what downstream analyses use.

**More reads per position (coverage) means more confident genotypes.** At 2× coverage a heterozygous site can easily look homozygous by chance; at 10–30× it rarely does.

> **Wheat note:** bread wheat is hexaploid, with three closely related sub-genomes (A, B and D). A read from one sub-genome can align to the matching gene on another, which can create false variants. Filtering the final VCF on quality, depth and missing data is therefore essential, and is not done by this pipeline.

## 2. What the pipeline does

`parabricks-germline-nf` takes paired-end FASTQ files for many samples and produces per-sample and cohort VCFs. The slow part, alignment and variant calling, runs on GPUs with NVIDIA Parabricks, which is many times faster than CPU-only BWA + GATK.

```text
FASTQ (1+ lanes per sample)
  └─ pbrun germline (BWA-MEM + sort + markdup + HaplotypeCaller, GPU)  -> cram/<sample>.cram
       └─ bgzip + index                                                -> vcf/<sample>.vcf.gz
            └─ bcftools merge, per contig, in sample chunks            -> merged/per_contig/<prefix>.<contig>.vcf.gz
                 └─ bcftools concat (reference order)                  -> merged/<prefix>.vcf.gz
```

Repository: `https://git.nbi.ac.uk/workflows/parabricks-germline-nf`

## 3. Just enough Nextflow

- **Process** — one step of the pipeline (e.g. `PBRUN_GERMLINE`). Each sample, or each contig, becomes its own **task**, and on the cluster each task is its own Slurm job.
- **Work directory** — every task runs in its own folder under `work/` (e.g. `work/8d/1e7b43...`). It holds the task's script (`.command.sh`), its log (`.command.log`) and its errors (`.command.err`). This is where you look when something fails.
- **Container** — every process runs inside a fixed image (Parabricks 4.3.2, bcftools 1.21, ...), so everyone gets the same tool versions.
- **Profile** — a named bundle of settings chosen with `-profile`. `jic` sets the right partitions, GPUs, local SSD and containers for our cluster; `test` uses a tiny bundled dataset.
- **`-resume`** — reuse every task that already finished successfully. You almost always want it.

> **Note:** Nextflow options have **one dash** (`-profile`, `-resume`); pipeline options have **two** (`--samplesheet`, `--ref`).

## 4. One-time setup

**Nextflow in a conda environment**

```bash
conda create -n nf-env -c conda-forge -c bioconda nextflow
conda activate nf-env
nextflow -version
```

**Clone the pipeline** (into scratch, where there is space for the work directory)

```bash
cd /jic/scratch/groups/<your-group>/<you>
git clone https://git.nbi.ac.uk/workflows/parabricks-germline-nf.git
cd parabricks-germline-nf
```

**Work offline**

Compute nodes have no internet access. Without the line below, Nextflow can sit for minutes trying to download things before giving up:

```bash
export NXF_OFFLINE=true
```

The launcher script (Section 8) sets this for you; you only need it when running `nextflow` by hand.

## 5. Containers

The images are too big for git (Parabricks alone is several GB), so the repo holds small **recipe files** (`containers/*.def`) instead.

- **Using the shared copy:** the `jic` profile already points at pre-built `.sif` images, so there's nothing to do.
- **Building your own** (on a node with internet access):

```bash
containers/build.sh /jic/scratch/groups/<your-group>/sif
```

This builds the four images, tests each one and writes a `containers.yml`. Then add `--container_dir /jic/scratch/groups/<your-group>/sif` to your runs.

## 6. Your samplesheet

A CSV with **one row per FASTQ pair**. Rows that share a `sample_id`, such as several lanes of one sample, are combined into one Parabricks run.

```csv
sample_id,read1,read2,gpus
S1,/path/S1_L001_R1.fastq.gz,/path/S1_L001_R2.fastq.gz,1
S1,/path/S1_L002_R1.fastq.gz,/path/S1_L002_R2.fastq.gz,1
S2,/path/S2_R1.fastq.gz,/path/S2_R2.fastq.gz,2
```

- `sample_id` becomes the sample name in the VCF. Use only letters, digits, `.`, `_` and `-`.
- `gpus` is optional (default 1). Give high-coverage samples 2 or 4.
- Paths can be absolute, or relative to the samplesheet's folder.
- Every FASTQ file is checked **before** anything runs, so a typo fails in seconds rather than hours in.

## 7. Test before you scale up

**Step 1: dry run (seconds, no GPU).** `-stub` replaces every tool with a command that just creates empty files, so this checks the wiring and the config:

```bash
interactive
conda activate nf-env
export NXF_OFFLINE=true
nextflow run . -profile test,stub -stub
```

Every process should finish with a ✔.

**Step 2: real GPU test on tiny data (a few minutes plus queue time)**

```bash
NF_CONDA_ENV=nf-env sbatch scripts/submit_slurm.sh -profile jic,test \
    --jic_gpu_scratch_gb 50 --min_scratch_gb 0 \
    --max_cpus 8 --max_memory '64 GB' --gpu_cpus 8 --gpu_memory '64 GB'
```

When it finishes, check the merged VCF:

```bash
ml bcftools
bcftools query -l results_test/merged/test.vcf.gz          # S1, S2, S3
bcftools view -H results_test/merged/test.vcf.gz | wc -l   # about 34 variant sites
```

> **Why the extra options?** The test data is tiny, so these shrink the requests. Without them the job would ask for 3.5 TB of local SSD and wait in the queue for an empty node.

## 8. A real run

```bash
NF_CONDA_ENV=nf-env sbatch scripts/submit_slurm.sh -profile jic \
    --samplesheet /path/to/samplesheet.csv \
    --ref /path/to/genome.fa \
    --prefix my_cohort \
    --outdir results
```

`scripts/submit_slurm.sh` runs Nextflow itself as a small, long-running Slurm job, the **head job**. It activates your conda environment, sets the offline variables and always adds `-resume`. The head job then submits every real task as its own Slurm job.

Useful options:

| Option | Use it when |
| --- | --- |
| `--contigs 1A,1B,...` | You only want some contigs in the merged VCF (e.g. skip unplaced scaffolds) |
| `--min_contig_length 1000000` | Your genome has thousands of tiny scaffolds |
| `--skip_concat` | You only need per-chromosome VCFs |
| `--run_flagstat` | You want mapping statistics for every CRAM |
| `--pbrun_args '--low-memory'` | Your GPUs have less than 40 GB of memory (not needed on our A100s) |

> **Reference index:** if the BWA index (`.amb .ann .bwt .pac .sa`) and `.fai` sit next to the FASTA, they are used as-is. Otherwise they are built and published to `results/reference/`. For a large genome that takes hours, so copy those files next to the FASTA for next time.

## 9. Watching a run

```bash
squeue -u $USER                     # the head job + the task jobs it submitted
tail -f nextflow_<jobid>.out        # the progress table
```

```text
[8d/1e7b43] BWA_INDEX (reference.fa)       | 1 of 1 ✔
[d4/376aa8] PBRUN_GERMLINE (S3)            | 3 of 3 ✔
[be/370374] BCF…ERGE_CHUNK (chrB:chunk2/2) | 4 of 4 ✔
```

The code in brackets (`d4/376aa8`) is the start of that task's work directory: `work/d4/376aa8...`.

After the run, `results/pipeline_info/` has an HTML **report** (CPU, memory and time used by every task) and a **timeline**. Use them to size the next run, just as you would use `seff` for a single Slurm job.

## 10. When something fails

Nextflow prints the failed task's command, its exit status and its work directory. Look there first:

```bash
cd work/d4/376aa8*      # use the code from the error message
cat .command.err        # what the tool complained about
cat .command.sh         # exactly what was run
```

Fix the cause, then resubmit **the same command**. Thanks to `-resume`, only the failed task and those after it run again; hours of finished GPU work are kept.

**Common causes:**

- **`Process requirement exceeds available memory`** when running by hand in a small `interactive` session: the local executor won't start a task that asks for more than the session has. Use the `stub` profile for dry runs, and `sbatch` with `-profile jic` for real ones.
- **`OUT_OF_MEMORY` or `TIMEOUT`** on the cluster: tasks killed by Slurm are retried automatically, up to 2 times. If one still fails, raise the matching option (`--gpu_memory`, `--merge_memory`, `--gpu_time`, ...) and resume.
- **Nextflow hangs at start-up:** `NXF_OFFLINE` isn't set (Section 4).
- **A FASTQ path wasn't found:** a typo in the samplesheet. Fix it and run again.

> **Don't delete `work/` while a project is in progress.** That's where `-resume` finds finished tasks. Once the results are safely in `results/` and you're done, delete `work/` to free scratch space; it can be huge.

## 11. What you get, and one important caveat

```text
results/
├── cram/                   <sample>.cram (+ .crai), needs the reference FASTA to read
├── vcf/                    <sample>.vcf.gz (+ .csi)
├── merged/per_contig/      <prefix>.<contig>.vcf.gz
├── merged/<prefix>.vcf.gz  all contigs, in reference order
└── pipeline_info/          report, timeline, trace
```

Sample columns in every merged VCF are sorted by `sample_id`.

> **The merged VCF is not joint-genotyped.** Each sample is called on its own, then the VCFs are combined. If a site is variant in some samples, every sample with no call there shows `./.` (missing), **even if it was well covered and simply matches the reference**. Keep this in mind for missing-data filters and allele frequencies.

## 12. A few habits worth building

- **Always run the dry run and the tiny GPU test first** on a new setup, before launching hundreds of samples.
- **Always use `-resume`.** The launcher adds it for you.
- **Keep the samplesheet and the exact command with your results**, so the analysis can be repeated.
- **Note the pipeline version** (`git log -1` in the repo folder) in your methods.
- **Clean up `work/`** once a project is finished.

## 13. Your Parabricks pipeline survival kit

| Command | What it does |
| --- | --- |
| `conda activate nf-env` | Make `nextflow` available |
| `export NXF_OFFLINE=true` | Stop Nextflow trying to reach the internet |
| `nextflow run . -profile test,stub -stub` | Dry run: check the setup in seconds |
| `NF_CONDA_ENV=nf-env sbatch scripts/submit_slurm.sh -profile jic ...` | Launch a run on the cluster |
| `tail -f nextflow_<jobid>.out` | Follow progress |
| `cat work/<xx>/<hash>*/.command.err` | See why a task failed |
| `containers/build.sh <dir>` | Build the container images yourself |
| `bcftools query -l <vcf>` | List the samples in a VCF |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Create the `nf-env` conda environment and check `nextflow -version`
- [ ] Clone the repository into your scratch space
- [ ] Run the dry run (`-profile test,stub -stub`) in an `interactive` session and get all ✔
- [ ] Submit the tiny GPU test with `scripts/submit_slurm.sh` and follow it with `tail -f`
- [ ] List the samples and count the variants in `results_test/merged/test.vcf.gz`
- [ ] Open `results_test/pipeline_info/report_*.html` and find which task used the most memory
- [ ] Find the work directory of one `PBRUN_GERMLINE` task and read its `.command.sh`
- [ ] Write a samplesheet for 2–3 of your own samples, one of them with two lanes
- [ ] Cancel a run halfway (`scancel` the head job), resubmit the same command, and confirm finished tasks show as `cached`
