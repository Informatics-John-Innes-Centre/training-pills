# WSL Basics for Bioinformatics

## Getting a real Linux environment on your Windows laptop

**Time:** about 30–40 minutes
**Level:** Beginner — no prior Linux experience needed

By the end of this tutorial, you will be able to:

- Explain why WSL is useful for bioinformatics work on a Windows machine
- Install WSL (and know when you'll need Computing's help to do it)
- Complete first-time setup and update your Linux environment
- Find your way between the Windows and Linux file systems
- Mount extra drives and network shares into `/mnt/` and access them
- Use WSL to `ssh` into HPC and other remote servers
- Avoid the most common WSL performance mistake

The main idea to take away:

> **WSL gives you an actual Linux system running alongside Windows — the same shell, the same tools, the same commands you'll use on the HPC — without needing a second computer.**

---

## 1. Why WSL?

Almost all bioinformatics tools (`samtools`, `bwa`, `conda`, pipeline frameworks, `ssh` clients used the "normal" way) are built for Linux first. Windows can approximate some of this, but WSL (Windows Subsystem for Linux) gives you a genuine Linux environment running directly alongside Windows — not a virtual machine you have to babysit separately.

This means:

- You can install and run the same command-line bioinformatics tools locally that you use on the HPC
- You get a proper `ssh` client for connecting to remote servers, with no separate program needed
- You can write and test small scripts locally before scaling them up on the cluster
- Everything you learn in the other pills in this series (`tmux`, `find`, `rsync`, ...) works identically inside WSL

## 2. Before you start: admin rights

**Installing WSL requires administrator rights on your Windows machine.** It enables a Windows feature (virtualisation-based Linux support), which isn't something a standard user account is allowed to do.

- If you're a local administrator on your laptop, you can install it yourself (Section 3).
- If you're not (common on managed JIC laptops), you'll need Computing to either install it for you or grant temporary admin rights so you can run the install command yourself.

> **Don't skip this step and try to work around it** — there isn't a reliable non-admin way to install WSL, so raise a ticket with Computing early if you think you'll need one.

## 3. Installing WSL

**Step 1 — Open PowerShell as Administrator**

Right-click the Start button (or search "PowerShell"), and choose **Run as administrator**.

**Step 2 — Install WSL**

```powershell
wsl --install
```

This enables the required Windows features, installs WSL2, and installs Ubuntu as the default Linux distribution — a sensible default, since most bioinformatics documentation assumes a Debian/Ubuntu-style system.

**Step 3 — Restart your computer** when prompted.

## 4. First-time setup

After restarting, Ubuntu will launch automatically (or launch it from the Start menu) and finish setting itself up. You'll be asked to create a **Linux username and password** — this is separate from your Windows login, and can be anything you like.

Once you're in, update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

`sudo` runs a command with administrator privileges *inside* Linux — it will ask for the Linux password you just created, not your Windows one.

## 5. Finding your way around

**From Windows, browse your Linux files:**

```text
\\wsl$\Ubuntu\home\<your-linux-username>\
```

Type this into the Windows Explorer address bar to see your Linux home directory as a normal folder.

**From Linux, access your Windows files:**

```bash
cd /mnt/c/Users/<your-windows-username>/
```

Your Windows `C:` drive is mounted inside Linux under `/mnt/c/`.

> **Performance tip:** keep your actual working files (scripts, test data, cloned repositories) inside the Linux file system (`~/`), not under `/mnt/c/`. Reading and writing files across the Windows/Linux boundary is noticeably slower than staying on one side — this matters once you're processing real sequencing data, even in small test form.

## 6. Mounting other drives and network shares into `/mnt/`

`C:` isn't the only thing that can show up under `/mnt/`. Any Windows drive letter — a second local disk, or a mapped network drive for shared/group storage — follows the same pattern.

**Drive letters that were already mapped before you started WSL** usually appear automatically:

```bash
ls /mnt/
# c  d  z
```

If a drive letter is missing (often because it was mapped in Windows *after* WSL was already running), restart the WSL engine so it re-detects your drives:

```powershell
wsl --shutdown
```

Then reopen your Linux shell.

**Mounting a drive letter manually**, if it still doesn't appear:

```bash
sudo mkdir -p /mnt/z
sudo mount -t drvfs Z: /mnt/z
```

**Mounting a network share directly by its UNC path** — useful for shared/group storage that isn't mapped to a drive letter at all. For example, your HPC home directory is exposed as a network share at `\\jic-hpc-data\HPC-Home`:

```bash
sudo mkdir -p /mnt/hpchome
sudo mount -t drvfs '\\jic-hpc-data\HPC-Home' /mnt/hpchome
```

Use single quotes around the UNC path so bash doesn't try to interpret the backslashes itself.

