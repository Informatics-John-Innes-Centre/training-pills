# Git Basics for Researchers

## Git for tracking research scripts

**Duration:** 1 hour
**Audience:** Science/biological researchers who write or use scripts
**Level:** Beginner

### Training aim

By the end of this session, participants should be able to:

- Understand why Git is useful for research scripts
- Create a Git repository
- Track changes to a script
- Create commits with meaningful messages
- View the history of their work
- Push a repository to GitLab

The main idea:

> **Git gives you a history of your scripts, so you can see what changed and go back to an earlier version.**

---

## 1. Why Git? — 0–5 min

Start with a familiar problem:

```text
analysis.py
analysis_v2.py
analysis_v2_final.py
analysis_v2_final2.py
analysis_v2_final_REALLY_FINAL.py
```

Git provides a history of the same script instead:

```text
        change        change        change
           ↓             ↓             ↓
Version 1 ────── Version 2 ────── Version 3
```

Why is this useful for researchers?

- Keep a history of scripts
- See what changed
- Go back to an earlier version
- Experiment without losing previous work
- Share scripts with colleagues
- Make computational work more reproducible

The focus of this training is tracking scripts, not becoming a software developer.

## 2. Git vs GitLab — 5–10 min

**Git** is the version-control system running on your computer.

**GitLab** can store and share your Git repository.

Basic idea:

```text
Your computer                 JIC GitLab
     │                            │
     │        git push            │
     ├───────────────────────────►│
     │                            │
     │        git pull            │
     │◄───────────────────────────┤
```

For this training, we will first work locally and then push the repository to GitLab.

## 3. Create your first repository — 10–25 min

We will use a small example research project.

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

> **Important:** `git status` is your friend. When you are unsure what is happening, run `git status`.

## 4. Make your first commit

Git does not automatically save every change you make.

There are two steps before a change becomes part of the repository history:

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

## 5. Make a change — 25–40 min

Now modify `analyse.py`. For example, change the analysis or add another calculation.

**Check what changed**

```bash
git status
git diff
```

`git diff` shows the changes you have made since the last commit. This is particularly useful when working with research scripts because you can see exactly what changed.

**Save the new version**

```bash
git add .
git commit -m "Update analysis"
```

Check the history again:

```bash
git log --oneline
```

You should now have something similar to:

```text
a82f123 Update analysis
3c92a11 Add initial analysis script
```

Every commit represents a version of your work.

## 6. The basic Git workflow

The most important workflow from this training is:

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

**Good commit messages** try to describe the actual change:

- `Add filtering for low-quality samples`
- `Fix calculation of mean expression`
- `Update analysis parameters`
- `Add sample metadata validation`

Avoid vague messages such as `changes`, `update`, `fix`, `stuff`, `final`.

## 7. Push the repository to GitLab — 40–50 min

Once the local repository is working, connect it to a remote GitLab repository.

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

Modify the script again. Then:

```bash
git status
git diff
git add .
git commit -m "Update analysis parameters"
git push
```

Refresh GitLab. The new commit should now appear in the repository history.

## 9. Looking back at previous versions — 50–55 min

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

This allows you to see what was included in an earlier version.

> **Key concept:** A commit is a saved snapshot of your work. The more meaningful your commits are, the easier it is to understand the history later.

## 10. The Git survival kit — 55–60 min

These are the commands to remember after the training:

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

### Practical exercise

During the session, participants should complete the following:

- [ ] Create a project directory
- [ ] Run `git init`
- [ ] Check the repository with `git status`
- [ ] Add a research script
- [ ] Create the first commit
- [ ] Modify the script
- [ ] Use `git diff`
- [ ] Create a second commit
- [ ] View the history with `git log --oneline`
- [ ] Create/connect a GitLab repository
- [ ] Push the repository to GitLab
- [ ] Make another change
- [ ] Commit and push it
