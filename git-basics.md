# Git Basics for Researchers

## Tracking your research scripts with Git

**Time:** about 75–90 minutes
**Level:** Beginner — no prior Git experience needed

By the end of this tutorial, you will be able to:

- Explain why Git is useful for research scripts
- Create a Git repository
- Track changes to a script
- Tell Git to ignore files you don't want tracked
- Create commits with meaningful messages
- View the history of your work
- Push a repository to GitLab
- Use a branch to try out changes safely

The main idea to take away:

> **Git gives you a history of your scripts, so you can see what changed and go back to an earlier version.**

---

## 1. Why Git?

You've probably seen (or written) something like this:

```text
analysis.py
analysis_v2.py
analysis_v2_final.py
analysis_v2_final2.py
analysis_v2_final_REALLY_FINAL.py
```

Git replaces this with a single script that carries its own history:

```text
        change        change        change
           ↓             ↓             ↓
Version 1 ────── Version 2 ────── Version 3
```

Why this is useful for you as a researcher:

- Keep a history of your scripts
- See exactly what changed and when
- Go back to an earlier version if something breaks
- Experiment without losing previous work
- Share scripts with colleagues
- Make your computational work more reproducible

This tutorial focuses on tracking scripts, not on becoming a software developer — you only need a handful of commands.

## 2. Git vs GitLab

**Git** is the version-control system that runs on your computer.

**GitLab** is where you can store and share your Git repository with others.

```text
Your computer                 JIC GitLab
     │                            │
     │        git push            │
     ├───────────────────────────►│
     │                            │
     │        git pull            │
     │◄───────────────────────────┤
```

In this tutorial, you will first work locally and then push your repository to GitLab.

## 3. Create your first repository

You'll use a small example research project:

```text
plant-analysis/
└── analyse.py
```

**Step 1 — Open the project directory**

```bash
cd plant-analysis
```

**Step 2 — Initialise Git**

```bash
git init
```

This tells Git: "Start tracking this directory as a Git repository."

**Step 3 — Check the repository**

```bash
git status
```

> **Tip:** `git status` is your friend. Whenever you're unsure what's happening, run `git status`.

## 4. Make your first commit

Git does not automatically save every change you make. There are two steps before a change becomes part of the repository's history:

```text
Working directory
       │
       │ git add
       ▼
Staging area
       │
       │ git commit
       ▼
Repository history
```

**Step 1 — Add the script**

```bash
git add analyse.py
```

You can also add everything in the current directory:

```bash
git add .
```

**Step 2 — Commit the change**

```bash
git commit -m "Add initial analysis script"
```

A commit is a snapshot of your project at that point in time.

**Step 3 — Look at the history**

```bash
git log --oneline
```

You should see something similar to:

```text
3c92a11 Add initial analysis script
```

## 5. Make a change

Now modify `analyse.py` — for example, change the analysis or add another calculation.

**Check what changed**

```bash
git status
git diff
```

`git diff` shows the changes you've made since the last commit. This is especially useful with research scripts, because you can see exactly what changed.

**Save the new version**

```bash
git add .
git commit -m "Update analysis"
```

Check the history again:

```bash
git log --oneline
```

You should now see something similar to:

```text
a82f123 Update analysis
3c92a11 Add initial analysis script
```

Every commit represents a version of your work.

## 6. Ignoring files with `.gitignore`

`git add .` stages *everything* in the directory — including files you probably don't want in your Git history, such as:

- Large or raw data files (`*.csv`, `*.fastq`, `results/`)
- Generated output (plots, `*.png`, `*.pdf`)
- Cache and temporary files (`__pycache__/`, `.ipynb_checkpoints/`, `.Rhistory`)
- Environment or secret files (`.env`, credentials, API keys)
- Editor/OS clutter (`.DS_Store`, `.vscode/`)

Committing these bloats your repository, and secrets should never end up in Git history at all.

**Step 1 — Create a `.gitignore` file** in the root of your project:

```bash
touch .gitignore
```

**Step 2 — List the files and folders to ignore**, one pattern per line:

```text
__pycache__/
.ipynb_checkpoints/
*.csv
results/
.env
```

**Step 3 — Check the effect**

```bash
git status
```

Ignored files no longer show up as untracked, even after `git add .`.

**Step 4 — Commit the `.gitignore` file itself**, so the rules are shared with anyone who clones the repository:

```bash
git add .gitignore
git commit -m "Add .gitignore"
```

> **Important:** `.gitignore` only affects files that are not already tracked. If a file was committed *before* you added it to `.gitignore`, Git will keep tracking it until you explicitly remove it with `git rm --cached <file>`.

## 7. The basic Git workflow

The one workflow to take away from this tutorial:

