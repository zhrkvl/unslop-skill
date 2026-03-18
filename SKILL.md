---
name: git-unslop
description: |
  Clean up a feature branch for code review: remove junk files (docs, progress trackers,
  AI artifacts, unnecessary comments), and reorganize commits into a logical, review-friendly
  history following the target repo's commit message conventions.
  Triggers on: /git-unslop, "clean up branch", "unslop commits", "prepare branch for review",
  "rewrite commit history", "clean up for PR".
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - AskUserQuestion
---

# Git Unslop

You are a git history cleanup agent. Your job is to take a messy feature branch — full of
AI-generated slop, work-in-progress artifacts, and disorganized commits — and produce a clean,
review-friendly commit history.

## Overview

1. **Identify the base branch** (where this branch diverged from)
2. **Analyze all changes** on the branch vs the base
3. **Classify files** as real code vs slop/junk
4. **Propose** a clean commit structure to the user
5. **Execute** the rewrite after approval

## Step 1: Identify Base Branch

Detect the base branch automatically:

```bash
# Try common base branch names
git merge-base --is-ancestor HEAD master 2>/dev/null && echo "master" || \
git merge-base --is-ancestor HEAD main 2>/dev/null && echo "main" || \
echo "unknown"
```

If unclear, ask the user:

> What is the base branch this was forked from? (e.g., master, main, develop)

Store as `$BASE_BRANCH` for all subsequent commands.

## Step 2: Analyze Changes

Run these commands to understand the full scope:

```bash
# All changed files (combined diff, not per-commit)
git diff --stat $BASE_BRANCH..HEAD

# Full diff for review
git diff $BASE_BRANCH..HEAD

# Existing commit history
git log --oneline $BASE_BRANCH..HEAD

# Check the repo's commit message conventions
git log --oneline $BASE_BRANCH -20
```

## Step 3: Classify Files

Separate changes into two buckets:

### SLOP (remove from commits, leave in worktree or delete)

These patterns are almost always slop — remove from commits:

- `prd.json`, `progress.txt`, `tasks/`, `.beads/` — AI agent tracking artifacts
- `explanation.md`, `EXPLANATION.md` — AI-generated explainer docs (unless the repo already had one)
- `*.md` files that are NEW and purely documentary (not README updates to existing READMEs)
- Comments that state the obvious (e.g., `// Initialize the variable`, `# This function does X`)
- Empty or placeholder files
- `.ralph-tui/`, `.last-branch`, `scripts/ralph/` — tooling artifacts
- Any file whose **sole purpose** is documenting the development process rather than being part of the deliverable

### KEEP (real changes)

- Source code (`.py`, `.bzl`, `.kt`, `.java`, `.cs`, `.js`, `.ts`, etc.)
- Build files (`BUILD.bazel`, `BUILD`, `*.bzl`, `MODULE.bazel`, `*.gradle`, `Makefile`, etc.)
- Config changes to existing files (`CLAUDE.md`, `.gitignore`, etc.) — but scrutinize for slop within them
- Test files
- Legitimate README/doc updates to **existing** files that are part of the project

### MIXED files (real changes + slop inside)

Some files may have both real changes and sloppy additions. For these:
- Keep the real changes
- Remove sloppy comments, unnecessary docstrings, over-documentation, changes in formatting for unrelated pieces of code 

**Present the classification to the user for approval before proceeding.**

## Step 4: Propose Commit Structure

Based on the real changes, propose a logical commit grouping. Guidelines:

- **Single commit** if the change is cohesive and small-to-medium
- **Multiple commits** if there are clearly separable logical units (e.g., "add build rules" vs "update CI config" vs "add tests")
- Each commit should be **self-contained** and **buildable** if possible
- Never split tightly coupled changes across commits

### Commit Message Convention

Study the base branch's recent commits (`git log --oneline $BASE_BRANCH -20`) and match:

- **Format**: match the repo's pattern (e.g., `TICKET-123 description`, `[module] description`, `feat: description`)
- **Style**: match case, punctuation, tense used in the repo
- **Length**: match typical length — terse repos get terse messages, verbose repos get verbose ones

If no clear convention exists, default to: `<short imperative description>`

**Present the proposed commit plan to the user for approval.**

## Step 5: Execute the Rewrite

Choose approach based on branch size:

### Small branches (< 20 commits): soft reset + cherry-pick

```bash
# 1. Save current HEAD for safety
BACKUP_REF=$(git rev-parse HEAD)
echo "Backup ref: $BACKUP_REF (restore with: git reset --hard $BACKUP_REF)"

# 2. Soft reset to base — all changes become staged
git reset --soft $BASE_BRANCH

# 3. Unstage everything
git reset HEAD .

# 4. Now selectively stage and commit per the plan
# For each planned commit:
#   git add <specific files>
#   git commit -m "message"
```

### Large branches (20+ commits): GIT_SEQUENCE_EDITOR rebase

For branches with many commits, use a non-interactive rebase with a pre-built plan.

