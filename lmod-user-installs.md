# Lmod User Installs

## Sharing your software with everyone on the HPC as a module

**Time:** about 30–40 minutes
**Level:** Intermediate — you should be comfortable with the [HPC Software](hpc-software-basics.md) and [Singularity](singularity-basics.md) pills

By the end of this tutorial, you will be able to:

- Explain what the user-installs area is and how it differs from the Software Catalogue
- Lay out a container so the user-installs checks accept it
- Write the command wrappers and the modulefile
- Load and test your module, and use it in a Slurm job
- Name versions so they don't clash with anyone else's
- Fix the usual reasons a module doesn't show up or doesn't run

The main idea to take away:

> **A user install is a container plus a few small text files in an agreed layout. Get the layout right and anyone at JIC can `ml` your software, instead of you sending them a `.sif` file and instructions.**

---

## 1. What the user-installs area is

There are two places where shared software lives on the HPC:

| | Software Catalogue / `ml` modules | User installs |
| --- | --- | --- |
| Who adds software | Research Computing only | Any JIC user |
| Where | `/nbi/software/...` | `/jic/software/user-installs/` |
| Endorsed by RC | Yes | No: whoever installed it maintains it |
| How people load it | `ml <tool>/<version>` | `ml <tool>/<version>` |

Use the user-installs area when a tool isn't in the catalogue and other people need it too: a lab's pipeline, an old version you had to build yourself, a tool from a vendor installer. If it's a common tool, it's better to ask RC to add it to the catalogue (see the [HPC Software](hpc-software-basics.md) pill).

Each institute has its own area (`/jic/`, `/ei/`, `/tsl/`, `/qib/`). All HPC users can read and run everything in them, and JIC users can write to the JIC one.

> **The area is checked automatically.** Informatics runs a compliance check (`check_ui_dirs.sh`) over every install. It looks for the layout in Section 2 and for a modulefile that points back at the install, so stick to that layout.

## 2. The layout

Every version of every tool gets its own folder under `software/`, and a modulefile with the same name under `modules/`:

```text
/jic/software/user-installs/
├── software/
│   └── seqkit/
│       └── 2.8.2/
│           ├── image.img          the Singularity image (a .sif, renamed)
│           └── bin/
│               └── seqkit         wrapper that runs seqkit from image.img
└── modules/
    └── seqkit/
        └── 2.8.2.lua              the modulefile
```

- **`image.img`** is a normal Singularity image file; only the name is fixed.
- **`bin/`** holds one small wrapper script per command the tool provides.
- **The modulefile** adds that `bin/` folder to your `PATH` when you load the module.

## 3. Get the image

Download or build the image as in the [Singularity](singularity-basics.md) pill: on `software23`, with Singularity's cache and temporary folder in `/tmp`. Then check it runs:

```bash
ssh software23
cd /tmp
singularity pull seqkit-2.8.2.sif docker://quay.io/biocontainers/seqkit:2.8.2--h9ee0642_1
singularity exec seqkit-2.8.2.sif seqkit version
exit
```

