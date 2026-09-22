# rsync Basics for Bioinformatics

## Moving and archiving large datasets safely

**Time:** about 30–40 minutes
**Level:** Beginner — no prior rsync experience needed

By the end of this tutorial, you will be able to:

- Explain why `rsync` is preferred over `cp`/`scp` for large datasets
- Use the trailing slash on source/destination correctly
- Use archive mode (`-a`) and know what it actually preserves
- Resume an interrupted transfer instead of starting again
- Decide when (not) to compress a transfer
- Understand the difference between symbolic and hard links, and why hard links are riskier
- Keep a transfer alive on the data mover node

The main idea to take away:

> **rsync only transfers what has actually changed, can resume where it left off, and — used correctly — preserves the metadata that makes your data trustworthy.**

---

## 1. Why rsync instead of `cp` or `scp`?

Sequencing datasets are big, transfers between scratch, archive, and other servers are routine, and connections occasionally drop. `cp` and `scp` restart from zero every time; `rsync` doesn't have to.

```text
cp / scp:  copies everything, every time, from scratch
rsync:     compares source and destination first,
           then only sends the differences
```

For bioinformatics specifically, this matters because:

- Archiving a run from scratch to archive storage can involve hundreds of GB or thousands of files
- Re-running the same `rsync` command after a dropped connection or a mistake costs you almost nothing — it skips whatever already transferred correctly
- You can re-sync a results directory after adding a few new samples without re-copying everything

## 2. Basic syntax — and the trailing-slash gotcha

```bash
rsync -a SOURCE/ DESTINATION/
```

The trailing slash on the **source** genuinely changes the behaviour, and it catches almost everyone at least once:

```text
rsync -a myrun  archive/      →  archive/myrun/...        (the folder itself is copied in)
rsync -a myrun/ archive/      →  archive/...               (the CONTENTS of myrun go straight into archive/)
```

If you forget the trailing slash and run the same command a second time, you get a folder nested inside itself:

```text
archive/myrun/myrun/...
```

> **Rule of thumb:** if you want the destination to end up looking like the source (same folder name, contents matched), put a trailing slash on **both** the source and destination.

A typical bioinformatics archiving command looks like this:

```bash
rsync -a "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
```

This deliberately keeps the source and destination directory names in sync, with a trailing slash on both, so re-running it is safe and idempotent.

## 3. Archive mode (`-a`) — and what `-r` actually does

`-a` (archive mode) is shorthand for a bundle of options, roughly `-rlptgoD`:

| Included in `-a` | Meaning |
| --- | --- |
| `-r` | recurse into subdirectories |
| `-l` | copy symbolic links as symbolic links |
| `-p` | preserve permissions |
| `-t` | preserve timestamps |
| `-g`, `-o` | preserve group and owner (if you have permission to) |
| `-D` | preserve device and special files |

This is why `-r` on its own "does nothing new" if you're already using `-a` — it's already included. But `-r`/recursion is **not** on by default in plain rsync: without `-a` or `-r`, rsync will not descend into subdirectories at all. If you ever saw a deep directory structure sync correctly without explicitly typing `-r`, it's because `-a` was already doing that job for you.

> **Practical takeaway:** always use `-a` for bioinformatics data. It gives you recursion plus the metadata preservation described below, in one flag.

## 4. Preserving timestamps and links — and the hard-link trap

Unlike a plain copy, `rsync -a` preserves file **timestamps**, which matters for provenance: you can still tell when a file was actually produced by the pipeline, not just when it was last copied.

`-a` also preserves **symbolic (soft) links** as links, rather than copying the data they point to. This is usually what you want — for example, linking to a shared reference genome instead of duplicating it in every project folder. If the link's target ever moves or is deleted, the link simply breaks (dangling) — that's an easy, visible failure.

