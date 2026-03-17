# git-unslop skill

Claude Code skill for cleaning up messy feature branches before code review.

## What it does

Takes a feature branch full of AI-generated slop (progress trackers, junk docs, obvious comments, agent artifacts) and disorganized commits, and rewrites it into a clean, review-friendly commit history matching the target repo's conventions.

## Installation

`git clone <repo-url> ~/.claude/skills/git-unslop`

## Usage

Invoke directly or describe what you need:

```
/git-unslop
```

```
clean up this branch for review
```

```
prepare branch for PR, rewrite commit history
```

The skill will:
1. Identify the base branch
2. Classify files as real code vs slop
3. Propose a clean commit structure for your approval
4. Execute the rewrite after confirmation

## Limitations

- **Destructive operation** — always creates a backup ref before rewriting, but review the plan carefully
- **No force-push** — you decide when and where to push
- Branches with merge commits may need manual intervention
