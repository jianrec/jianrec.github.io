---
title: "Git Common Commands and Workflow Notes"
date: 2026-06-24 19:25:00 +0800
tags: [Git, GitHub, Version Control, Command Line, Developer Tools]
main_category: "Technical Notes"
discipline: "Other"
course: "Git"
material_type: "Study Notes"
description: "Everyday Git commands organized along a few main threads: the working tree, the staging area, the local repository, remotes, and branch collaboration."
ref: git-common-commands-and-workflow
lang: en
---

Learning Git is not about memorizing commands. What actually gets confusing is: which layer a file is in right now, where you want it to go next, and whether a given command will lose something.

These notes follow the order of everyday use. First get the working tree, staging area, local repository and remote repository pipeline working end to end, then fill in rollback, undo, branches and collaboration.

<!--more-->

## Start with Git's Three Layers

Almost all everyday Git operations revolve around these three layers:

```text
Working Tree
  the files you are currently editing in the folder

Staging Area / Index
  changes already added with git add, ready to go into the next commit

Repository
  the version history already saved with git commit
```

A routine commit looks like this:

```bash
git status
git diff
git add .
git commit -m "describe change"
git push
```

When in doubt, run `git status` first. It is the cheapest command and the best protection against mistakes.

## Installation and Initialization

Installing on macOS:

```bash
brew install git
```

