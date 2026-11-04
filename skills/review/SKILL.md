---
name: review
description: Quick project-aware review of the current diff (or named files) before a commit, using the reviewer agent and the project's `.claude/orchestrate.md` checklist and established patterns. Use when the user asks to review changes, "run the reviewer", or before committing work done outside the full pipeline. For multi-step work use `orchestrate:run`.
argument-hint: [files, commit range, or what to focus on]
---

# Quick review

Review target: $ARGUMENTS (if empty: staged changes, then unstaged and untracked changes).

1. Work out the target. If there is nothing to review, say so in one line and stop.
2. Pick the reviewer: the project's own `reviewer` agent if one exists, else `orchestrate:reviewer`.
3. Spawn it with: the repository root (absolute path), the exact target (files, or `git diff` / `git diff --staged` / a commit range), a one-paragraph summary of what the change is meant to do, and "attempt 1". If there is no plan, the DoD is: the stated intent is met and the profile's verify commands pass. If the diff spans more than about 15 files or more than one subsystem, spawn one reviewer per area in one message.
4. Relay the verdict and findings briefly. Offer to fix the blocking ones and the cheap "should fix" ones. Do not fix anything until the user agrees, unless they asked you to apply fixes.
5. If the reviewer keeps flagging something the project does on purpose, offer to add it to the profile's **Established patterns** so it is not flagged again.