**Hard links are riskier**, and `rsync -a` does *not* preserve them as hard links by default (you'd need the separate `-H`/`--hard-links` option). It's worth understanding why hard links deserve caution if you come across them:

- A hard link isn't a pointer to a file — it's another name for the *same* underlying data.
- Two hard-linked "copies" look like independent files, but editing, truncating, or corrupting one changes the other too, because they're the same data.
- The underlying data is only actually freed once *every* hard-linked name has been deleted — but nothing warns you that a name you're deleting is the last one.

> **The danger:** if you assume a hard-linked file is a safe independent backup, you may be wrong on both counts — modifying it can silently affect what you thought was untouched, and deleting "duplicates" one by one can quietly destroy your only copy on the last delete. Prefer symbolic links (or genuine independent copies) when the goal is a safety copy.

## 5. Resuming an interrupted transfer

This is the single biggest reason to reach for `rsync` over a plain copy: it can pick up where it left off.

```bash
rsync -a -P "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
```

`-P` is shorthand for `--partial --progress`:

- `--partial` keeps partially-transferred files instead of deleting them if the transfer is interrupted, so the next run can continue from there instead of re-sending the whole file
- `--progress` shows a running progress indicator per file, so you're not staring at a silent terminal during a multi-hour transfer

If your connection drops, or you cancel with `Ctrl-C`, just run the **exact same command again**. rsync compares source and destination and only sends what's missing or different.

> **Tip:** for a very large number of files (common in bioinformatics — think thousands of FASTQ or per-read files), plain rsync output can be sparse or overwhelming. `--info=progress2` gives you one overall progress line for the whole transfer instead of one line per file.

## 6. Compression — usually not needed on a fast network

Some guides suggest compressing data in transit to save bandwidth. In rsync this is the `-z`/`--compress` option, and — importantly — **it is not on by default**. If you've never typed `-z`, you were never compressing.

On a fast internal network (like a local HPC/JIC network), compression is usually **not worth it**:

- Your bottleneck is unlikely to be network bandwidth
- Compression costs CPU time on both ends, which can actually slow down a transfer on a fast link
- A lot of bioinformatics data is already compressed (BAM, CRAM, `.fastq.gz`) — trying to compress it again wastes CPU for essentially no size reduction

```bash
rsync -a -P "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
```

is normally the right call — no `-z` needed. Since `-z` already defaults to off, you don't need `--no-compress` either; it's a no-op that just documents intent. (It works because rsync lets you negate almost any option with `--no-OPTION`, which is occasionally useful for undoing something implied by another flag — but it isn't needed here.)

> **When compression *does* make sense:** slow or high-latency links (e.g. transferring uncompressed text files over a slow home internet connection), not a fast local network moving already-compressed genomic data.

## 7. Checking a transfer actually finished

Because rsync only reports what it *did* transfer, the reassuring way to confirm a big archiving job is complete is to run it again:

```bash
rsync -a -P "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
```

If everything already matches, the second run finishes almost instantly and reports (close to) nothing to do. That's your confirmation the archive is complete and correct — a useful habit for anything you're about to delete from scratch.

You can also preview what *would* happen without changing anything, using a dry run:

```bash
rsync -a -P --dry-run "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
```

## 8. Keeping the transfer alive on the data mover node

Large archiving transfers are often run from a dedicated **data mover** node rather than the login node. Some data movers don't have `tmux` installed — if that's the case for yours, use `screen` instead. The commands map directly onto what you already know from tmux:

| tmux | screen | What it does |
| --- | --- | --- |
| `tmux new -s <name>` | `screen -S <name>` | Start a new named session |
| `Ctrl-b d` | `Ctrl-a d` | Detach from the session |
| `tmux ls` | `screen -ls` | List sessions |
| `tmux attach -t <name>` | `screen -r <name>` | Reattach to a session |

```bash
screen -S archive
rsync -a -P "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
# Ctrl-a, then d to detach — the transfer keeps running
```

## 9. Your rsync survival kit

| Command | What it does |
| --- | --- |
| `rsync -a SRC/ DEST/` | Copy, preserving timestamps/permissions/symlinks, recursing into subdirectories |
| `rsync -a -P SRC/ DEST/` | Same, with progress and resumability if interrupted |
| `rsync -a -P --dry-run SRC/ DEST/` | Preview what would be transferred, without changing anything |
| Trailing slash on `SRC/` | Copy the *contents* of `SRC`, not the folder itself |
| `--info=progress2` | One overall progress line instead of one per file |
| `-z` / `--compress` | Compress in transit — usually skip this on a fast local network |

**The command to remember:**

```bash
rsync -a -P "${SCRATCHDIR}/${SAMPLE}/" "${ARCHIVEDIR}/${SAMPLE}/"
```

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Create a small test directory with a few files and a subdirectory
- [ ] Copy it with `rsync -a SRC/ DEST/` and confirm the contents match
- [ ] Repeat the copy *without* the trailing slash on the source and see the nested-folder result for yourself
- [ ] Add a new file to the source and re-run `rsync -a` — confirm only the new file is transferred
- [ ] Start a transfer of a larger test directory with `rsync -a -P`, cancel it with `Ctrl-C` partway through, then re-run the same command and confirm it resumes rather than starting over
- [ ] Run the same completed transfer a second time and confirm it reports (almost) nothing left to do
- [ ] Create a symbolic link to a file and confirm `rsync -a` preserves it as a link rather than copying the data
- [ ] If you use a data mover node, start a `screen` session, launch a transfer, detach, log out, log back in, and reattach