**Accessing a mounted share** works exactly like any other directory:

```bash
cd /mnt/hpchome
ls
```

**Unmounting** when you're done with it:

```bash
sudo umount /mnt/hpchome
```

> **Note:** manual mounts (via `mount -t drvfs`) don't persist across a `wsl --shutdown` or a reboot — you'll need to re-run the mount command afterwards, unless you add them to `/etc/fstab` (below).

**Making a mount permanent with `/etc/fstab`**

Add an entry so the mount is recreated automatically every time WSL starts, the same way regular Linux mounts work:

```bash
sudo nano /etc/fstab
```

```text
Z:                      /mnt/z          drvfs   defaults 0 0
\\jic-hpc-data\HPC-Home /mnt/hpchome    drvfs   defaults 0 0
```

Each line is `<source> <mount point> <filesystem type> <options> <dump> <pass>` — the mount point directory must already exist (`sudo mkdir -p /mnt/hpchome` first, as above).

Test it without rebooting:

```bash
sudo mount -a
```

This mounts everything listed in `/etc/fstab` that isn't already mounted, so you can confirm the entry works before trusting it to happen automatically. From now on, both mounts will also reappear after `wsl --shutdown` or a full reboot, without you running `mount` by hand.

## 7. Using WSL day to day

Once installed, you can open a Linux shell in a few ways:

```powershell
wsl
```

from any PowerShell/Command Prompt window, or simply open **Ubuntu** from the Start menu.

**Install bioinformatics tools inside WSL** just as you would on any Linux machine — for example, `miniconda`/`mambaforge` and `conda`/`mamba` environments work exactly the same as they do on the HPC.

**Connect to the HPC** using WSL's built-in `ssh`, no separate SSH client required:

```bash
ssh your-username@hpc.nbi.ac.uk
```

> **Note:** `hpc.nbi.ac.uk` and `slurm.nbi.ac.uk` are both aliases for the same cluster — you may see either used interchangeably in documentation or by colleagues.

## 8. Managing WSL itself (from Windows)

These commands run in PowerShell (outside WSL), and manage WSL as a whole:

```powershell
wsl --list --verbose     # see installed distributions and their WSL version
wsl --shutdown           # fully restart the WSL engine (useful if it hangs)
wsl --update             # update the WSL platform itself
```

## 9. Your WSL survival kit

| Command | What it does |
| --- | --- |
| `wsl --install` | Install WSL and the default Linux distribution (needs admin rights) |
| `wsl` | Open a Linux shell from PowerShell/Command Prompt |
| `sudo apt update && sudo apt upgrade -y` | Update installed Linux packages |
| `\\wsl$\Ubuntu\home\<user>\` | Browse your Linux files from Windows Explorer |
| `/mnt/c/Users/<user>/` | Access your Windows files from Linux |
| `sudo mount -t drvfs Z: /mnt/z` | Manually mount a Windows drive letter |
| `sudo mount -t drvfs '\\server\share' /mnt/name` | Mount a network share by UNC path |
| `sudo umount /mnt/name` | Unmount a share |
| Entry in `/etc/fstab` + `sudo mount -a` | Make a mount persist across WSL restarts |
| `wsl --list --verbose` | List installed distributions |
| `wsl --shutdown` | Restart the WSL engine |

**The rule to remember:**

```text
Keep your work here:      ~/  (Linux file system)      — fast
Not here:                 /mnt/c/...  (Windows file system) — slow from Linux
```

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Confirm whether you have local admin rights; if not, raise a ticket with Computing
- [ ] Run `wsl --install` from an administrator PowerShell window
- [ ] Restart your computer and complete first-time Ubuntu setup (Linux username/password)
- [ ] Run `sudo apt update && sudo apt upgrade -y`
- [ ] Open your Linux home directory from Windows Explorer via `\\wsl$\...`
- [ ] From inside WSL, browse to your Windows `C:` drive via `/mnt/c/`
- [ ] Create a test file in `~/` and another in `/mnt/c/...`, and notice which "feels" like your normal Linux home
- [ ] Check `/mnt/` for any other drive letters already mapped in Windows
- [ ] Manually mount a network share by UNC path with `sudo mount -t drvfs '\\server\share' /mnt/name`, then `cd` into it and list its contents
- [ ] Unmount it with `sudo umount`, then run `wsl --shutdown` and reopen WSL to see which drives reappear automatically
- [ ] Add that share to `/etc/fstab`, test it with `sudo mount -a`, then confirm with `wsl --shutdown` that it now mounts automatically
- [ ] Mount your HPC home directory from `\\jic-hpc-data\HPC-Home` and confirm you can see your files
- [ ] `ssh` into a remote server (e.g. the HPC) directly from your WSL terminal
