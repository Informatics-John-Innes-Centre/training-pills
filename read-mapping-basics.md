# Read Mapping with BWA and samtools

## From FASTQ reads to a sorted, indexed BAM on linux-playground

**Time:** about 40–50 minutes
**Level:** Beginner — assumes you can move around the command line ([Linux](linux-basics.md) pill); the [md5sum](md5sum-basics.md) and [pipes](pipes-redirection-basics.md) pills help

By the end of this tutorial, you will be able to:

- Download and unpack a training dataset from the Training Repo
- Check that the files arrived intact with `md5sum -c`
- Read a FASTQ file and a FASTA reference from the command line
- Index a reference with `bwa index`
- Map paired-end reads with `bwa mem` and recognise a SAM file
- Turn a SAM file into a sorted, indexed BAM with `samtools`
- Check how well the mapping went with `samtools flagstat`
- Look for variation at the terminal with `samtools tview`, and open your BAMs in IGV
- Map every sample in one loop

The main idea to take away:

> **Mapping is always the same three steps: index the reference once, map each sample's reads to it, then sort and index the result. Everything downstream (IGV, variant calling) reads that sorted, indexed BAM.**

---

## 1. The exercise: looking for progeny mutants

A resistant parent line was treated with **EMS**, a chemical that causes random point mutations (mostly `G` → `A` and `C` → `T`). Six mutant progeny were sequenced, along with the parent, over a small set of candidate genes. Your job is to map every sample to the reference and look for positions where a mutant differs from the parent.

| Sample | What it is |
| --- | --- |
| `5406` | The resistant parent |
| `M49S1`, `M52S1`, `M55S1`, `M57S1`, `M59S1`, `M61S1` | EMS mutants |

Each sample is **paired-end** Illumina MiSeq data, with reads of 301 bases: `_R1` and `_R2` are the two ends of the same DNA fragments, so the two files always go into the mapper together.

## 2. Log in and set up your folder

Connect to **linux-playground** with PuTTY (Windows) or a terminal (Mac, Linux, [WSL](wsl-basics.md)). First time? The [Linux Playground](linux-playground.md) pill walks through logging in and linking your U drive.

```bash
ssh <your-username>@linux-playground.nbi.ac.uk
```

> **In a class?** Open a second PuTTY window or terminal, log in again and type `atraining`. That window then shows the trainer's screen, read-only, so you can follow along in one window and type in the other. Press `Ctrl-b` then `d` to leave it.

So that you can open your results in Windows later, first create a folder called `informatics` in your **U drive** (using File Explorer). Then, on linux-playground:

```bash
mountu                 # links your U drive's informatics folder to ~/u_drive (asks for your password)
ls ~/u_drive           # create a text file in Windows and check it appears here
mkdir ~/u_drive/intro
cd ~/u_drive/intro
```

> **Why the U drive?** Anything you write in `~/u_drive` lands in `U:\informatics` on your PC. That is how you will open your BAM files in IGV at the end, without copying anything.

## 3. Download the data

