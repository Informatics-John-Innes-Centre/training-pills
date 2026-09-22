# Git Basics for Researchers

## Tracking your research scripts with Git

**Time:** about 1 hour
**Level:** Beginner — no prior Git experience needed

By the end of this tutorial, you will be able to:

- Explain why Git is useful for research scripts
- Create a Git repository
- Track changes to a script
- Create commits with meaningful messages
- View the history of your work
- Push a repository to GitLab

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

## 6. The basic Git workflow

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

## 7. Push your repository to GitLab

Once your local repository is working, connect it to a remote GitLab repository.

**Add the remote repository**

Use the repository URL provided by GitLab:

```bash
git remote add origin <repository-url>
```

For example:

```bash
git remote add origin https://gitlab.example.ac.uk/username/plant-analysis.git
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

## 8. Make another change and push it

Modify the script again, then:

```bash
git status
git diff
git add .
git commit -m "Update analysis parameters"
git push
```

Refresh GitLab — the new commit should now appear in the repository history.

## 9. Looking back at previous versions

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

## 10. Your Git survival kit

These are the commands to remember after this tutorial:

| Command | What it does |
| --- | --- |
| `git status` | Show what is happening |
| `git diff` | Show changes |
| `git add .` | Prepare changes for a commit |
| `git commit -m "message"` | Save a version |
| `git push` | Send commits to GitLab |
| `git log --oneline` | Show the history |

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
- [ ] View the history with `git log --oneline`
- [ ] Create/connect a GitLab repository
- [ ] Push the repository to GitLab
- [ ] Make another change
- [ ] Commit and push it
