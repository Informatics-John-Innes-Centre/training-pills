# grep, sed, and awk for Bioinformatics

## Searching, editing, and extracting columns from plain-text data

**Time:** about 40–50 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Search text files for patterns with `grep`
- Perform simple find-and-replace edits with `sed`
- Extract and work with columns of data using `awk`
- Combine all three into a single pipeline over a real bioinformatics file

The main idea to take away:

> **`grep` finds the lines you want, `sed` edits them, `awk` works with their columns — together they cover most of what people otherwise reach for a full script to do.**

---

## 1. Why these three tools?

A huge amount of bioinformatics data is plain text with a simple structure: VCF, SAM, BED, GFF, FASTA, sample sheets, pipeline logs. `grep`/`sed`/`awk` let you search, edit, and extract from these files directly on the command line — often faster than opening a script, and perfectly suited to a quick check on a remote server.

## 2. `grep` — finding lines

**Basic search**

```bash
grep "ERROR" pipeline.log
```

Prints every line containing `ERROR`.

**Case-insensitive**

```bash
grep -i "error" pipeline.log
```

**Count matches instead of printing them**

```bash
grep -c "^>" genome.fasta
```

Counts sequences in a FASTA file, by counting lines starting with `>`.

**Show line numbers**

```bash
grep -n "ERROR" pipeline.log
```

**Invert the match — show lines that *don't* match**

```bash
grep -v "^#" variants.vcf
```

Skips VCF header lines (which start with `#`), leaving just the variant records.

**Show surrounding context**

```bash
grep -B2 -A2 "ERROR" pipeline.log
```

Shows 2 lines before (`-B`) and after (`-A`) each match — useful for seeing what led up to an error.

**Extended regular expressions**

```bash
grep -E "chr(1|2|X)" regions.bed
```

`-E` enables patterns like alternation (`|`), so you can match several options at once.

## 3. `sed` — editing lines

**Substitute the first match on each line**

```bash
sed 's/old/new/' file.txt
```

**Substitute every match on each line** (add `g` for "global")

```bash
sed 's/old/new/g' file.txt
```

**A real example — stripping a `chr` prefix from a BED file**

```bash
sed 's/^chr//' regions.bed
```

**Edit the file in place**, instead of just printing the result

```bash
sed -i 's/^chr//' regions.bed
```

> **Caution:** `-i` overwrites the file immediately, with no undo. Run the command without `-i` first to check the output looks right, or use `sed -i.bak '...' file` to keep a backup copy (`file.bak`) alongside the edited original.

**Print only a range of lines**

```bash
sed -n '5,10p' file.txt
```

`-n` suppresses normal output, and `p` prints only lines 5 through 10.

**Delete lines matching a pattern**

```bash
sed '/^#/d' variants.vcf
```

(Equivalent to `grep -v "^#"` from Section 2 — the same job, different tool.)

## 4. `awk` — working with columns

By default, `awk` splits each line into fields on whitespace: `$1` is the first field, `$2` the second, and so on; `$0` is the whole line; `NF` is the number of fields on the current line.

**Print one column**

```bash
awk '{print $1}' samples.tsv
```

**Use tabs explicitly as the separator** (important for real tab-separated files, where a run of spaces could otherwise confuse the default whitespace splitting)

```bash
awk -F'\t' '{print $2}' samples.tsv
```

**Filter rows by a column's value**

```bash
awk -F'\t' '$7=="PASS"' variants.vcf
```

Prints only lines where the 7th column (the VCF `FILTER` field) is exactly `PASS`.

**Print specific columns, only for matching rows**

```bash
awk -F'\t' '$7=="PASS" {print $1, $2, $6}' variants.vcf
```

Prints chromosome, position, and quality score, only for `PASS` variants.

**Sum a column**

```bash
awk '{sum += $2} END {print sum}' coverage.txt
```

`END { }` runs once, after every line has been processed — a common pattern for totals.

**Combine with `sort`/`uniq` for a quick tally**

```bash
awk '{print $1}' regions.bed | sort | uniq -c
```

Counts how many regions fall on each chromosome.

> **Tip:** `awk` syntax is famously fiddly — field separators, quoting, `BEGIN`/`END` blocks — and it's one of the easiest things to get an AI assistant to write or explain for you. Describe what you want in plain English ("print the second column where the fourth column is greater than 30, tab-separated"), ask it to generate the `awk` one-liner, then run it on a small test file to confirm it does what you expect before trusting it on the real data. The same goes for a fiddly `sed` substitution or regular expression you're not confident about — there's no need to memorise the syntax when you can ask for help and verify the result.

## 5. Putting it together — a real pipeline

**Extract `PASS` variants from a VCF, keep chromosome/position/quality, and count them**

```bash
grep -v "^#" variants.vcf | awk -F'\t' '$7=="PASS" {print $1, $2, $6}' | wc -l
```

Read right to left through what each stage does:

```text
grep -v "^#"                    → drop header lines
  | awk -F'\t' '$7=="PASS" ...'  → keep only PASS variants, print 3 columns
  | wc -l                        → count the resulting lines
```

This is the same "small tools connected with pipes" idea from the `pipes-redirection-basics` pill, applied to a real file.

## 6. Your grep/sed/awk survival kit

| Command | What it does |
| --- | --- |
| `grep "pattern" file` | Print matching lines |
| `grep -c "pattern" file` | Count matching lines |
| `grep -v "pattern" file` | Print lines that *don't* match |
| `grep -n` / `-i` | Show line numbers / ignore case |
| `sed 's/old/new/g' file` | Replace all matches on each line |
| `sed -i 's/old/new/g' file` | Same, editing the file in place |
| `sed -n '5,10p' file` | Print only a range of lines |
| `awk '{print $1}' file` | Print the first column |
| `awk -F'\t' '{print $2}' file` | Same, with tab as the field separator |
| `awk '$3 > 30'` | Print rows where column 3 is greater than 30 |
| `awk '{sum+=$1} END{print sum}'` | Sum a column |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Use `grep` to count how many sequences are in a test FASTA file
- [ ] Use `grep -v` to strip header lines (`^#`) from a test VCF or similar file
- [ ] Use `sed 's/.../.../g'` to replace a word in a test file, without `-i` first
- [ ] Re-run the same `sed` command with `-i` to edit the file directly
- [ ] Use `awk` to print a single column from a tab-separated test file
- [ ] Use `awk` with a condition (`$N == "value"`) to filter rows
- [ ] Combine `grep`, `awk`, and `wc -l` in one pipeline to count matching rows from a raw file
