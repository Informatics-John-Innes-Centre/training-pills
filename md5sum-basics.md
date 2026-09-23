# Checking File Integrity with md5sum

## Making sure your data arrived exactly as it left

**Time:** about 20–30 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Explain why checksums matter for bioinformatics data
- Calculate a checksum for a file
- Generate a checksum file for many files at once
- Verify a set of files against a checksum file
- Avoid the most common checksum mistake

The main idea to take away:

> **A checksum is a fingerprint of a file's exact contents — if two files have the same checksum, they are byte-for-byte identical; if not, something changed.**

---

## 1. Why checksums matter

Sequencing data typically travels a long road: sequencer → core facility storage → download or transfer to your server → archive. At multi-GB/TB scale, silent corruption during a transfer or a copy is rare but real — and unlike a crashed transfer, silent corruption doesn't announce itself.

`md5sum` calculates a short fingerprint (a 128-bit hash) for a file's contents. Recalculating it later and comparing tells you, with very high confidence, whether the file is still exactly what it was.

This matters most when:

- A sequencing facility delivers raw data along with a checksums file — always verify before you start analysing or delete anything else
- You've moved a large archive to another server or storage tier and want to confirm nothing was corrupted
- You're about to delete the "original" copy of something after copying it elsewhere

## 2. Calculating a checksum

```bash
md5sum sample.fastq.gz
```

```text
3b3a...9f2c  sample.fastq.gz
```

The long hexadecimal string is the checksum. Two files with identical contents will always produce the same checksum, regardless of filename or location.

## 3. Checksums for many files at once

For a whole directory of files, generate one checksum file covering all of them:

```bash
md5sum *.fastq.gz > checksums.txt
```

`checksums.txt` now contains one line per file — the checksum and the filename — and travels with the data as its integrity record.

## 4. Verifying against a checksum file

Once you (or a sequencing facility) have a `checksums.txt`, verify the files against it:

```bash
md5sum -c checksums.txt
```

```text
sample1.fastq.gz: OK
sample2.fastq.gz: OK
sample3.fastq.gz: FAILED
```

- `OK` — the file's current contents match the recorded checksum
- `FAILED` — the file has changed, is corrupted, or is missing entirely

> **Practice:** always run `md5sum -c checksums.txt` right after a transfer, before you delete the source copy or start relying on the destination copy.

## 5. The gotcha: compression changes the checksum

A checksum is a fingerprint of the file's exact bytes — not of the "same information." If you decompress, recompress, or otherwise re-encode a file, its checksum changes, even though the underlying sequence data is unchanged.

```bash
md5sum sample.fastq.gz     # one checksum
gunzip sample.fastq.gz
md5sum sample.fastq        # a completely different checksum — this is expected
```

This isn't a corruption — it's a different file. The rule to remember:

> **Always compute and compare checksums on the same representation of the file.** If a sequencing facility gives you checksums for the `.gz` files, verify against the `.gz` files — don't decompress first and expect a match.

## 6. Your md5sum survival kit

| Command | What it does |
| --- | --- |
| `md5sum file` | Print the checksum for one file |
| `md5sum *.fastq.gz > checksums.txt` | Generate checksums for many files at once |
| `md5sum -c checksums.txt` | Verify files against a checksum file (`OK`/`FAILED`) |

**The workflow to remember:**

```text
Before transfer:   md5sum *.fastq.gz > checksums.txt
   ↓
   ... move / copy / archive the data ...
   ↓
After transfer:    md5sum -c checksums.txt
   ↓
   All OK?  → safe to delete the source copy
   Any FAILED?  → re-transfer before deleting anything
```

> **Note:** `sha256sum` works identically (`sha256sum file`, `sha256sum -c checksums.txt`) and is cryptographically stronger — some facilities provide `.sha256` files instead of `.md5`. For integrity-checking against accidental corruption (not malicious tampering), either is fine; use whichever the data provider gives you.

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Calculate the checksum of a test file with `md5sum`
- [ ] Copy the file to a new location and confirm the checksum still matches
- [ ] Generate a `checksums.txt` for several files with `md5sum *.ext > checksums.txt`
- [ ] Run `md5sum -c checksums.txt` and confirm every file reports `OK`
- [ ] Modify one file slightly, re-run the check, and confirm it now reports `FAILED`
- [ ] Compress one of your test files, check its checksum, then decompress it and confirm the checksum is different — even though the content is "the same" data
