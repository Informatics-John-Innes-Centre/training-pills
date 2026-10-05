# Open OnDemand Basics

## Using the HPC from your browser, no SSH client required

**Time:** about 15–20 minutes
**Level:** Beginner — just need an active HPC account

By the end of this pill, you will be able to:

- Explain what Open OnDemand is and when it's useful
- Log in and find your way around the landing page
- Browse and manage your files from the browser
- Get a shell on the cluster without an SSH client
- Submit and monitor jobs through the Job Composer
- Launch a graphical app (desktop, Jupyter, RStudio, etc.) as an interactive session

The main idea to take away:

> **Open OnDemand is a web portal onto the same HPC cluster you'd reach over SSH — file browser, shell, job management, and graphical apps, all through a browser tab, no plugins or client software to install.**

---

## 1. What is Open OnDemand?

Normally, using the HPC means an SSH client, a terminal, and (for anything
graphical) fiddling with X11 forwarding or a VNC setup. Open OnDemand gives
you the same cluster through a web browser instead:

- File management
- Command-line shell access
- Job management and monitoring
- Graphical desktop environments and desktop applications (Jupyter, RStudio,
  a full desktop, and more)

Nothing about the underlying cluster changes — it's the same filesystem, the
same Slurm queue, the same partitions. Open OnDemand is just a different
front door, useful when you're on a machine without an SSH client, want a
GUI app without setting up X11/VNC yourself, or just prefer working in a
browser tab.

## 2. Accessing Open OnDemand

You need a registered, active HPC account — same credentials as everywhere
else on the cluster. Point a browser at:

```text
https://ood.hpc.nbi.ac.uk
```

A few practical recommendations:

- **Use a private/incognito window.** It clears cookies and session
  information as soon as you close it, which matters more here than on a
  typical website since this session has access to your HPC account.
- **Use Chrome if you can.** It natively supports copy/paste into the
  graphical sessions. Firefox and Edge work too, but copy/paste only works
  through the noVNC tool drawer (the tab on the left once you're inside a
  desktop session), not directly.

On first connection you'll be prompted to log in with your standard
username and password. After that you land on the Open OnDemand dashboard.

## 3. File browsing

**Files → Home Directory** opens a browser-based file manager: browse, edit,
upload, download, and manage files without `scp` or `rsync` for quick,
one-off tasks. (For moving a lot of data, or automating transfers, the
`rsync-basics` pill is still the right tool.)

## 4. Shell access

**Cluster → Shell Access** opens a terminal straight into the cluster's
submission nodes — the same place an SSH client would put you — entirely in
the browser tab. Close the tab when you're done; there's nothing extra to
disconnect.

## 5. Managing and composing jobs

Under the **Jobs** menu:

- **Active Jobs** — view and manage jobs you've already submitted, a
  browser-based alternative to `squeue`.
- **Job Composer** — build and submit a job through a form-based interface
  instead of writing an `sbatch` script by hand. Useful for getting started;
  once you're comfortable with Slurm directly, see the `slurm-basics` pill
  for the script-based workflow, which gives you more control and is easier
  to automate/reuse.

## 6. Interactive apps

This is where Open OnDemand earns its keep for graphical work: launch an app
as an interactive session on a compute node, and it streams to your browser
— no VNC/X11 setup on your end. Available apps include:

- Generic Desktop
- RELION
- CryoSPARC
- Jupyter Notebook
- RStudio
- PyCharm Community Edition
- RShiny

Each launches on an actual compute node (not the login node), so you get
real CPU/memory resources for the session — you'll typically be asked to
specify how much before it starts.

## 7. Your Open OnDemand survival kit

| Menu | What it's for |
| --- | --- |
| Files → Home Directory | Browse/manage files in the browser |
| Cluster → Shell Access | A terminal on the cluster, no SSH client needed |
| Jobs → Active Jobs | Check on jobs you've submitted |
| Jobs → Job Composer | Build and submit a job via a form |
| Interactive Apps | Launch a desktop, Jupyter, RStudio, etc. on a compute node |

## Practice exercise

- [ ] Open `https://ood.hpc.nbi.ac.uk` in a private/incognito window and log in
- [ ] Browse your home directory under Files
- [ ] Open a shell session under Cluster → Shell Access and run a command
- [ ] Look at Jobs → Active Jobs (even with nothing running, confirm the page loads)
- [ ] Launch one Interactive App (Jupyter Notebook is a good first try) and confirm it connects
