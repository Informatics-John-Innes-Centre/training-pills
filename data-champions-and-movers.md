# Data Champions and the JIC Archive Data-Mover Nodes

## Who can write to `/jic/archive`, and why it's locked down that way

**Time:** about 10 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Explain what a Data Champion is and what they can do that you can't
- Know which partition to use for downloading data onto HPC storage
- Understand why `/jic/archive` is read-only almost everywhere except one set of nodes
- Know who to talk to if you want your group to have a Data Champion
- Run an archiving transfer from a `jic-data-mover` node inside `tmux`, using `rsync`

The main idea to take away:

> **`/jic/archive` is deliberately read-only everywhere except the `jic-data-mover` nodes, and only Data Champions can use those to write to it. That's not a bug or an inconvenience — it's what stops an accidental `rm -rf` or an overwrite from reaching your group's long-term archive.**

---

## 1. What a Data Champion is

A Data Champion is a nominated, trusted person in a research group (or a designated JIC Informatics/Research Computing staff member) given the extra access needed to manage that group's data lifecycle properly — downloading data into HPC storage and archiving finished data — rather than every group member having that access by default.

It isn't a separate login or a formal application process: if your group doesn't have one and you think it should, raise it with JIC Informatics (informatics@jic.ac.uk, or the [Wednesday drop-in sessions](https://intranet.nbi.ac.uk/cms/6741) if your institution runs them) — see the [Informatics Websites](informatics-websites.md) pill for other ways to reach the team.

## 2. Downloading data onto HPC storage

The HPC storage is deliberately hard to reach *from* the outside: your own device is the gateway, not the HPC itself — direct external access from the HPC is limited to a short list of exceptions (online databases/repositories like ENA, SRA or GEO).

For downloading data (e.g. from a sequencing provider or an external collaborator) directly onto HPC storage rather than routing it through your own machine first, there's a dedicated **download partition**. JIC Informatics, Research Groups, Data Champions and other designated users can use it.

## 3. The `jic-data-mover` nodes: the only way to write to the archive

This is the part that trips people up: `/jic/archive` is **read-only to every group member across the HPC** — you can browse it and copy files out of it from any normal login node, but you can't write to it, move things into it, or delete anything from it there.

Writing to the archive only works from one set of nodes: **`jic-data-mover01`** through **`jic-data-mover06`**, and only Data Champions can access them. On those same nodes, the regular working areas (`/jic/scratch`, `/jic/research-groups`, etc.) are mounted **read-only** too — deliberately, so that a command run on a data-mover node can't accidentally delete or overwrite your live working data while you're moving something into the archive.

| Area | From a normal login node | From a `jic-data-mover` node |
| --- | --- | --- |
| `/jic/archive` | Read-only | Read-write (Data Champions only) |
| `/jic/scratch`, `/jic/research-groups`, etc. | Read-write | Read-only |

The read/write split is the only thing that changes on a `jic-data-mover` node — storage performance is the same as on scratch, so there's no speed penalty for archiving from there instead of from a login node.

`/jic/archive` also keeps snapshots, the same mechanism covered in the [Recovering Deleted Files from Snapshots](recovering-deleted-files-snapshots.md) pill — so even archived data that gets overwritten or removed by mistake has a recovery path.

## 4. A typical archiving workflow

Putting it together, archiving a finished run from scratch into `/jic/archive` as a Data Champion looks like this:

```text
your laptop → ssh → HPC login node → ssh → jic-data-mover0X → tmux + rsync
```

```bash
# 1. SSH into the HPC as normal
ssh hpc.nbi.ac.uk

# 2. From the login node, SSH into one of the data-mover nodes
ssh jic-data-mover01

# 3. Start a named tmux session so the transfer survives a dropped connection
tmux new -s archive

# 4. Run the transfer with rsync, trailing slash on both sides, -P for
#    progress and resumability
rsync -a -P "/jic/scratch/${GROUP}/${SAMPLE}/" "/jic/archive/${GROUP}/${SAMPLE}/"

# 5. Detach once it's running (Ctrl-b, then d) and check back later
tmux attach -t archive
```

This is the same `tmux`/`rsync` pairing covered in the [tmux Basics](tmux-basics.md) and [rsync Basics](rsync-basics.md) pills, just run from a `jic-data-mover` node instead of the login node so the write to `/jic/archive` is actually permitted. Everything those two pills teach — naming sessions, detaching/reattaching, the trailing-slash rule, resuming an interrupted transfer by re-running the exact same command — applies unchanged here.

## 5. Why it's worth it

This two-way read-only split means a single person's mistake — a wrong path in an `rm` or an `rsync` run from the wrong place — can't simultaneously destroy both your live working copy and your archived copy. Moving data into long-term storage is treated as its own deliberate step through a narrow, controlled gate, not something that happens as a side effect of everyday work on a login node.

## Your survival kit

| Question | Answer |
| --- | --- |
| I want to download data straight onto HPC storage | Use the download partition (JIC Informatics, Research Groups, Data Champions and designated users) |
| I want to write to `/jic/archive` | You need to be a Data Champion, working from a `jic-data-mover01`–`06` node |
| I'm a Data Champion — can I still edit my group's scratch data from a data-mover node? | No — scratch/research-groups areas are read-only there on purpose |
| I accidentally deleted or overwrote something in the archive | Check its snapshots — see the snapshot-recovery pill |
| My group doesn't have a Data Champion yet | Ask JIC Informatics (informatics@jic.ac.uk) |
| I'm archiving a large transfer as a Data Champion | `ssh` to a `jic-data-mover` node, start a `tmux` session, run `rsync -a -P SRC/ DEST/` inside it — see the tmux and rsync pills |

## Practice exercise

- [ ] Find out whether your research group already has a Data Champion
- [ ] Try browsing `/jic/archive` from a normal login node and confirm it's read-only for you
- [ ] If you are a Data Champion, confirm you can reach a `jic-data-mover` node and that the working areas are read-only there
- [ ] If you are a Data Champion, SSH into a `jic-data-mover` node, start a named `tmux` session, and run a test `rsync -a -P` transfer into `/jic/archive` inside it

---

Based on JIC Informatics' Data Management training course — see the [Data Management Guide](https://jic-datamanagement.informatics.jic.ac.uk/data-management) and the [full slide deck](https://jic-datamanagement.informatics.jic.ac.uk/data-management/presentation) for the complete session, including uploading to EBI/SRA/GEO and the recommended end-to-end data pipeline.
