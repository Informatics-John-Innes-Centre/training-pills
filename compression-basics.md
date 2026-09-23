# Working with Compressed Data in Bioinformatics

## Compressing, decompressing, and archiving without the wasted disk space

**Time:** about 30–40 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Explain why compression matters for bioinformatics data
- Compress and decompress files with `gzip`/`gunzip`
- Read compressed files without decompressing them first
- Bundle multiple files/directories with `tar`
- Recognise the bioinformatics-specific compressed formats (BAM, CRAM, `bgzip`) and why they're different from plain `gzip`

The main idea to take away:

> **Most bioinformatics tools can read compressed data directly — so the best way to "decompress" a file is often to not decompress it at all.**

---

## 1. Why compress?

Sequencing data is huge, and most of it compresses extremely well (DNA/RNA sequence and quality strings are very repetitive). Compression gives you:

- Less disk usage on shared, quota-limited storage
- Faster transfers (fewer bytes to move — see the `rsync-basics` pill)
- Less I/O — smaller files are quicker to read off disk

Crucially, almost every mainstream bioinformatics tool (`bwa`, `samtools`, `bcftools`, `seqkit`, `fastqc`, ...) can read `.gz` files **directly**. That means you rarely need to keep an uncompressed copy on disk at all — decompressing a 50 GB FASTQ file just to read it is usually unnecessary work and unnecessary storage.

## 2. `gzip` and `gunzip` — the basics

**Compress a file**

```bash
gzip sample.fastq
```

This replaces `sample.fastq` with `sample.fastq.gz` (the original is removed by default).

**Decompress a file**

```bash
gunzip sample.fastq.gz
```

This replaces `sample.fastq.gz` with `sample.fastq`.

> **Tip:** if you want to keep the original file as well, use `gzip -k sample.fastq` (`-k` = keep).

## 3. Reading compressed files without decompressing

This is the habit worth building early: don't decompress a file just to look at it or pipe it into something else.

**View a compressed file**

```bash
zcat sample.fastq.gz | head
```

`zcat` streams the decompressed content straight to your terminal (or into a pipe) without ever writing an uncompressed copy to disk.

**Search inside a compressed file**

```bash
zcat sample.fastq.gz | grep "N" | head
```

**Feed a compressed file straight into a tool that doesn't accept `.gz` natively**

```bash
zcat sample.fastq.gz | some_tool
```

> **Why this matters:** for a large FASTQ file, decompressing it to disk first can temporarily double your disk usage (compressed + uncompressed) and cost you a lot of I/O time you didn't need to spend. Streaming with `zcat` avoids both.

## 4. `tar` — bundling files and directories

`gzip` compresses a single file. To compress a whole directory (multiple files), you first bundle it into a single archive with `tar`, typically compressing at the same time.

**Create a compressed archive of a directory**

```bash
tar -czvf results.tar.gz results/
```

