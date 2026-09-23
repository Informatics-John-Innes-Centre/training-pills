# Finding Files with `find`

## Locating exactly the files you need across large project directories

**Time:** about 20–30 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Search a directory tree for files matching a name or pattern
- Filter by file type, size, or modification time
- Safely preview matches before acting on them
- Run a command on every match with `-exec` — including deleting or moving matches into a folder

The main idea to take away:

> **`find` searches by what a file actually is (name, type, size, age) rather than where you remember putting it — essential once a project has hundreds of samples and subdirectories.**

---

## 1. Why `find`?

Bioinformatics projects tend to sprawl: one directory per sample, several output files per tool, results scattered across scratch and archive storage. `ls` only shows you one directory at a time. `find` searches recursively through an entire directory tree in one command.

```text
project/
├── sample01/aligned.bam
├── sample02/aligned.bam
├── sample03/aligned.bam
...
├── sample147/aligned.bam
```

```bash
find project/ -name "aligned.bam"
```

finds all 147, without you writing a loop or opening a single folder.

## 2. Basic syntax

```bash
find [path] [options] [expression]
```

**Find by name**

```bash
find . -name "*.fastq.gz"
```

Searches the current directory (`.`) and everything below it for files matching the pattern. Quote the pattern so your shell doesn't try to expand the `*` itself.

**Find by name, ignoring case**

```bash
find . -iname "*.fastq"
```

`-iname` matches regardless of case — useful when you're not sure whether an extension was written `.BAM`, `.bam`, or something a collaborator typed differently.

> **Tip:** `.` always means "the current directory" in `find`, same as everywhere else in the shell.

## 3. Filtering by type

```bash
find . -type f -name "*.vcf.gz"   # files only
find . -type d -name "results"    # directories only
```

`-type f` restricts matches to regular files, `-type d` to directories — useful when a sample name might match both a directory and a file inside it.

## 4. Filtering by size

Handy for tracking down what's actually using up your storage quota:

```bash
find . -type f -size +100M
```

Finds every file larger than 100 MB. Tighten or loosen the threshold as needed:

```bash
find . -type f -size +1M
```

Sizes accept `k` (KB), `M` (MB), or `G` (GB) — `find . -type f -name "*.bam" -size +5G` narrows this down to large BAM files specifically.

## 5. Filtering by age

```bash
find . -type f -mtime +5000
```

Finds files that haven't been modified in the last 5000 days (~14 years) — the kind of ancient, forgotten file that's often a safe candidate for archiving or cleanup.

```bash
find . -name "*.bam" -mtime -7
```

The `-` means "less than" — here, BAM files modified in the *last* 7 days (e.g. from this week's run). `-mtime +N` means "more than N days ago."

## 6. Combining filters — real bioinformatics examples

The real power of `find` shows up once you combine several filters in one command. A few patterns that come up constantly in bioinformatics work:

**Find large, old files eating into your scratch quota — good candidates to archive or delete**

```bash
find scratch/ -type f -size +1G -mtime +90
```

Large files (over 1 GB) that haven't been touched in 90 days — a much better starting point for cleanup than guessing.

**Find raw FASTQ files that haven't been compressed yet**

```bash
find . -name "*.fastq" ! -name "*.gz"
```

`!` negates the following test — this finds `.fastq` files that are *not* also matching `*.gz`, i.e. reads still sitting around uncompressed (see the `compression-basics` pill for why that's worth fixing).

**Find one mate of paired-end reads**

Illumina paired-end data is usually named with `_R1_`/`_R2_` (or `_1`/`_2`). To iterate over samples without processing each pair twice, search for just one mate:

```bash
find . -name "*_R1_*.fastq.gz"
```

Each match gives you one sample; you can derive the R2 filename by substituting `_R1_` for `_R2_` inside a script.

## 7. Limiting how deep it searches

For a large project tree, `find` can be slower than necessary if it recurses further than you need:

```bash
find . -maxdepth 1 -name "*.fastq.gz"
```

Only looks in the current directory, not subdirectories — closer to what `ls` would show you, but still pattern-matched.

## 8. Acting on what you find

**Preview first — always**

Before deleting or modifying anything, run the `find` on its own and read the list of matches:

```bash
find . -name "*.tmp"
```

**Delete matches (only after you've checked the list above)**

```bash
find . -name "*.tmp" -delete
```

**Move matches into a folder** — e.g. tidying old logs into an `archive/` subfolder instead of deleting them outright

```bash
mkdir -p archive/
find . -maxdepth 1 -name "*.log" -exec mv {} archive/ \;
```

**Run a command on every match with `-exec`**

```bash
find . -name "*.bam" -exec samtools index {} \;
```

`{}` is replaced with each matching filename in turn, and `\;` ends the `-exec` command.

**The same thing, often faster for many files, using `xargs`**

```bash
find . -name "*.bam" | xargs -I{} samtools index {}
```

**Generate checksums for everything you find** (ties together with the `md5sum-basics` pill)

```bash
find . -name "*.fastq.gz" -exec md5sum {} \; > checksums.txt
```

> **Safety habit:** never bolt `-delete` or `-exec rm` onto a `find` command you haven't already run and inspected without it. It's very easy to match more than you intended.

## 9. A typical bioinformatics workflow

```text
find scratch/ -type f -size +1G -mtime +90
        ↓ (inspect the list)
find scratch/ -name "*.fastq" ! -name "*.gz" -exec pigz {} \;
        ↓ (compress anything still raw)
find scratch/ -name "*.fastq.gz" -exec md5sum {} \; > checksums.txt
        ↓ (checksum before you archive or delete anything)
rsync -a -P scratch/project/ archive/project/
```

Locate what's large and old, compress what's still uncompressed, checksum it, then archive — each pill in this series covers one link in that chain.

## 10. Your `find` survival kit

| Command | What it does |
| --- | --- |
| `find . -name "*.ext"` | Find files matching a pattern, recursively |
| `find . -iname "*.ext"` | Same, case-insensitive |
| `find . -type f` / `-type d` | Restrict to files / directories |
| `find . -size +100M` | Find files larger than a given size (`k`/`M`/`G`) |
| `find . -mtime -7` / `+30` | Find files modified in the last / more than N days |
| `find . -maxdepth 1 ...` | Don't recurse past a given depth |
| `find . -name "*.ext" -delete` | Delete matches (preview first!) |
| `find . -name "*.ext" -exec CMD {} \;` | Run `CMD` on every match |
| `find . -name "*.ext" -exec mv {} dest/ \;` | Move matches into another folder |
| `find . -size +1G -mtime +90` | Large + old files — cleanup/archiving candidates |
| `find . -name "*.fastq" ! -name "*.gz"` | Files not yet compressed |
| `find . -name "*_R1_*.fastq.gz"` | One entry per paired-end sample |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Create a small test directory tree with a few subdirectories and files of different extensions
- [ ] Find all files with a given extension using `find . -name "*.ext"`
- [ ] Restrict a search to directories only with `-type d`
- [ ] Find files above a certain size with `-size`
- [ ] Find files modified in the last few minutes with `-mmin -5`
- [ ] Preview a `-delete` candidate list before ever adding `-delete`
- [ ] Use `-exec` to run a simple command (e.g. `ls -l {}`) on every match
- [ ] Use `-exec ... mv {} dest/ \;` to move matches into a new folder instead of deleting them
- [ ] Combine `-size` and `-mtime` to shortlist large, old files as cleanup candidates
- [ ] Use `!` to find files matching one pattern but not another (e.g. uncompressed vs compressed)
- [ ] Generate a `checksums.txt` for a set of matches using `find ... -exec md5sum {} \;`