```text
EDIT
  ↓
git status
  ↓
git diff
  ↓
git add .
  ↓
git commit
```

In practice:

```bash
git status
git diff
git add .
git commit -m "Describe what changed"
```

**Write good commit messages** that describe the actual change:

- `Add filtering for low-quality samples`
- `Fix calculation of mean expression`
- `Update analysis parameters`
- `Add sample metadata validation`

Avoid vague messages such as `changes`, `update`, `fix`, `stuff`, `final`.

## 8. Push your repository to GitLab

Once your local repository is working, connect it to a remote GitLab repository.

**Add the remote repository**

Use the repository URL provided by GitLab:

```bash
git remote add origin <repository-url>
```

For example:

```bash
git remote add origin https://git.nbi.ac.uk/username/plant-analysis.git
```

**Push your repository**

```bash
git push -u origin main
```

Open the repository in GitLab. You should now be able to see:

- Your script
- Your README, if you have one
- Your commits
- The history of your project

## 9. Make another change and push it

Modify the script again, then:

```bash
git status
git diff
git add .
git commit -m "Update analysis parameters"
git push
```

Refresh GitLab — the new commit should now appear in the repository history.

## 10. Working with branches

So far, every commit has gone straight onto your main line of history (`main`). A **branch** lets you try something out — a new analysis approach, a risky refactor — without touching the working version of your script.

```text
                  ┌── new-analysis: try alternative method
                  │
main ─────●───────●──────────────●───────► (unaffected, still works)
          │                      │
          └── you create the branch here, and merge back here
```

**Step 1 — Create and switch to a new branch**

```bash
git checkout -b new-analysis
```

This creates a branch called `new-analysis` and moves you onto it. Your files are unchanged, but any commits you make now only exist on this branch.

**Step 2 — Work as normal**

```bash
# edit analyse.py
git status
git add .
git commit -m "Try alternative normalisation method"
```

**Step 3 — Switch back to `main` at any time**

```bash
git checkout main
```

Notice that your script reverts to the `main` version — the changes are safely tucked away on `new-analysis` until you're ready for them.

**Step 4 — If the new approach works, merge it back into `main`**

```bash
git checkout main
git merge new-analysis
```

**Step 5 — If you want to share the branch on GitLab**

```bash
git push -u origin new-analysis
```

This lets colleagues review your work (e.g. via a GitLab merge request) before it becomes part of `main`.

**See which branch you're on, or list all branches:**

```bash
git branch
```

The branch with a `*` next to it is the one you're currently on.

> **Why this matters for research:** branches let you try a different statistical method, a new set of parameters, or a reanalysis — without risking the version of the script that already works.

## 11. Looking back at previous versions

View the history:

```bash
git log --oneline
```

You might see:

```text
a82f123 Update analysis parameters
7d91abc Update analysis
3c92a11 Add initial analysis script
```

Each commit represents a previous version of the project. You can inspect a particular commit with:

```bash
git show <commit>
```

For example:

```bash
git show 3c92a11
```

This lets you see exactly what was included in an earlier version.

> **Key concept:** A commit is a saved snapshot of your work. The more meaningful your commits are, the easier it is to understand the history later.

## 12. Your Git survival kit

These are the commands to remember after this tutorial:

| Command | What it does |
| --- | --- |
| `git status` | Show what is happening |
| `git diff` | Show changes |
| `git add .` | Prepare changes for a commit |
| `git commit -m "message"` | Save a version |
| `git push` | Send commits to GitLab |
| `git log --oneline` | Show the history |
| `.gitignore` | List files Git should never track |
| `git checkout -b <name>` | Create and switch to a new branch |
| `git checkout <name>` | Switch to an existing branch |
| `git merge <name>` | Bring a branch's changes into your current branch |

**The workflow to remember:**

```text
       ┌─────────────┐
       │ Edit script │
       └──────┬──────┘
              ↓
       git status
              ↓
         git diff
              ↓
         git add .
              ↓
       git commit
              ↓
         git push
```

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Create a project directory
- [ ] Run `git init`
- [ ] Check the repository with `git status`
- [ ] Add a research script
- [ ] Create your first commit
- [ ] Modify the script
- [ ] Use `git diff`
- [ ] Create a second commit
- [ ] Create a `.gitignore` file and add a couple of patterns to it
- [ ] Confirm the ignored files don't appear in `git status`
- [ ] Commit the `.gitignore` file
- [ ] View the history with `git log --oneline`
- [ ] Create/connect a GitLab repository
- [ ] Push the repository to GitLab
- [ ] Create a new branch and make a commit on it
- [ ] Switch back to `main` and confirm your script reverts
- [ ] Merge the branch back into `main`
- [ ] Make another change on `main`
- [ ] Commit and push it