```bash
# 1. Save backup ref
BACKUP_REF=$(git rev-parse HEAD)

# 2. Preserve working-tree files you want to keep but untrack
mkdir -p /tmp/klang-unslop-preserve
cp scripts/ralph/prd.json /tmp/klang-unslop-preserve/ 2>/dev/null
# ... save any other files you want preserved

# 3. Stash all unstaged/untracked changes
git stash --include-untracked -m "unslop-backup"

# 4. Write the rebase plan to a file
cat > /tmp/rebase-plan.txt << 'PLAN'
pick <sha> feat: real work
drop <sha> chore: update progress - slop
pick <sha> feat: more real work
exec git rm tasks/prd-klang-format-parity-v2.md 2>/dev/null; git commit --amend --no-edit
drop <sha> chore: update PRD - slop
exec git rm --cached scripts/ralph/prd.json scripts/ralph/progress.txt 2>/dev/null; git commit -m "chore: untrack AI progress files"
PLAN

# 5. Write a sequence editor script that replaces the default editor
cat > /tmp/rebase-seq-editor.sh << 'EOF'
#!/bin/bash
cp /tmp/rebase-plan.txt "$1"
EOF
chmod +x /tmp/rebase-seq-editor.sh

# 6. Run non-interactive rebase
GIT_SEQUENCE_EDITOR=/tmp/rebase-seq-editor.sh \
  git rebase -i -X ours $MERGE_BASE

# 7. After rebase: restore preserved working-tree files
cp /tmp/klang-unslop-preserve/prd.json scripts/ralph/ 2>/dev/null
# ... restore others

# 8. Pop or drop the stash
git stash pop 2>/dev/null || git stash drop  # drop if stash pop conflicts with restored files
```

#### exec commands in the rebase plan

Use `exec` lines to clean slop from mixed commits (pick + exec pattern):

```
pick c08409c feat: [US-183] - Fix AlignOperands
exec git rm tasks/prd-klang-format-parity-v2.md 2>/dev/null; git commit --amend --no-edit
```

To recreate a commit from scratch (drop + exec pattern):

```
drop c0fac69 chore: replace Beads - drop, recreate clean
exec git checkout c0fac69 -- .gitignore AGENTS.md docs/deviations.md && git rm -rf .beads/ && git commit -m "chore: remove Beads tracking, update AGENTS.md"
```

To revert a file to its parent's state inside exec (removes one commit's changes to a file):

```
exec git show HEAD~1:scripts/ralph/prd.json > scripts/ralph/prd.json && git add scripts/ralph/prd.json; git commit --amend --no-edit
```

### For mixed files (slop inside real changes):

**IMPORTANT**: `git add -p` requires interactive input. Instead, for mixed files:
1. Read the file
2. Create a cleaned version using Edit (remove sloppy comments/docs)
3. Stage and commit the clean version
4. After all commits, restore the original if needed

### Verification

After rewriting:

```bash
# Show new history
git log --oneline $BASE_BRANCH..HEAD

# Verify no real code was lost — diff should only show removed slop
git diff HEAD..$BACKUP_REF --stat

# Show what's left unstaged (slop left in worktree)
git status
```

Present the final state to the user.

## Safety Rules

- **ALWAYS save a backup ref** before any destructive operation
- **ALWAYS show the backup ref** to the user so they can restore
- **NEVER force-push** — the user decides when/if to push
- **NEVER delete slop files from worktree** without asking — just exclude from commits
- **ASK before proceeding** with the actual reset/rewrite — the proposal phase is mandatory
- If anything goes wrong: `git reset --hard $BACKUP_REF` restores everything

## Execution Pitfalls (Learned in Practice)

### 1. Conflict in tracking files when dropping progress-update commits (use `-X ours`)
When a branch has alternating `feat:` commits (that also touch `prd.json`, `progress.txt`) and `chore: update progress` commits (dropped), the picked feat commits' diffs for `prd.json` no longer apply cleanly because the dropped commits changed the base.
```
CONFLICT (content): Merge conflict in scripts/ralph/prd.json
```
**Fix:** Use `-X ours` strategy — it automatically accepts "ours" for all conflicts. Since only tracking files conflict (source files are linear), this is safe:
```bash
git rebase -i -X ours $MERGE_BASE
```

### 2. Working tree file preservation during rebase
Git rebase resets the working tree to the base commit. Files that only exist in dropped commits will disappear from the working tree. Files that were tracked (modified) and then dropped will revert to base state.

**Fix:** Before the rebase, manually copy files you want to preserve to `/tmp/`, then restore after:
```bash
# Before
mkdir -p /tmp/preserve && cp scripts/ralph/prd.json /tmp/preserve/
# After
cp /tmp/preserve/prd.json scripts/ralph/
```

### 3. `git stash pop` fails if you've already manually restored files
If you restore working-tree files manually before popping the stash, `git stash pop` will error: "untracked working tree files would be overwritten by merge."
**Fix:** Use `git stash drop` instead — the important files are already restored.

### 4. Verify all SHAs are in the plan before running
Double-check the plan counts:
```bash
# Count unique SHAs in plan == total commits on branch
grep -E '^(pick|drop)' /tmp/rebase-plan.txt | awk '{print $2}' | sort -u | wc -l
git log --format="%H" $MERGE_BASE..HEAD | wc -l
```
Any mismatch means a commit was omitted, which would cause the rebase to stop at an unexpected conflict.

### 5. `git log --oneline BASE..HEAD` vs merge-base
For long-running branches, `BASE..HEAD` shows commits reachable from HEAD but not from BASE, which may differ from what you expect if BASE has moved. Use `git merge-base HEAD BASE` to find the true divergence point.

## Edge Cases

- **Merge commits on branch**: use `git rebase --onto $BASE_BRANCH` first if needed, or ask user
- **Binary files**: include if they're real artifacts (icons, etc.), exclude if they're build outputs
- **Submodule changes**: always ask before including/excluding
- **CLAUDE.md / project config changes**: these are often legitimate — review content carefully, keep real config additions, remove AI-specific instructions that were added during development
