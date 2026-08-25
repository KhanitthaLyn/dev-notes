# Git & Version Control Notes

## 1. Introduction to Git & Version Control

- **What is Git:** An essential tool utilized in almost every software engineering job role.
- **The "Version History" Concept:** Git operates similarly to the "version history" function in Google Docs or Microsoft Word. It saves snapshots of files at specific points in time, serving as a critical safety net that lets developers view and restore code to previous states if mistakes occur.
- **Repository:** A specific directory or folder tracked and managed by Git.

## 2. Git Installation

- Download the installer directly from [git-scm.com/downloads](https://git-scm.com/downloads).

**macOS**
Recommended via Xcode Command Line Tools:
```bash
git
```
If prompted by a pop-up window, click **Install** to automatically set up Git. Alternatively, download Xcode from the App Store.

**Windows**
Download the Windows version, run the installer using all default options, then verify the setup in PowerShell:
```powershell
git
```

## 3. Command Line & Navigation

Git is primarily interacted with through the command line (Terminal on Mac, PowerShell on Windows).

> **Essential concept:** All terminal instructions execute inside a specific directory. When launched, terminals default to the Home folder (`$HOME`).

| Command | Description |
|---|---|
| `ls` | Lists all contents (files and folders) inside the current directory |
| `cd <path>` | Change Directory — moves into your project folder, e.g. `cd ~/desktop/git-tutorial` |

## 4. Basic Git Workflow: Tracking & Saving Changes

```bash
git init
```
Initializes Git tracking inside your chosen project directory.

**User configuration** — set up your name and email to avoid commit-related errors. This attaches authorship to all your changes:
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

```bash
git status
```
Checks which files have been modified, created, or left untracked since the previous commit.

### Working Area vs. Staging Area

- **Working Area** — holds current modifications and raw file edits before you run a save command.
- **Staging Area** — where Git places changes gathered via `git add` to prepare them for the upcoming commit.

> **Note:** Git tracks *changes*, not files. If you modify a file further after staging it, it can appear in both the Working and Staging Areas simultaneously.

### Staging and Committing

| Command | Description |
|---|---|
| `git add <file>` | Tracks/selects specific file changes for the next commit |
| `git add .` | Stages all changes in the current directory and its subdirectories |
| `git commit -m "<message>"` | Creates a permanent snapshot (commit) in history with a descriptive message |
| `git commit --amend` | Alters the most recent commit — fix a typo or add a forgotten file without cluttering history |
| `git log` | Displays the active version history and list of past commits |
| `git diff` | Shows line-by-line changes in the Working Area not yet staged |
| `git diff --staged` | Shows changes that are staged but not yet committed (alias: `--cached`) |

Use `git diff` right before `git add` or `git commit` to review exactly what's about to be saved.

## 5. Undoing & Resetting Changes

| Command | Description |
|---|---|
| `git reset <file>` / `git reset .` | Unstages files, moving them back into the Working Area |
| `git checkout -- <file>` / `git checkout -- .` | Discards raw edits in the Working Area, reverting to the last commit |

### Reset Modes

For undoing commits (not just unstaging):

| Mode | Effect |
|---|---|
| `git reset --soft <hash>` | Moves the branch pointer back; keeps changes **staged** |
| `git reset --mixed <hash>` *(default)* | Moves the pointer back; keeps changes in the **Working Area**, unstaged |
| `git reset --hard <hash>` | Moves the pointer back and **permanently discards** all changes — use with caution |

### Safer Alternative: `git revert`

```bash
git revert <commit-hash>
```
Creates a **new commit** that reverses a specific past commit, without rewriting history. This is the safer option on a shared/remote branch, since `git reset` rewrites history that others may have already pulled.

### Stashing Work in Progress

```bash
git stash          # save current changes (staged + unstaged) without committing
git stash list     # show all stashed change sets
git stash pop      # re-apply the most recent stash and remove it from the list
git stash apply    # re-apply the most recent stash but keep it in the list
```
Useful when you need to switch branches quickly but aren't ready to commit.

## 6. Code Editor Integration

Modern editors like **VS Code** have native Git integrations. Developers can visually inspect files in a Git panel (similar to `git status`), stage edits with a **+** icon (similar to `git add`), and discard modifications visually.

## 7. Viewing and Restoring History

```bash
git log --all
```
Lists the entire commit history, including commits positioned ahead of the current view.

- **HEAD** — a pointer indicating which commit/version you are currently viewing.
- **Viewing old commits:** `git checkout <commit-hash>` temporarily views a historical version of the codebase.
- **The branching side effect:** editing and committing while on a detached old commit creates a separate branch — history is no longer linear (visualize with `git log --all --graph`).

### Restoring a Previous Version (Google Docs equivalent)

To restore an old state while keeping a linear, single-branch history:
```bash
git checkout <commit-hash> .
git commit -m "version restored"
```
The first command copies the old files into your current directory and stages them automatically; the commit finalizes the restore without branch deviations.

> **Branch name:** labels like `master` or `main` point to the latest commit on that branch, making it easy to return via `git checkout master`.

## 8. Branching

A **branch** is an independent line of development — a movable pointer to a commit. Branching lets you work on new features or fixes without affecting the main codebase (`main`/`master`) until you're ready to combine the work back in.

| Command | Description |
|---|---|
| `git branch` | Lists local branches; current branch marked with `*` |
| `git branch <name>` | Creates a new branch without switching to it |
| `git checkout <name>` | Switches the Working Area to the specified branch |
| `git checkout -b <name>` | Creates and switches to a new branch in one step |
| `git switch <name>` | Modern alternative to `checkout` for switching branches |
| `git switch -c <name>` | Modern alternative to `checkout -b` |
| `git branch -d <name>` | Deletes a branch (only if already merged) |
| `git branch -D <name>` | Force-deletes an unmerged branch |

**Typical workflow:** create a branch for a new feature → commit changes on that branch → merge it back into `main` once complete and tested.

## 9. Merging & Conflict Resolution

```bash
git merge <branch-name>
```
While on your target branch (e.g. `main`), combines the history and changes from `<branch-name>` into it.

- **Fast-forward merge** — if no new commits exist on the target branch since the feature branch was created, Git simply moves the pointer forward (no merge commit).
- **Three-way merge** — if both branches have diverged, Git creates a new **merge commit** tying the two histories together.

### Merge Conflicts

Occur when the same lines of the same file are changed differently on both branches, and Git can't automatically decide which version to keep. Git marks the conflict directly in the file:

```
<<<<<<< HEAD
your current branch's version
=======
the incoming branch's version
>>>>>>> branch-name
```

**Resolving a conflict:**
1. Manually edit the file to keep the correct code (or a combination of both).
2. Remove the conflict markers.
3. Stage and commit:
```bash
git add <file>
git commit
```

> Editors like VS Code display conflicts visually with buttons like *Accept Current Change*, *Accept Incoming Change*, or *Accept Both Changes*.

## 10. Remote Repositories & Collaboration

A **remote** is a version of your repository hosted elsewhere (e.g. GitHub, GitLab, Bitbucket) that allows multiple people to collaborate on the same project.

| Command | Description |
|---|---|
| `git clone <url>` | Downloads a full copy of a remote repository, including its history, and connects it as `origin` |
| `git remote add origin <url>` | Manually connects a local repo (started with `git init`) to a remote |
| `git remote -v` | Lists connected remotes and their URLs |
| `git push origin <branch>` | Uploads local commits on the specified branch to the remote |
| `git push -u origin <branch>` | Pushes and sets up tracking so future `push`/`pull` don't need the remote/branch specified |
| `git fetch` | Downloads new commits/branches from the remote without merging them |
| `git pull` | `git fetch` + `git merge` — downloads and immediately merges remote changes into the current branch |

**Pull Request (PR) / Merge Request (MR):** a platform feature (GitHub/GitLab — not a native Git command) where a developer proposes merging their branch into another, allowing teammates to review, comment, and approve before merging.

### Typical Team Workflow

```bash
git clone <url>
git checkout -b feature/my-feature
# make changes and commit
git push -u origin feature/my-feature
# open a Pull Request on GitHub/GitLab
# after review and approval, merge the PR
git checkout main
git pull
```

## 11. Miscellaneous Features

**Aliases** — set up shortcuts to save keystrokes:
```bash
git config --global alias.s status
# now `git s` works instead of `git status`
```

**`.gitignore`** — forces Git to ignore untracked files you never want in history, such as those holding API keys or passwords (e.g. `secret.txt`).

| Pattern | Effect |
|---|---|
| `secret.txt` | Ignores that specific file |
| `*.log` | Ignores all files with that extension |
| `node_modules/` | Ignores an entire folder |
| `!important.log` | Negates a pattern, re-including a file that would otherwise be ignored |

> GitHub maintains a collection of ready-made [`.gitignore` templates](https://github.com/github/gitignore) for specific languages/frameworks (Node, Python, Java, etc.) — a good starting point instead of writing one from scratch.

**Removing Git from a project** — all version records live in a hidden `.git` directory:
```bash
rm -rf .git
```
This permanently removes all Git history and stops tracking, returning the directory to a normal local folder.