The dataset is on the [Training Repo](https://repo.informatics.jic.ac.uk/bioinformatics26/) as a single zip file:

```bash
wget https://repo.informatics.jic.ac.uk/bioinformatics26/data.zip
unzip data.zip
cd data
ls *
```

```text
mappings:
5406.bam  5406.bam.bai  M49S1.bam  ...  refSNP.fasta.amb  refSNP.fasta.ann  ...

nr_pep:
mynr.fasta.gz

rawdata:
5406_R1.fastq.gz  5406_R2.fastq.gz  M49S1_R1.fastq.gz  ...  md5sums.txt

reference:
refSNP.fasta  refSNP.fasta.fai
```

- `rawdata/`: the reads, two FASTQ files per sample, plus `md5sums.txt`
- `reference/`: the reference sequence (`refSNP.fasta`)
- `mappings/`: **finished results**, so you can compare your own BAMs against them, or skip ahead if you get stuck
- `nr_pep/`: protein sequences for a later exercise; not used here

## 4. Check the downloads with md5sum

A checksum is a fingerprint of a file: if even one byte changed in transit, the fingerprint changes. `md5sums.txt` holds the fingerprints of the original files, and `-c` checks yours against them:

```bash
cd rawdata
cat md5sums.txt
md5sum -c md5sums.txt
cd ..
```

```text
5406_R1.fastq.gz: OK
5406_R2.fastq.gz: OK
M49S1_R1.fastq.gz: OK
...
```

Every line should say `OK`. Anything else means the file is damaged: download it again before going further. The [md5sum](md5sum-basics.md) pill covers this in more detail.

## 5. Look at the data first

**The reads (FASTQ).** Four lines per read: a name, the sequence, a `+`, and one quality letter per base. The files are gzipped, so read them with `zcat` (see the [compression](compression-basics.md) pill):

```bash
zcat rawdata/5406_R1.fastq.gz | head -4
```

```text
@HWI-M01242:86:000000000-AC0YE:1:1116:18512:10339
ATCCACGTTGATGAACACGGCATCCAGCTACCTGCTGGACCAGTACAAGGTGATGGATGG...
+
CCCCCGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGGG...
```

How many reads? Count the lines and divide by 4:

```bash
echo $(( $(zcat rawdata/5406_R1.fastq.gz | wc -l) / 4 ))
```

```text
1873
```

**The reference (FASTA).** A `>` header line, then the sequence. The `.fai` index lists every sequence and its length:

```bash
head -3 reference/refSNP.fasta
cut -f1,2 reference/refSNP.fasta.fai
```

```text
contig_1791_1   3251
contig_1793_1   1278
contig_1795_1   5266
contig_1796_1   3757
contig_1797_1   3704
```

Five contigs, about 17 kb in total: tiny on purpose, so every step below takes seconds. A real genome works the same way, only slower.

## 6. Check your tools

```bash
bwa              # prints its version and the list of commands
samtools --version
```

If either says `command not found`, ask the trainer, or load it as you would on the HPC (see the [HPC Software](hpc-software-basics.md) pill).

> **Stuck at any step?** Ask `biowizard`, linux-playground's built-in helper, in plain English: `biowizard why did that fail?` See the [Linux Playground](linux-playground.md) pill for how it works.

## 7. Index the reference (once)

Before BWA can map anything, it builds an index of the reference, a bit like the index at the back of a book, so that it can find where each read belongs quickly:

```bash
bwa index reference/refSNP.fasta
ls reference/
```

```text
refSNP.fasta  refSNP.fasta.amb  refSNP.fasta.ann  refSNP.fasta.bwt  refSNP.fasta.fai  refSNP.fasta.pac  refSNP.fasta.sa
```

The five new files (`.amb .ann .bwt .pac .sa`) are the index. Keep them next to the FASTA: BWA finds them by name. You index a reference **once** and reuse it for every sample.

## 8. Map one sample with bwa mem

```bash
mkdir mymappings
bwa mem reference/refSNP.fasta rawdata/5406_R1.fastq.gz rawdata/5406_R2.fastq.gz > mymappings/5406.sam
```

`bwa mem` writes its results to the screen, so `>` redirects them into a file. Its progress messages go to the screen separately; they are not errors.

**What's in a SAM file?** A header (lines starting `@`), then one line per read: where it mapped and how.

```bash
grep -v '^@' mymappings/5406.sam | head -2 | cut -f1-9
```

```text
HWI-M01242:86:000000000-AC0YE:1:1116:18512:10339  99   contig_1795_1  998   60  301M  =  1343   608
HWI-M01242:86:000000000-AC0YE:1:1116:18512:10339  147  contig_1795_1  1343  60  263M  =  998   -608
```

Both lines are the same pair: the first read maps forward at position 998, its mate in reverse at 1343, 608 bases of DNA fragment in total.

| Column | Meaning |
| --- | --- |
| 1 | Read name (the same for both reads of a pair) |
| 2 | **Flag**: a number encoding paired/unmapped/reverse strand, etc. |
| 3 | Contig the read mapped to (`*` if unmapped) |
| 4 | Position on that contig |
| 5 | **Mapping quality** (MAPQ): how sure BWA is of the placement (60 = certain) |
| 6 | **CIGAR**: how the read lines up, e.g. `301M` = all 301 bases aligned; `263M` = a shorter (trimmed) read, all aligned; `283M18S` = 283 aligned and the last 18 soft-clipped (left unaligned) |
| 7–9 | Where its mate mapped, and the fragment length |
| 10–11 | The read's sequence and qualities |

## 9. Sort and index with samtools

SAM is plain text: easy to read, but large and slow. Tools like IGV need a **BAM**: the same data, compressed, **sorted by position**, and **indexed** so that a viewer can jump straight to any region:

```bash
samtools sort -o mymappings/5406.bam mymappings/5406.sam
samtools index mymappings/5406.bam
ls mymappings/
```

```text
5406.bam  5406.bam.bai  5406.sam
```

The `.bai` file is the index. Keep it next to the BAM. Once you have the BAM, the SAM is no longer needed:

```bash
rm mymappings/5406.sam
```

> **Reading a BAM.** BAM is binary, so `head` shows gibberish. Use `samtools view` instead: `samtools view mymappings/5406.bam | head`, or add a region to see only the reads there: `samtools view mymappings/5406.bam contig_1791_1:1000-1100`.

## 10. How well did it map?

```bash
samtools flagstat mymappings/5406.bam
```

```text
3746 + 0 in total (QC-passed reads + QC-failed reads)
...
3737 + 0 mapped (99.76% : N/A)
...
3734 + 0 properly paired (99.68% : N/A)
```

The lines to look at:

- **`in total`**: 3,746 reads, which is 1,873 pairs × 2. If this doesn't match your FASTQ count, a file was cut short.
- **`mapped`**: the percentage of reads that found a place on the reference: 99.76% here, because the reads were selected for these genes. On real data, a low number points to contamination, the wrong reference, or poor-quality reads.
- **`properly paired`**: both reads of a pair mapped facing each other at a sensible distance, the best evidence that a placement is right.

Compare yours with the provided result: `samtools flagstat mappings/5406.bam`.

## 11. All samples at once

Typing four commands per sample gets old fast. Two improvements:

**Skip the SAM file.** Pipe `bwa mem` straight into `samtools sort` (see the [pipes](pipes-redirection-basics.md) pill). Nothing is written until the sorted BAM:

```bash
bwa mem reference/refSNP.fasta rawdata/M49S1_R1.fastq.gz rawdata/M49S1_R2.fastq.gz \
  | samtools sort -o mymappings/M49S1.bam
samtools index mymappings/M49S1.bam
```

**Loop over the samples.** Each sample's name is everything before `_R1.fastq.gz` (see the [shell scripting](shell-scripting-basics.md) pill):

```bash
for R1 in rawdata/*_R1.fastq.gz; do
    SAMPLE=$(basename "$R1" _R1.fastq.gz)
    echo "Mapping $SAMPLE"
    bwa mem reference/refSNP.fasta "rawdata/${SAMPLE}_R1.fastq.gz" "rawdata/${SAMPLE}_R2.fastq.gz" \
      | samtools sort -o "mymappings/${SAMPLE}.bam"
    samtools index "mymappings/${SAMPLE}.bam"
done
ls mymappings/
```

You should end up with a `.bam` and a `.bam.bai` for all seven samples, each with about 99% of reads mapped.

> **On real data:** add `-t 4` to `bwa mem` (and `-@ 4` to `samtools sort`) to use 4 CPUs, and run it as a [Slurm](slurm-basics.md) job on the HPC rather than on a shared machine.

## 12. Look for variation

**At the terminal: `samtools tview`.** A text-mode viewer that shows the reads stacked under the reference:

```bash
samtools tview mymappings/M49S1.bam reference/refSNP.fasta
```

- Dots (`.` and `,`) are bases that match the reference on the forward and reverse strand. A **letter** is a mismatch.
- A column where most reads show the same letter is a likely mutation; a lone letter here and there is usually a sequencing error.
- Press `g` and type a position (e.g. `contig_1791_1:1000`) to jump there, `?` for help, `q` to quit.

Open the parent (`5406.bam`) at the same position: if the change is in a mutant but **not** in the parent, it's a candidate EMS mutation.

**Not every difference is a mutation.** While you look, you'll also see:

- **Differences shared by every sample, parent included.** The parent line isn't identical to the reference, so these were there before EMS. Not what you're looking for.
- **Messy columns near the ends of a contig**, where few reads overlap and many are only partly aligned. Treat these with suspicion.
- **A single odd letter** in an otherwise clean column: a sequencing error.

A real EMS mutation here is in **every read** of one mutant (these lines are inbred, so it is homozygous) and in **none** of the parent's.

**In IGV (easier).** Because you worked in `~/u_drive`, your files are already in `U:\informatics\intro\data` on your PC. In [IGV](https://igv.org):

1. **Genomes → Load Genome from File**: `reference\refSNP.fasta`
2. **File → Load from File**: select the parent and a few mutants from `mymappings\` (the `.bam` files; IGV finds the `.bai` next to them)
3. Pick a contig from the drop-down and scan along: mismatches show as coloured bars in the coverage track.

## 13. A few habits worth building

- **Check md5sums before anything else.** A truncated download can still map, and give you wrong answers without any error.
- **Look at your data before running tools on it**: `head` the FASTQ, count the reads, list the contigs.
- **Index the reference once**, next to the FASTA, and reuse it.
- **Keep BAMs, not SAMs.** Sort, index, then delete the SAM, or pipe straight to `samtools sort`.
- **Always run `samtools flagstat`** after mapping: a low mapping rate is the first sign something is wrong.

## 14. Your read mapping survival kit

| Command | What it does |
| --- | --- |
| `wget <url>` / `unzip file.zip` | Download and unpack the data |
| `md5sum -c md5sums.txt` | Check every file against its expected checksum |
| `zcat reads.fastq.gz \| head -4` | Look at the first read of a gzipped FASTQ |
| `bwa index ref.fasta` | Build the BWA index (once per reference) |
| `bwa mem ref.fasta R1.fastq.gz R2.fastq.gz > out.sam` | Map paired-end reads |
| `samtools sort -o out.bam out.sam` | Convert to BAM, sorted by position |
| `samtools index out.bam` | Create the `.bai` index |
| `bwa mem ... \| samtools sort -o out.bam` | Map and sort in one go, no SAM file |
| `samtools flagstat out.bam` | Mapping statistics |
| `samtools view out.bam contig:start-end` | Show the reads in one region |
| `samtools tview out.bam ref.fasta` | Browse the alignment at the terminal |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Log in to linux-playground, run `mountu`, and create `~/u_drive/intro`
- [ ] Download and unzip `data.zip`, then check every FASTQ with `md5sum -c`
- [ ] Count the reads in `M59S1_R1.fastq.gz`, and check `M59S1_R2.fastq.gz` has the same number
- [ ] Index `refSNP.fasta` and list the files `bwa index` created
- [ ] Map `5406` step by step (SAM → sorted BAM → index), then delete the SAM
- [ ] Find a read in the SAM with MAPQ 60 and one with a CIGAR other than `301M`
- [ ] Run `samtools flagstat` on your BAM and on `mappings/5406.bam`: do they agree?
- [ ] Map all seven samples with the loop in Section 11
- [ ] In `samtools tview` or IGV, find the mutation in each mutant, and note its contig, position and base change
- [ ] Find one difference that is **not** a mutation, and explain why

## Answers

Check yourself after the last exercise. Each mutant carries its own mutation, and all six fall in the same contig, `contig_1795_1`: a strong hint that it is the gene behind the resistance.

| Mutant | Position | Change |
| --- | --- | --- |
| `M49S1` | `contig_1795_1:2319` | G → T |
| `M52S1` | `contig_1795_1:1436` | G → A |
| `M55S1` | `contig_1795_1:2417` | G → A |
| `M57S1` | `contig_1795_1:1489` | C → T |
| `M57S1` | `contig_1797_1:1950` | C → T |
| `M59S1` | `contig_1795_1:1587` | G → A |
| `M61S1` | `contig_1795_1:3999` | C → T |

Nearly all are `G` → `A` or `C` → `T`, the typical EMS signature. `contig_1793_1:753` (G → T) is in **every** sample, parent included: a difference between this line and the reference, not a mutation.