The first time you use it, configure your identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global --list
```

Initialize a repository after entering the project directory:

```bash
git init
```

After initialization a `.git` directory appears — this is where Git stores version information. It is hidden by default; you can see it with:

```bash
ls -ah
```

To show hidden files in Finder:

```text
Command + Shift + .
```

## Your First Commit

Add a single file:

```bash
git add README.md
```

Add all changes in the current directory:

```bash
git add .
```

Commit to the local repository:

```bash
git commit -m "init project"
```

`git add` only puts things in the staging area; only `git commit` creates a local version.

## Checking Status and History

Check the current status:

```bash
git status
```

See changes in the working tree that are not yet staged:

```bash
git diff
```

See the commit history:

```bash
git log
```

The more commonly used compact version:

```bash
git log --oneline --graph --decorate --all
```

This command shows commits, branches and merge relationships all at once.

## Rolling Back Versions

`HEAD` refers to the current commit:

```text
HEAD      current commit
HEAD^     previous commit
HEAD^^    the one before that
HEAD~10   10 commits back
```

Roll back to the previous commit:

```bash
git reset --hard HEAD^
```

Roll back to a specific commit:

```bash
git reset --hard <commit_id>
```

You need to keep the three `reset` modes straight:

| Command | commit | Staging area | Working tree | When to use |
| --- | --- | --- | --- | --- |
| `git reset --soft <id>` | rolled back | kept | kept | Undo the commit but keep changes staged |
| `git reset --mixed <id>` | rolled back | reset | kept | Undo the commit and the add, keep the file edits |
| `git reset --hard <id>` | rolled back | reset | reset | Go all the way back to a given version |

`--hard` changes the working tree directly, and unsaved edits are lost. Check `git status` before using it.

If you regret a rollback, you can use:

```bash
git reflog
git reset --hard <commit_id>
```

`git log` shows the commit history; `git reflog` shows the local record of `HEAD` movements. It rescues a lot of local mistakes.

## Undoing Changes

Handle it according to where the change lives:

| Scenario | Command |
| --- | --- |
| Working tree is messed up, not yet added | `git restore <file>` |
| Already added, want to unstage | `git restore --staged <file>` |
| Unstaged it and now want to discard the file edits too | `git restore <file>` |
| Already committed, not yet pushed | `git reset --soft/--mixed/--hard <id>` |
| Already pushed, do not want to rewrite remote history | `git revert <commit_id>` |

The old commands still work:

```bash
git checkout -- <file>
git reset HEAD <file>
```

These days I prefer `restore` and `switch` because the semantics are clear: `restore` handles file restoration, `switch` handles branch switching.

## Deleting Files

Delete a file and stage the deletion:

```bash
git rm <file>
git commit -m "remove file"
```

If you deleted it manually, you can also do:

```bash
git add <file>
git commit -m "remove file"
```

Deleted by mistake and not yet committed:

```bash
git restore <file>
```

## Connecting to GitHub

Link a remote repository:

```bash
git remote add origin git@github.com:<user>/<repo>.git
git remote -v
```

Push for the first time and bind the upstream:

```bash
git push -u origin main
```

If the default branch is called `master`:

```bash
git push -u origin master
```

Once bound, day-to-day syncing is just:

```bash
git push
git pull
```

Remove the remote link:

```bash
git remote rm origin
```

Clone a remote repository:

```bash
git clone git@github.com:<user>/<repo>.git
```

## SSH Key

First check whether the machine already has a key:

```bash
ls -ah ~/.ssh
```

Generate a new key:

```bash
ssh-keygen -t ed25519 -C "email@example.com"
```

On older setups you can use RSA:

```bash
ssh-keygen -t rsa -C "email@example.com"
```

Copy the public key:

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

The RSA equivalent:

```bash
pbcopy < ~/.ssh/id_rsa.pub
```

`id_xxx` is the private key — never leak it; `id_xxx.pub` is the public key, which you can paste into GitHub's SSH Keys page.

## Branches

List branches:

```bash
git branch
```

Create a branch:

```bash
git branch &lt;name&gt;
```

Switch branches:

```bash
git switch &lt;name&gt;
```

Create and switch:

```bash
git switch -c &lt;name&gt;
```

The old commands:

```bash
git checkout &lt;name&gt;
git checkout -b &lt;name&gt;
```

Merge a branch into the current branch:

```bash
git merge &lt;name&gt;
```

Delete a merged branch:

```bash
git branch -d &lt;name&gt;
```

Force-delete an unmerged branch:

```bash
git branch -D &lt;name&gt;
```

`-D` throws away the unmerged commits on that branch, so only use it once you are sure you do not need them.

## Conflicts

When a merge fails, check the status first:

```bash
git status
```

The conflicted file will contain:

```text
<<<<<<< HEAD
content on the current branch
=======
content from the merged branch
>>>>>>> feature
```

Edit it manually into what you actually want, then:

```bash
git add <file>
git commit
```

View the branch graph:

```bash
git log --graph --oneline --decorate --all
```

## stash

Your work is not finished but you need to switch branches temporarily:

```bash
git stash
```

List them:

```bash
git stash list
```

Restore and drop the most recent stash:

```bash
git stash pop
```

Restore only, without deleting the record:

```bash
git stash apply
```

## cherry-pick

Copy a specific commit from another branch onto the current branch:

```bash
git cherry-pick <commit_id>
```

A typical scenario is a bug fix that has already landed on `main` while `dev` needs the same change. `cherry-pick` copies a single commit, not an entire branch merge.

## Working with Others

List remote branches:

```bash
git branch -r
```

Pull remote updates:

```bash
git pull
```

Push the current branch:

```bash
git push origin <branch-name>
```

Create a local branch based on a remote branch:

```bash
git switch -c <branch-name> origin/<branch-name>
```

Set the upstream:

```bash
git branch --set-upstream-to=origin/<branch-name> <branch-name>
```

If you do not push a new local branch, nobody else can see it. A failed push usually means the remote already has new commits — pull or fetch first, then resolve the conflicts.

## rebase

Tidy up local commit history that has not been pushed:

```bash
git fetch origin
git rebase origin/main
```

This suits the case where the local branch has not been pushed yet and you want a straighter history. Do not casually rebase a public branch that has already been pushed and that others have built on top of.

## .gitignore

`.gitignore` holds the files that should not go into the repository:

```gitignore
.DS_Store
node_modules/
.venv/
__pycache__/
*.log
.env
```

Common things to ignore:

- Files generated automatically by the operating system, such as `.DS_Store`.
- Build artifacts, caches and temporary files.
- Local virtual environments.
- Local configuration containing passwords, tokens or keys.

`.gitignore` itself should be committed to Git so the whole team shares the same rules.

If a file is already tracked by Git, adding it to `.gitignore` later does not automatically take effect; you need:

```bash
git rm --cached <file>
```

## Handy macOS Commands

Go back to the user home directory:

```bash
cd ~
```

Jump to the home directory in Finder:

```text
Command + Shift + H
```

Show hidden files:

```bash
ls -ah
```

List files by modification time, newest at the bottom:

```bash
ls -ltr
```

View a file's contents:

```bash
cat <file>
```

List only the directory itself without expanding its contents:

```bash
ls -d .ssh
```

## The Workflows I Actually Use

Solo development:

```bash
git status
git diff
git add .
git commit -m "describe change"
git push
```

New feature:

```bash
git switch main
git pull
git switch -c feature/xxx

git add .
git commit -m "add xxx"
git push -u origin feature/xxx
```

Quick bug fix:

```bash
git stash
git switch main
git pull
git switch -c fix/bug-name

git add .
git commit -m "fix bug-name"
git push -u origin fix/bug-name

git switch <old-branch>
git stash pop
```

In the end, a few rules are enough:

- When in doubt, run `git status` first.
- `git add` only stages; it does not save history.
- `git commit` saves locally.
- `git push` syncs to the remote.
- `git reflog` can rescue a lot of local mistakes.
- Before using `reset --hard` or `branch -D`, make sure the working tree has no important changes.
