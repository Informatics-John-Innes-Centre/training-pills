# Linux Playground

## A shared Linux machine for practising and for training labs

**Time:** about 15–20 minutes
**Level:** Beginner — no prior experience needed; pairs well with the [Linux](linux-basics.md) pill

By the end of this tutorial, you will be able to:

- Explain what linux-playground is for, and what it isn't for
- Log in from Windows, Mac or Linux
- Link your U drive, so files you make are visible in Windows
- Know where to keep files, and how much space there is
- Get data onto the machine and results off it
- Find the bioinformatics tools that are already installed
- Follow a trainer's screen during a class

The main idea to take away:

> **linux-playground is a safe place to learn: a real Linux machine with the common tools already installed, where nothing you do can break anything that matters. For real analyses, use the HPC.**

---

## 1. What it is

linux-playground (`linux-playground.nbi.ac.uk`) is a small Linux server run by JIC Informatics for training. Many pills in this series, and most of our hands-on courses, use it for their exercises.

| | |
| --- | --- |
| System | AlmaLinux 9 |
| CPUs / memory | 8 cores, 15 GB, **shared by everyone logged in** |
| Who can log in | Everyone at the NBI, with your usual username and password |
| Internet | Yes, so `wget` and `curl` work |

**Use it for:** learning commands, working through pills and course labs, testing a small command before you run it on the HPC.

**Don't use it for:** real analyses, large data, or anything confidential. It is small and shared, other people (including trainers) can see what runs on it, and it isn't backed up. For real work, use the HPC (see the [Slurm](slurm-basics.md) and [Open OnDemand](open-ondemand-basics.md) pills).

## 2. Logging in

You need your **NBI username and password**, the same ones you use to log in to your PC. You connect from the institute network; off site, connect to the VPN first.

**Windows: PuTTY.** Open PuTTY, type `linux-playground.nbi.ac.uk` in **Host Name**, leave the port at 22, and click **Open**. Log in with your username, then your password.

**Windows, Mac, Linux or [WSL](wsl-basics.md): a terminal.** Windows 10 and 11 include `ssh` in PowerShell and Windows Terminal; on a Mac use the Terminal app.

```bash
ssh <your-username>@linux-playground.nbi.ac.uk
```

Two things surprise people the first time:

- **A question about the host's "fingerprint"** appears on your first connection only. It's your computer asking whether it can trust this server. Type `yes` (or click **Accept** in PuTTY).
- **Nothing appears while you type your password.** No dots, no stars. That's normal: type it and press Enter.

You'll then see a welcome banner and a prompt like this:

```text
[username@v1373 ~]$
```

`v1373` is the machine's real name. You're in your home folder (`~`) and ready to type commands. To log out, type `exit` or press `Ctrl-d`.

## 3. Link your U drive

Your files on linux-playground are only on linux-playground. To see them in Windows as well, for example to open a result in Excel or IGV, link a folder of your U drive with `mountu`.

1. **Once only, in Windows:** open File Explorer, go to your U drive (`U:`), and create a folder named exactly `informatics`.
2. **On linux-playground**, after each restart of the machine:

   ```bash
   mountu
   ```

   It asks for your password (again, nothing appears as you type).

3. Check it worked: create a text file in `U:\informatics` from Windows, then:

   ```bash
   ls ~/u_drive
   ```

Anything you save in `~/u_drive` is now in `U:\informatics`, and the other way round. If `mountu` says *"The network share is already mounted"*, it's already done.

> **Work in `~/u_drive` during labs.** Your results are then saved on your U drive, and you can open them from Windows straight away, with no copying.

## 4. Where to keep files

| Place | Space | Notes |
| --- | --- | --- |
| `~/u_drive` | Your U drive | **Best place for your work.** Visible from Windows |
| `~` (home) | About 5 GB free, **shared by all users** | Fine for small exercises. Don't fill it up |
| `/tmp` | Under 1 GB, shared | Files are deleted after 10 days |

Check how much space a folder uses with `du -sh <folder>`, and how full a disk is with `df -h ~`.

## 5. Getting data in and out