- `-c` create an archive
- `-z` compress it with gzip
- `-v` verbose (list files as they're added)
- `-f` the archive filename comes next

**Extract a compressed archive**

```bash
tar -xzvf results.tar.gz
```

**Archive without compressing** (useful if the contents are already compressed — see Section 6)

```bash
tar -cvf archive.tar file1 file2
tar -xvf archive.tar
```

| Command | What it does |
| --- | --- |
| `tar -cvf archive.tar file1 file2` | Bundle `file1` and `file2` into `archive.tar` |
| `tar -xvf archive.tar` | Extract the contents of `archive.tar` |
| `tar -czvf archive.tar.gz dir/` | Bundle and compress a directory into `archive.tar.gz` |
| `tar -xzvf archive.tar.gz` | Extract a `.tar.gz` archive |

## 5. Bioinformatics compressed formats aren't all the same

Plain `gzip` is not the only compression you'll meet — and for some formats, plain `gzip` is actually the wrong choice.

- **BAM** (compressed alignments) and **CRAM** (even more compressed alignments) are already compressed binary formats. There is essentially nothing left to gain by running `gzip` on a `.bam` or `.cram` file — you'll spend CPU time for little or no size reduction.
- **`bgzip`** (block gzip, from `htslib`/`samtools`) looks like `gzip` and produces files that *are* readable by ordinary `gunzip`/`zcat` — but it compresses in independent blocks. This is what allows tools like `tabix` to build an index and jump straight to a specific region of a VCF or BED file, instead of having to decompress the whole file to find it.

```bash
bgzip variants.vcf        # produces variants.vcf.gz, block-compressed
tabix -p vcf variants.vcf.gz   # builds variants.vcf.gz.tbi, enabling random access
```

> **Rule of thumb:** if a tool asks for a `bgzip`-compressed + `tabix`-indexed file (common for VCF/BED/GFF), plain `gzip` will not work correctly for indexed/random access — use `bgzip`, not `gzip`, for those formats.

## 6. Don't compress what's already compressed

Running `gzip` (or `-z` in `tar`, or `-z` in `rsync`) on data that's already compressed — `.bam`, `.cram`, `.fastq.gz`, `.vcf.gz` — wastes CPU time for little or no size reduction, because there's very little redundancy left to squeeze out.

```bash
# Not useful — bam is already compressed
gzip aligned.bam

# Fine — bundling already-compressed files together, without re-compressing them
tar -cvf aligned_files.tar aligned1.bam aligned2.bam
```

If you're archiving a mix of compressed and uncompressed files, it's usually simplest to compress the uncompressed ones individually first, then `tar` everything together without `-z`.

## 7. Compressing large files faster

Standard `gzip` only uses a single CPU core, which can make compressing a very large file (e.g. a raw FASTQ before it's been touched by any pipeline) slow. If your server has multiple cores available, `pigz` (parallel gzip) does the same job across several cores:

```bash
pigz sample.fastq
```

It produces a normal `.gz` file, fully compatible with `gunzip`/`zcat` — it's just faster to create.

## 8. Checking a compressed file is intact

Before deleting an uncompressed original, it's worth confirming the compressed version isn't corrupted:

```bash
gzip -t sample.fastq.gz
```

No output means the file is fine; an error means something went wrong during compression or transfer.

## 9. Your compression survival kit

| Command | What it does |
| --- | --- |
| `gzip file` | Compress a file (removes the original) |
| `gzip -k file` | Compress a file, keeping the original too |
| `gunzip file.gz` | Decompress a `.gz` file |
| `zcat file.gz` | Stream a compressed file's contents without decompressing to disk |
| `gzip -t file.gz` | Test a compressed file for corruption |
| `tar -czvf out.tar.gz dir/` | Bundle and compress a directory |
| `tar -xzvf out.tar.gz` | Extract a `.tar.gz` archive |
| `bgzip file` + `tabix -p <type> file.gz` | Block-compress and index a VCF/BED/GFF for random access |
| `pigz file` | Parallel `gzip` — faster on multi-core machines |

**The habit to remember:**

```text
Need to look inside a .gz file?     → zcat file.gz | ...   (don't decompress to disk)
Need to run a tool on it?            → most tools accept .gz directly — check first
Need random access (VCF/BED)?        → bgzip + tabix, not plain gzip
Already compressed (BAM/CRAM/.gz)?   → don't gzip it again
```

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Compress a test FASTQ (or any text file) with `gzip`
- [ ] Decompress it again with `gunzip`
- [ ] Compress it again, this time keeping the original with `gzip -k`
- [ ] Use `zcat file.gz | head` to view the first few lines without decompressing
- [ ] Bundle a small directory into a `.tar.gz` with `tar -czvf`
- [ ] Extract it into a new location with `tar -xzvf`
- [ ] Run `gzip -t` on one of your compressed files to confirm it's intact
- [ ] If available, try `bgzip` + `tabix` on a small VCF file and confirm you can query a single region without decompressing the whole file
