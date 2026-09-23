# Pipes and Redirection

## Connecting small commands into a pipeline

**Time:** about 25–35 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Connect commands together with a pipe (`|`)
- Redirect output to a file (`>`, `>>`)
- Separate normal output from error messages (`2>`)
- Chain commands conditionally (`&&`, `||`)
- Combine these into real bioinformatics one-liners

The main idea to take away:

> **Almost every bioinformatics tool is designed to read from standard input and write to standard output — pipes and redirection are what let you connect them into a pipeline without writing a script.**

---

## 1. Why this matters

`samtools`, `zcat`, `grep`, `awk`, `sort` — nearly every command-line bioinformatics tool follows the same convention: read from **standard input**, write to **standard output**, and report problems on **standard error**. Once you know that, you can connect tools together freely, instead of needing every step to have its own "output file" option.

```text
zcat sample.fastq.gz | grep -c "^@"
       │                    │
   writes to           reads from
  standard output    standard input
```

## 2. The pipe: `|`

A pipe sends one command's output directly into the next command's input, without ever touching disk.

```bash
zcat sample.fastq.gz | head
```

Streams the decompressed content straight into `head`, showing the first 10 lines.

**Count something**

```bash
zcat sample.fastq.gz | wc -l
```

**Chain more than two commands**

```bash
zcat sample.fastq.gz | grep -A3 "^@READ123" | head -4
```

Each `|` connects one more stage — there's no limit beyond readability.

## 3. Redirecting output to a file: `>` and `>>`

**Overwrite** the destination file with a command's output:

```bash
samtools view sample.bam > sample.sam
```

**Append** to the end of an existing file instead of replacing it:

```bash
echo "Run finished at $(date)" >> run.log
```

> **Danger:** `>` overwrites the destination file completely and silently, even if the command that follows fails. Never do `command file.txt > file.txt` — by the time the command reads the file, `>` has already emptied it. If you need to edit a file in place, use a tool designed for that (e.g. `sed -i`, covered in the `grep-sed-awk-basics` pill).

## 4. Separating errors: `2>`

Every command has two separate output streams: standard output (results) and standard error (problems). By default both print to your terminal, mixed together — but you can send them to different places.

```bash
my_pipeline.sh > results.log 2> errors.log
```

Now `results.log` only has real output, and `errors.log` only has whatever the tool reported as an error — much easier to check "did anything go wrong?" without scrolling through normal output.

**Combine both into a single file**, in order:

```bash
my_pipeline.sh > combined.log 2>&1
```

`2>&1` means "send stream 2 (errors) to wherever stream 1 (output) is currently going."

> **Bioinformatics habit:** when running something long inside a `tmux`/Slurm job, redirecting stderr separately means you can `grep` just `errors.log` afterwards instead of hunting for the word "Error" inside megabytes of normal tool output.

## 5. Chaining commands conditionally: `&&` and `||`

**Run the second command only if the first succeeded** (`&&`):

```bash
samtools sort in.bam -o sorted.bam && samtools index sorted.bam
```

If the sort fails, the index step is skipped — you won't end up indexing a broken or half-written file.

**Run the second command only if the first failed** (`||`):

```bash
mkdir results || echo "Could not create results directory"
```

**Every command's exit code drives this.** By convention, `0` means success and anything else means failure — `&&`/`||` are just checking that number for you.

## 6. Putting it together — real bioinformatics one-liners

**Count reads in a FASTQ file** (4 lines per read)

```bash
zcat sample.fastq.gz | wc -l | awk '{print $1/4}'
```

**Count sequences in a FASTA file**

```bash
grep -c "^>" genome.fasta
```

**Extract mapped reads, sort, and index — stopping if anything fails**

```bash
samtools view -b -F 4 in.bam | samtools sort -o mapped_sorted.bam && samtools index mapped_sorted.bam
```

**Run a pipeline, keeping output and errors separate**

```bash
./run_pipeline.sh > pipeline.log 2> pipeline.err
```

## 7. Your pipes & redirection survival kit

| Syntax | What it does |
| --- | --- |
| `cmd1 \| cmd2` | Send `cmd1`'s output into `cmd2`'s input |
| `cmd > file` | Write output to `file`, overwriting it |
| `cmd >> file` | Append output to the end of `file` |
| `cmd 2> file` | Send error messages to `file` |
| `cmd > out.log 2>&1` | Send both output and errors to the same file |
| `cmd1 && cmd2` | Run `cmd2` only if `cmd1` succeeded |
| `cmd1 \|\| cmd2` | Run `cmd2` only if `cmd1` failed |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Pipe `zcat` on a compressed test file into `head`
- [ ] Count the lines in a file using a pipe into `wc -l`
- [ ] Redirect a command's output into a new file with `>`, then confirm the file's contents
- [ ] Append a second line to that file with `>>` and confirm both lines are present
- [ ] Run a command that fails on purpose (e.g. `ls nonexistent-file`) and redirect its error message to a separate file with `2>`
- [ ] Chain two commands with `&&` so the second only runs if the first succeeds
- [ ] Deliberately make the first command fail and confirm the second one is skipped