**From the internet**, including our [Training Repo](https://repo.informatics.jic.ac.uk/), where course data lives:

```bash
wget https://repo.informatics.jic.ac.uk/bioinformatics26/data.zip
unzip data.zip
```

**From or to your PC:** save it in `U:\informatics` and it appears in `~/u_drive` (Section 3). For other folders, copy files with `scp` from your PC's terminal, or use WinSCP on Windows (host `linux-playground.nbi.ac.uk`, the same username and password):

```bash
scp results.txt <your-username>@linux-playground.nbi.ac.uk:~/
```

## 6. Software already installed

Everything below is ready to use. Just type its name; there's nothing to load.

| Tool | Version | Used for |
| --- | --- | --- |
| `bwa` | 0.7.18 | Mapping DNA reads to a reference |
| `bowtie2` | 2.5.4 | Mapping DNA reads |
| `hisat2` | 2.2.1 | Mapping RNA-seq reads |
| `samtools` | 1.18 | SAM/BAM files |
| `bcftools` | 1.21 | VCF files and variant calling |
| `fastqc` | 0.12.1 | Read quality reports |
| `salmon` | 1.10.0 | Transcript quantification |
| `Trinity`, `velveth`/`velvetg` | | Transcriptome and genome assembly |
| `quicktree` | | Phylogenetic trees |
| `python3` | 3.12 | Python scripting |
| `conda` | 25.1 | Installing more software (see the [conda](conda-basics.md) pill) |
| `git`, `nano`, `vim`, `tmux` | | Version control, editing, persistent sessions |

Check where a tool is and which version you're running:

```bash
command -v samtools
samtools --version
```

Not installed: R, Singularity/Apptainer and `screen`. For those, use the HPC.

## 7. Following a class

In a course, the trainer shares their screen through linux-playground. Open **two** PuTTY windows or terminals, and log in on both:

- **Window 1, to watch:** type `atraining`. You'll see the trainer's terminal live, with their last few commands at the top. You can't type in it. To leave, press `Ctrl-b`, then `d`.
- **Window 2, to work:** type the commands yourself here.

If `atraining` says `error connecting to /tmp/training`, the trainer hasn't started the session yet. Try again in a minute.

> **For trainers** (members of the `informatics` group): `ctraining` starts the shared screen, or joins it if another trainer has already started it; `kill-training` stops it at the end. Anything typed in it runs as the trainer who started it, so don't use `sudo` or open anything private there. Setup details: [linux-playground repo](https://git.nbi.ac.uk/Miguel-projects/linux-playground).

## 8. Being a good neighbour

- **Keep it small.** Everyone shares 8 CPUs and 15 GB of memory. Exercises are sized for this; real data isn't.
- **Don't leave things running.** Check with `top` (press `q` to quit), and stop your own runaway command with `Ctrl-c`.
- **Clean up** large files you no longer need, especially in your home folder and `/tmp`.
- **Nothing confidential.** It's a shared training machine.

## 9. When something goes wrong

| What you see | What it means |
| --- | --- |
| `Permission denied, please try again.` at login | Wrong username or password. Use your NBI username, not your email address |
| `Connection timed out` | You're off site without the VPN, or the address is mistyped |
| `mount error(2): No such file or directory` from `mountu` | There's no `informatics` folder in your U drive yet (Section 3, step 1) |
| `mount error(13): Permission denied` from `mountu` | Wrong password; run `mountu` again |
| `ls ~/u_drive` is empty but `U:\informatics` isn't | The link was lost when the machine restarted; run `mountu` again |
| `command not found` | Check the spelling. If the tool isn't in Section 6, it isn't installed |
| `No space left on device` | Your home folder or `/tmp` is full; delete what you don't need, or work in `~/u_drive` |

## 10. Your linux-playground survival kit

| Command | What it does |
| --- | --- |
| `ssh <user>@linux-playground.nbi.ac.uk` | Log in |
| `exit` | Log out |
| `mountu` | Link `U:\informatics` to `~/u_drive` |
| `cd ~/u_drive` | Go to your U drive folder |
| `wget <url>` | Download a file |
| `scp <file> <user>@linux-playground.nbi.ac.uk:~/` | Copy a file from your PC |
| `du -sh <folder>` / `df -h ~` | How much space a folder uses / how full the disk is |
| `command -v <tool>` | Is this tool installed, and where |
| `atraining` | Watch the trainer's screen (`Ctrl-b` then `d` to leave) |
| `top` | See what's running (`q` to quit) |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Log in to linux-playground, accept the fingerprint, and find `v1373` in your prompt
- [ ] Create `U:\informatics` in Windows, run `mountu`, and see a file you made in Windows with `ls ~/u_drive`
- [ ] Create a file in `~/u_drive` from linux-playground, and open it in Windows
- [ ] Check how much free space there is with `df -h ~`
- [ ] Download a file from the [Training Repo](https://repo.informatics.jic.ac.uk/) with `wget`
- [ ] Find `bwa` and `samtools` with `command -v`, and print their versions
- [ ] Log in from a second window, and log out of both with `exit`
- [ ] Now try the [Linux](linux-basics.md) pill's exercises here
