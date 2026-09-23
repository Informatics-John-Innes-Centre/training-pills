# nano Basics for Bioinformatics

## Editing scripts and config files directly on the server

**Time:** about 20–30 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Open and create files with `nano`
- Move around and edit text without a mouse
- Save changes and exit
- Search for and replace text
- Cut, copy, and paste lines
- Avoid the most common nano mistakes

The main idea to take away:

> **nano is the simplest editor available on almost every server — you don't need to leave the terminal (or your tmux session) to fix a script.**

---

## 1. Why nano?

Working on an HPC, you'll constantly need to tweak something without transferring a file back and forth to your laptop: a Slurm submission script, a sample sheet, a config file, a one-line fix to a pipeline script. `nano` is installed almost everywhere, has no learning curve to speak of, and — unlike `vim`/`emacs` — shows you the available commands on screen the whole time.

```text
$ nano align_job.sh
```

opens (or creates) the file directly in your terminal, ready to edit.

## 2. Reading the screen

When nano opens, the bottom of the screen shows the commands available, using `^` for `Ctrl`:

```text
^G Get Help   ^O Write Out   ^W Where Is   ^K Cut Text
^X Exit       ^R Read File   ^\ Replace    ^U Paste Text
```

So `^O` means "hold `Ctrl` and press `O`". You don't need to memorise the full list — it's always on screen if you forget.

## 3. Opening and creating files

```bash
nano align_job.sh
```

If `align_job.sh` doesn't exist yet, nano creates it — there's no separate "new file" step.

**Open at a specific line** (useful when an error message tells you exactly where the problem is):

```bash
nano +42 align_job.sh
```

## 4. Moving around and editing

Arrow keys move the cursor as normal. A few shortcuts that save time:

| Shortcut | Action |
| --- | --- |
| `Ctrl+A` | Jump to the start of the line |
| `Ctrl+E` | Jump to the end of the line |
| `Ctrl+Y` | Page up |
| `Ctrl+V` | Page down |

Typing just inserts text at the cursor, and `Backspace`/`Delete` remove characters — no special "insert mode" to switch into.

## 5. Saving and exiting

**Save (Write Out)**

```text
Ctrl+O
```

nano will show the current filename and ask you to confirm — press `Enter` to save under that name.

**Exit**

```text
Ctrl+X
```

If you have unsaved changes, nano asks `Save modified buffer?` before exiting — `Y` to save and exit, `N` to discard changes and exit, `Ctrl+C` to cancel and go back to editing.

> **Tip:** `Ctrl+O` then `Ctrl+X` is the two-keystroke habit worth building — save, then exit.

## 6. Searching and replacing

**Search for text**

```text
Ctrl+W
```

Type what you're looking for and press `Enter`. Press `Ctrl+W` again, then `Enter`, to jump to the next match.

**Search and replace**

```text
Ctrl+\
```

nano asks what to search for, then what to replace it with, then lets you confirm each replacement one at a time (`Y`/`N`), or `A` to replace all remaining matches at once.

## 7. Cutting, copying, and pasting lines

```text
Ctrl+K   cut the current line (and add it to the clipboard)
Ctrl+U   paste whatever was last cut
```

Pressing `Ctrl+K` repeatedly on consecutive lines cuts them all into the same clipboard, so you can then move a whole block with a single `Ctrl+U` at the new location.

## 8. A couple of things worth knowing before you rely on nano

- **Line numbers**: start nano with `nano -l align_job.sh` (or press `Alt+#` inside nano) to show line numbers — handy when a tool's error message points you to a specific line.
- **Tabs vs spaces matter in some files.** If you're editing a YAML config or a Python script, be aware that mixing tabs and spaces can break it in ways that are hard to spot visually. If in doubt, check your editor's whitespace settings, or use `cat -A file` to reveal hidden tab/space characters.
- **Executable scripts**: nano only edits the file's contents — if you've just written a new shell script, remember to make it executable afterwards:

```bash
chmod +x align_job.sh
./align_job.sh
```

## 9. Your nano survival kit

| Shortcut | What it does |
| --- | --- |
| `nano file` | Open (or create) a file |
| `Ctrl+O` | Save (Write Out) |
| `Ctrl+X` | Exit |
| `Ctrl+W` | Search |
| `Ctrl+\` | Search and replace |
| `Ctrl+K` | Cut the current line |
| `Ctrl+U` | Paste |
| `Ctrl+G` | Show help |

**The workflow to remember:**

```text
nano file
  ↓
edit as needed
  ↓
Ctrl+O   (save)
  ↓
Enter    (confirm filename)
  ↓
Ctrl+X   (exit)
```

## Practice exercise

Work through the following steps yourself to make sure everything sticks:

- [ ] Create a new file with `nano test.sh`
- [ ] Type a few lines of text
- [ ] Save with `Ctrl+O`, then exit with `Ctrl+X`
- [ ] Reopen the file and confirm your changes are there
- [ ] Use `Ctrl+W` to search for a word in the file
- [ ] Use `Ctrl+\` to replace one word with another
- [ ] Cut a line with `Ctrl+K` and paste it somewhere else with `Ctrl+U`
- [ ] Make the file executable with `chmod +x` and run it