Copy the `.sif` somewhere you can reach from a login node, e.g. your scratch folder. The rest of this pill runs on a login node or in an `interactive` session, not on `software23` (Singularity there can't see `/jic/scratch`).

## 4. Create the install folder

```bash
TOOL=seqkit
VERSION=2.8.2
UI=/jic/software/user-installs
DEST=$UI/software/$TOOL/$VERSION

ls $UI/modules/$TOOL 2>/dev/null     # check nobody has this version already

mkdir -p "$DEST/bin" "$UI/modules/$TOOL"
cp /jic/scratch/groups/<your-group>/<you>/seqkit-2.8.2.sif "$DEST/image.img"
```

> **Never overwrite someone else's version.** If `ls` shows the version already exists, use theirs, or pick a version name that says how yours differs (Section 7).

## 5. Write the wrappers

A wrapper is a three-line script that runs the command of the same name inside `image.img`. It also binds the institute file systems, so the tool can see files on scratch:

```bash
cat > "$DEST/bin/$TOOL" <<'EOF'
#!/bin/bash
# Runs the command named like this file from the image next to bin/
here=$(dirname "$(readlink -f "$0")")
binds=""
for d in /jic /nbi /ei /tsl /qib; do
  if [[ -d $d ]]; then binds+="${binds:+,}$d"; fi
done
exec singularity exec ${binds:+-B "$binds"} "$here/../image.img" "$(basename "$0")" "$@"
EOF
chmod 755 "$DEST/bin/$TOOL"
```

The wrapper runs whatever command matches its own filename. So if a tool provides several commands, copy the same wrapper under each name:

```bash
cp "$DEST/bin/minimap2" "$DEST/bin/paftools.js"    # e.g. minimap2 provides both
```

## 6. Write the modulefile

Use this format, which is the one the user-installs tools generate. Fill in the name, a one-line description, the version, the commands and your username:

```bash
cat > "$UI/modules/$TOOL/$VERSION.lua" <<EOF
help([[
Description
===
Provides access to commands provided by the SeqKit $VERSION installation.

Use \`module whatis\`, \`module spider\` or \`module show\` to see more details.
]])

if (mode() == "load") then
    LmodMessage("Loaded module " .. myModuleFullName())
end

whatis("Name: SeqKit")
whatis("Description: A toolkit for FASTA/Q file manipulation.")
whatis("Version: $VERSION")
whatis("Command List: seqkit")
whatis("Deployment Type: Singularity")
whatis("Mutable: False")
whatis("Maintainer/Contact: $USER")

local base = "$DEST"
prepend_path("PATH",pathJoin(base,"bin"))
EOF
```

- **`Deployment Type: Singularity`** and **`Mutable: False`**: the install is a container and won't change.
- **`Maintainer/Contact`**: you. People will come to you with questions about this install.

**Make it readable by everyone:**

```bash
chmod 755 "$DEST" "$DEST/bin"
chmod 644 "$DEST/image.img" "$UI/modules/$TOOL/$VERSION.lua"
```

## 7. Naming versions

The version is the folder name and what people type after `ml <tool>/`, so make it say exactly what's inside:

- **Upstream releases:** use the plain version, `seqkit/2.8.2`.
- **Anything built differently** from the upstream release: add a suffix, e.g. `jasmine/2.2.1-smrtlink13.1` for a jasmine taken from a PacBio SMRT Link installer rather than Bioconda. Then nobody confuses it with an upstream `jasmine/2.2.1`.
- **Never reuse a version name** for different contents. Scripts that load `seqkit/2.8.2` must get the same software next year.

## 8. Load and test it

```bash
ml spider seqkit          # should list 2.8.2
ml seqkit/2.8.2
seqkit version
seqkit stats /jic/scratch/groups/<your-group>/<you>/reads.fastq.gz   # a file on scratch
```

The user-installs modules are on everyone's default module path, so a new module shows up as soon as its modulefile exists. If `ml spider` doesn't find yours, check the modulefile's name and location first. You can also add the folder by hand for the current session:

```bash
ml use /jic/software/user-installs/modules
```

**In a Slurm job**, load it like any other module:

```bash
#!/bin/bash
#SBATCH -p jic-short
#SBATCH -c 4
#SBATCH --mem=4G
#SBATCH -t 0-01:00

ml seqkit/2.8.2
seqkit stats -j 4 *.fastq.gz > stats.txt
```

Finally, ask a colleague to load it from their account. That's the only real check that the permissions are right.

## 9. When it goes wrong

- **`ml` can't find the module:** check the modulefile is at `modules/<tool>/<version>.lua`, then try `ml use /jic/software/user-installs/modules` (Section 8).
- **Works for you, "Permission denied" for others:** a folder or file isn't readable by everyone. Rerun the `chmod` lines in Section 6.
- **`No such file or directory` for a file that exists:** the wrapper isn't binding that file system. Check the `binds` loop in the wrapper (Section 5).
- **`command not found` for one of the tool's commands:** there's no wrapper for it in `bin/`. Copy the wrapper under that name (Section 5).
- **`FATAL: could not open image`:** `image.img` is missing, renamed, or not readable.

## 10. Your user-installs survival kit

| Command / path | What it does |
| --- | --- |
| `/jic/software/user-installs/software/<tool>/<version>/` | The install: `image.img` + `bin/` |
| `/jic/software/user-installs/modules/<tool>/<version>.lua` | The modulefile |
| `ls /jic/software/user-installs/modules/<tool>` | See which versions already exist |
| `ml spider <tool>` | List all versions of a module |
| `ml use /jic/software/user-installs/modules` | Add the user-installs modules to this session |
| `ml show <tool>/<version>` | Show what a module does when loaded |
| `chmod 755 <dirs>` / `chmod 644 <files>` | Make an install readable by everyone |
| `singularity inspect --deffile image.img` | Show the recipe an image was built from |

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] List `/jic/software/user-installs/modules/` and open one modulefile with `ml show`
- [ ] Pull the BioContainers image of a small tool on `software23` and check it runs
- [ ] Choose a version name nobody has used, and create its `software/` folder with `image.img` and `bin/`
- [ ] Write the wrapper and modulefile, and fix the permissions
- [ ] Load it with `ml`, and run it on a file in your scratch folder
- [ ] Use it in a small Slurm job
- [ ] Ask a colleague to load it from their account
