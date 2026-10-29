# Git and DVC for Genome Assembly Workflows

## Keeping track of which assembly actually produced your results

**Time:** about 20–25 minutes
**Level:** Intermediate — assumes you're comfortable with [Git Basics](git-basics.md) and have run at least one assembly

By the end of this tutorial, you will be able to:

- Explain why Git alone isn't enough for an assembly project
- Decide what goes in Git and what goes in DVC (Data Version Control)
- Set up a DVC remote and track a large file or directory with it
- Commit QC metrics so two assembly versions can be compared with `git diff`
- Tag a release so a collaborator can reproduce it byte-for-byte

The main idea to take away:

> **Git tracks your pipeline and your QC numbers; DVC tracks the multi-gigabyte files those tools produce. Together, a Git tag plus a `dvc pull` is a complete, reproducible snapshot of an assembly — a directory called `assembly_final_v2_ACTUAL` is not.**

---

## 1. The problem DVC solves

A real assembly project runs through QC, assembly, polishing, scaffolding, and contamination screening, each step with parameters that change the output. Without a system for it, you end up with `assembly_final`, `assembly_final_v2`, and `assembly_final_v2_ACTUAL`, and nobody can say which one made the figure in the manuscript.

Git is built for diffable text and is poor at large binaries — every version of a FASTA is stored in full, and a 3 GB file committed to Git makes cloning painfully slow within a few commits. DVC fixes this by replacing a large file with a small `.dvc` pointer file (a hash and a size) that Git tracks instead. The actual data lives in a remote cache (S3, GCS, Azure, or a plain SSH server), and anyone who checks out a commit runs `dvc pull` to fetch the exact matching bytes.

## 2. What goes where

| Goes in... | Examples |
| --- | --- |
| **Git** | Pipeline scripts, `config/params.yaml`, QC reports (BUSCO/QUAST text output), `.dvc` pointer files |
| **DVC** | Raw reads (FASTQ), intermediate and final assemblies (FASTA), BAM/CRAM files, reference databases |

Rule of thumb: if it's text and under about 1 MB, Git. If it's binary or larger, DVC.

## 3. Setting it up

```bash
mkdir mygenome && cd mygenome
git init
dvc init
git add .dvc .dvcignore
git commit -m "Initialize DVC"
```

Point DVC at a remote cache — this app's own infra uses SSH, but the pattern is identical for S3/GCS/Azure:

```bash
dvc remote add -d storage ssh://user@server/path/to/cache
git add .dvc/config
git commit -m "Configure DVC remote"
```

Track your raw reads, then push the data (not just the pointer) to the remote:

```bash
dvc add data/raw/sample_R1.fastq.gz data/raw/sample_R2.fastq.gz
git add data/raw/*.dvc data/raw/.gitignore
git commit -m "Track raw reads for sample"
dvc push
```

> **Don't skip `dvc push`.** A `.dvc` pointer committed to Git with no matching data in the remote is a broken promise — a collaborator's `dvc pull` will succeed and quietly leave them with nothing.

## 4. A `.gitignore` that doesn't eat your pointers

The single most common DVC mistake: ignoring large files so aggressively that the `.dvc` pointer files get ignored too, silently losing your data history even though DVC thinks everything is tracked.

```gitignore
# Ignore large data by default
data/raw/*
assemblies/*
*.fasta
*.bam

# But never ignore DVC pointer files
!*.dvc
!.dvc/
```

## 5. Tracking an assembly and its QC together

```bash
dvc add assemblies/flye_run1
git add assemblies/flye_run1.dvc assemblies/.gitignore
git commit -m "Assembly run 1 output"
dvc push
```

QC output (BUSCO, QUAST) is small text, so it goes straight into Git alongside the assembly it describes — commit them together, not in a separate repo, or the two drift apart:

```bash
quast.py assemblies/flye_run1/assembly.fasta -o qc/quast_run1
busco -i assemblies/flye_run1/assembly.fasta -l eukaryota_odb10 -o busco_run1 -m genome
git add qc/quast_run1/report.tsv qc/busco_run1/short_summary.txt
git commit -m "QC metrics for assembly run 1"
```

Now `git diff` between two QC commits shows exactly how N50 and BUSCO completeness changed between runs — the actual payoff of this whole setup.

## 6. Tagging a release

A tag is an immutable label a collaborator (or future you) can return to. Put the headline quality numbers in the tag message itself:

```bash
git tag -a v1.0-assembly -m "First release assembly, BUSCO 96.2%, N50 12.4 Mb"
git push origin v1.0-assembly
dvc push
```

## 7. Reproducing from scratch

This is the real test of the whole setup:

```bash
git clone <repo-url>
cd mygenome
git checkout v1.0-assembly
dvc pull
```

If the pulled assembly's hash matches the one recorded in the `.dvc` file, you've reproduced the exact data used for that release. If it doesn't, your remote or pointer is broken — better to find that out now than when a reviewer asks.

## 8. Common mistakes

- **Committing a large FASTA to Git directly.** Once it's in history, getting it back out means rewriting history with `git filter-repo`, which breaks every existing clone. Prevent this with a pre-commit hook that rejects files over a size threshold.
- **Forgetting `dvc push` before a release.** `git status` being clean says nothing about whether DVC's data has actually reached the remote — check `dvc status -c` separately.
- **Not recording the assembler version.** Assembly output can change between releases of the same tool; pin and record the version alongside the parameters.
- **Treating N50 as the whole story.** It can be gamed by concatenating contigs — version completeness (BUSCO), correctness (QUAST misassemblies), and contamination screening together.

## 9. Your Git + DVC survival kit

| Command | What it does |
| --- | --- |
| `dvc init` | Set up DVC in a Git repo |
| `dvc remote add -d <name> <url>` | Set the default remote cache |
| `dvc add <file-or-dir>` | Start tracking something with DVC |
| `dvc push` / `dvc pull` | Sync the cache with the remote |
| `dvc status -c` | Check whether the cache matches what's pushed |
| `git tag -a <name> -m "..."` | Tag an immutable, citable release |

## Practice exercise

- [ ] Initialize Git and DVC in an empty directory and configure a remote
- [ ] Track a file with `dvc add`, commit the pointer, and `dvc push` it
- [ ] Write a `.gitignore` that excludes large data but keeps `!*.dvc`
- [ ] Commit a small QC text file next to a DVC-tracked assembly output
- [ ] Tag a release with the quality numbers in the tag message
- [ ] Clone into a fresh directory, check out the tag, and confirm `dvc pull` reproduces the same file hash

---

Based on [Version Control for Genome Assembly Workflows: Best Practices](https://www.zubairkhalid.com/knowledge/bioinformatics/version-control-genome-assembly-git-dvc) by Dr. Zubair Khalid — see the original for a fuller treatment, including DVC pipelines (`dvc.yaml`/`dvc repro`), branching strategies, and CI checks.
