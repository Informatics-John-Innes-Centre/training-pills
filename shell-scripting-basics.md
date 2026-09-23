# Shell Scripting Basics for Bioinformatics

## Turning commands you type by hand into something reusable

**Time:** about 40–50 minutes
**Level:** Beginner — comfortable running individual shell commands already

By the end of this tutorial, you will be able to:

- Write and run a basic shell script
- Use variables and command substitution
- Accept arguments from the command line
- Loop over multiple samples
- Make a script stop immediately when something goes wrong

The main idea to take away:

> **A script is just the commands you'd type by hand, saved so you — and Slurm — can run them exactly the same way every time, on any sample.**

---

## 1. Why script instead of typing commands?

Typing commands interactively is fine for exploring, but it doesn't scale to "do this for 96 samples," and it isn't reproducible — nobody (including future you) can see exactly what you ran six months from now. A script fixes both problems, and it's also *exactly* what an `sbatch` job is: a script Slurm runs on your behalf (see the `slurm-basics` pill).

## 2. The shebang, and making a script executable

Every script starts with a **shebang** line, telling the shell what should interpret it:

```bash
#!/bin/bash
```

**Make it executable**

```bash
chmod +x align_sample.sh
```

**Run it**

```bash
./align_sample.sh
```

The `./` is required — it tells the shell to look in the current directory rather than your `PATH`.

> **Tip:** if you forget `chmod +x`, you can still run it with `bash align_sample.sh` — but making it executable is worth the one-off habit.

## 3. Variables

```bash
SAMPLE="sample01"
echo "Processing $SAMPLE"
```

No spaces around `=` when assigning. Use `$SAMPLE` (or `${SAMPLE}` when it needs to be unambiguous, e.g. `${SAMPLE}_R1.fastq.gz`) to read it back.

**Command substitution** — capture a command's output into a variable:

```bash
TODAY=$(date +%F)
echo "Run date: $TODAY"
```

**Always quote variables that might contain spaces**, especially filenames:

```bash
cp "$SAMPLE_DIR/aligned.bam" "$RESULTS_DIR/"
```

Without the quotes, a path containing a space would be split into multiple arguments.

## 4. Accepting arguments

Arguments passed on the command line are available as `$1`, `$2`, and so on; `$@` is all of them, and `$#` is how many there are.

```bash
#!/bin/bash
# usage: ./align_sample.sh sample01

SAMPLE="$1"
bwa mem ref.fa "${SAMPLE}_R1.fastq.gz" "${SAMPLE}_R2.fastq.gz" > "${SAMPLE}.sam"
```

```bash
./align_sample.sh sample01
```

This is the same script now reusable for any sample, instead of one script per sample.

## 5. Loops

**Loop over a list of files**

```bash
for f in *.fastq.gz; do
    echo "Found $f"
done
```

**Loop over a list of sample names read from a file** — the same `samples.txt` pattern used by the Slurm array example in `slurm-basics`:

```bash
while read -r SAMPLE; do
    echo "Processing $SAMPLE"
    bwa mem ref.fa "${SAMPLE}_R1.fastq.gz" "${SAMPLE}_R2.fastq.gz" > "${SAMPLE}.sam"
done < samples.txt
```

`read -r` reads one line at a time into `SAMPLE`; `-r` stops backslashes in the file being treated specially.

## 6. Conditionals

```bash
if [ -f "${SAMPLE}.bam" ]; then
    echo "Already aligned, skipping"
else
    bwa mem ref.fa "${SAMPLE}_R1.fastq.gz" "${SAMPLE}_R2.fastq.gz" > "${SAMPLE}.sam"
fi
```

`[ -f file ]` tests whether a file exists — useful for skipping samples that were already processed by an earlier, interrupted run.

**Check whether the previous command succeeded**

```bash
bwa mem ref.fa reads_1.fq reads_2.fq > aligned.sam
if [ $? -ne 0 ]; then
    echo "Alignment failed" >&2
    exit 1
fi
```

`$?` holds the exit code of the last command (`0` = success). `>&2` sends the message to standard error, and `exit 1` stops the script with a non-zero (failure) status — see the `pipes-redirection-basics` pill for why that matters to anything downstream checking with `&&`/`||`.

## 7. Failing loudly instead of silently

By default, a shell script keeps going even after a command fails — which, in a multi-step pipeline, can mean silently calling variants on an empty or half-written BAM file because the alignment step actually failed three lines earlier.

Add this near the top of every script:

```bash
set -euo pipefail
```

- `-e` — exit immediately if any command fails
- `-u` — treat using an undefined variable as an error, instead of silently substituting an empty string
- `-o pipefail` — make a pipeline (`cmd1 | cmd2`) fail if *any* stage fails, not just the last one

```bash
#!/bin/bash
set -euo pipefail

SAMPLE="$1"
bwa mem ref.fa "${SAMPLE}_R1.fastq.gz" "${SAMPLE}_R2.fastq.gz" | samtools sort -o "${SAMPLE}.bam"
samtools index "${SAMPLE}.bam"
```

> **Habit worth building:** put `set -euo pipefail` at the top of every script you write, including — especially — the ones you submit with `sbatch`.

## 8. A realistic example

Putting Sections 2–7 together — a script that loops over samples, skips ones already done, and stops immediately on failure:

```bash
#!/bin/bash
set -euo pipefail

while read -r SAMPLE; do
    if [ -f "${SAMPLE}.bam" ]; then
        echo "Skipping ${SAMPLE}, already done"
        continue
    fi

    echo "Aligning ${SAMPLE}"
    bwa mem ref.fa "${SAMPLE}_R1.fastq.gz" "${SAMPLE}_R2.fastq.gz" \
        | samtools sort -o "${SAMPLE}.bam"
    samtools index "${SAMPLE}.bam"
done < samples.txt
```

This is exactly the kind of script you'd either run directly, or hand to `sbatch` to run on a compute node.

## 9. Your shell scripting survival kit

| Syntax | What it does |
| --- | --- |
| `#!/bin/bash` | The shebang — always the first line |
| `chmod +x script.sh` | Make a script executable |
| `VAR="value"` / `$VAR` | Assign / read a variable (no spaces around `=`) |
| `$(command)` | Command substitution — capture output into a variable |
| `$1`, `$@`, `$#` | First argument, all arguments, argument count |
| `for x in ...; do ... done` | Loop over a list or files |
| `while read -r x; do ... done < file` | Loop over lines in a file |
| `if [ -f file ]; then ... fi` | Test whether a file exists |
| `$?` | Exit code of the last command |
| `set -euo pipefail` | Stop the script immediately on any failure |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Write a script with a shebang, make it executable, and run it with `./`
- [ ] Add a variable and print it with `echo`
- [ ] Use command substitution to store the output of `date` in a variable
- [ ] Write a script that accepts a sample name as `$1` and uses it in a filename
- [ ] Write a `for` loop over a set of test files
- [ ] Write a sample list in a text file and loop over it with `while read -r ... done < file`
- [ ] Add an `if [ -f ... ]` check that skips a step if the output file already exists
- [ ] Add `set -euo pipefail` to a script, then confirm it stops immediately when a command fails
