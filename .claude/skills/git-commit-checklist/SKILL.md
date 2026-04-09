---
name: git-commit-checklist
description: Use when the user asks to commit, make a git commit, or save changes to this repo. Runs the project-specific pre-commit checklist before committing.
version: 1.0.0
user-invocable: true
---

# Pre-Commit Checklist for share-claude-skills

Before committing, run through this checklist.

## Update the README file tree

Check whether the file tree in `README.md` matches the current repo contents.

Get the current tree (excluding `.git`):
```bash
tree -I '.git' --dirsfirst
```

**Tree format rules:**
- Count how many lines each skill occupies in the tree (every file and subdirectory line under `skills/<skill-name>/` counts).
- If **every** skill is 10 lines or fewer: use a full listing showing all files.
- If **any** skill exceeds 10 lines: collapse that skill to show only its top-level contents. List subdirectories with a suffix like `(directory containing 5 files)`.

Update the fenced code block under `## What's here` in `README.md` to match, then stage it with `README.md` in the same commit.
