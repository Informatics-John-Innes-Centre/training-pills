# Linux Basics for Bioinformatics

## Finding your way around a terminal, with nothing else assumed

**Time:** about 40–50 minutes
**Level:** Beginner — no prior command-line experience needed

By the end of this tutorial, you will be able to:

- Navigate the filesystem from the command line
- Tell the difference between absolute and relative paths
- Create, copy, move, and delete files and directories
- View the contents of a file safely
- Use wildcards to act on many files at once
- Read a file's permissions and understand `Permission denied`
- Get help without leaving the terminal

The main idea to take away:

> **A terminal is just a way to tell the computer what to do, one instruction at a time — a small vocabulary of commands covers almost everything you'll need on an HPC.**

---

## 1. Why the command line?

On an HPC, there usually isn't a graphical desktop — you connect over SSH straight into a terminal, and everything (moving files, running tools, submitting jobs) happens through typed commands. This isn't a stripped-down version of "real" computing — it's the normal way bioinformatics work happens, and every other pill in this series (`tmux`, `rsync`, `find`, `slurm`, ...) assumes you're comfortable here first.

## 2. Where am I? Navigating the filesystem

**Print your current location**

```bash
pwd
```

("print working directory")

**List what's in the current directory**

```bash
ls
```

**List with more detail** (sizes, permissions, dates)

```bash
ls -l
```

**Include hidden files** (anything starting with `.`, like `.bashrc`)

```bash
ls -a
```

**Move into a directory**

```bash
cd project
```

**Move up one level**

```bash
cd ..
```

**Jump straight to your home directory**

```bash
cd
```

or

```bash
cd ~
```

**Jump back to wherever you just were**

```bash
cd -
```

## 3. Absolute vs relative paths

A **relative** path is interpreted starting from wherever you currently are:

```bash
cd project/results
```

An **absolute** path always starts from the very top (`/`), and means the same thing regardless of where you currently are:

```bash
cd /nbi/scratch/username/project/results
```

`~` is shorthand for your home directory, and can be used inside a path too:

```bash
cd ~/project
```

> **Tip:** if a command isn't finding a file you're sure exists, run `pwd` first — it's very easy to assume you're in one directory when you're actually in another.

## 4. Creating and removing

**Create a directory**

```bash
mkdir results
```

**Create nested directories in one go**

```bash
mkdir -p project/results/sample01
```

`-p` creates any missing parent directories along the way, and doesn't complain if `results` already exists.

**Create an empty file**

```bash
touch notes.txt
```

**Remove a file**

```bash
rm notes.txt
```

**Remove a directory and everything inside it**

```bash
rm -r old_results/
```

> **There is no undo, and no recycle bin.** `rm` deletes immediately and permanently. Before running `rm -r` on anything, it's worth listing the same path with `ls` first to double-check exactly what you're about to remove — the same habit recommended for `find ... -delete` in the `find-basics` pill.

## 5. Copying and moving

**Copy a file**

```bash
cp sample.fastq.gz backup/
```

**Copy a directory and its contents**

```bash
cp -r project/ project_backup/
```

**Move a file** (also how you rename something)

```bash
mv draft.txt final.txt
mv sample.fastq.gz /nbi/scratch/username/raw_reads/
```

Unlike `cp`, `mv` doesn't leave a copy behind — the file exists in exactly one place, just with a new name or location.

## 6. Viewing file contents

**Print an entire file to the terminal**

```bash
cat samples.txt
```

Fine for short files; for anything long, it'll scroll past faster than you can read it.

**Page through a file** — the tool actually meant for this

```bash
less pipeline.log
```

Use the arrow keys or Page Up/Down to move, `/searchterm` to search, and **`q` to quit** — the single most important thing to know about `less`, since it isn't obvious and leaves a lot of people stuck looking at a file wondering how to get back to their prompt.

**Look at just the start or end of a file**

```bash
head samples.txt      # first 10 lines
tail samples.txt      # last 10 lines
tail -n 20 samples.txt # last 20 lines
```

**Watch a file as it grows** — invaluable for a running job's log

```bash
tail -f pipeline.log
```

`Ctrl+C` to stop watching (this doesn't stop the job itself, just the `tail`).

## 7. Wildcards — acting on many files at once

`*` matches any number of characters:

```bash
ls *.fastq.gz
rm *.tmp
cp sample01_R*.fastq.gz project/
```

`?` matches exactly one character:

```bash
ls sample0?.bam    # matches sample01.bam, sample02.bam, ... but not sample010.bam
```

Brace expansion generates several names from a pattern:

```bash
mkdir -p project/{raw,trimmed,aligned}
# creates project/raw, project/trimmed, project/aligned in one command
```

> **Safety habit:** before running `rm` with a wildcard, run the exact same pattern with `ls` first, to see precisely what would be affected.

## 8. Permissions — reading `ls -l` and understanding "Permission denied"

```bash
ls -l samples.txt
```

```text
-rw-r--r-- 1 username groupname 214 Sep 10 09:15 samples.txt
```

The first 10 characters describe the type and permissions:

```text
-  rw-  r--  r--
│   │    │    │
│   │    │    └─ others: read only
│   │    └────── group: read only
│   └─────────── owner: read, write
└─────────────── file type (- = file, d = directory)
```

`r` = read, `w` = write, `x` = execute. If you ever see `Permission denied`, `ls -l` on the file (or its containing directory) is the first thing to check — it tells you exactly who is and isn't allowed to do what.

**Make a script executable** (the one permission change you'll do constantly — see the `nano-basics` and `shell-scripting-basics` pills)

```bash
chmod +x my_script.sh
```

## 9. Getting help without leaving the terminal

**The manual page for a command**

```bash
man rsync
```

`q` to quit, same as `less` (it's the same pager underneath).

**A shorter summary, for most commands**

```bash
rsync --help
```

> **Also worth remembering:** if you're stuck on the exact syntax for something, an AI assistant is a perfectly good place to ask — the same tip mentioned in the `grep-sed-awk-basics` pill for tricky `awk`/`sed` syntax applies just as well here.

## 10. Your Linux survival kit

| Command | What it does |
| --- | --- |
| `pwd` | Show your current directory |
| `ls` / `ls -l` / `ls -a` | List files (detailed / including hidden) |
| `cd path` / `cd ..` / `cd ~` | Change directory |
| `mkdir -p path` | Create a directory (and parents) |
| `touch file` | Create an empty file |
| `rm file` / `rm -r dir/` | Delete a file / a directory and its contents |
| `cp file dest` / `cp -r dir dest` | Copy a file / a directory |
| `mv old new` | Move or rename |
| `cat file` | Print a whole file |
| `less file` | Page through a file (`q` to quit) |
| `head` / `tail` / `tail -f` | View the start / end / live end of a file |
| `*`, `?`, `{a,b}` | Wildcards for matching multiple names |
| `ls -l` | Show a file's permissions and ownership |
| `chmod +x file` | Make a file executable |
| `man command` | Read a command's manual |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Check your current directory with `pwd`
- [ ] Create a nested directory structure in one command with `mkdir -p`
- [ ] Create a few empty test files with `touch`
- [ ] Copy one, move (rename) another, and delete a third
- [ ] Use a wildcard to list, then copy, a subset of your test files
- [ ] View a file with `cat`, then the same file with `less`, and quit `less` with `q`
- [ ] Use `tail -f` on a file, then add a line to it from another terminal/session and watch it appear
- [ ] Run `ls -l` on one of your files and identify the owner, group, and permission bits
- [ ] Make a test script executable with `chmod +x` and run it
- [ ] Look up a command you already know with `man`, and quit with `q`
