---
name: commit-message
description: Writes a clear Git commit message from the current changes. Use when the user asks for a commit message, or wants the staged or unstaged changes summarized before committing.
---

# Commit message

Write a commit message for the user's current changes.

## Steps
1. Run `git status --short` with run_command to see what changed.
2. Run `git diff --staged --stat`. If nothing is staged, run `git diff --stat` instead.
3. Read the actual changes with `git diff --staged | Select-Object -First 400` (or `git diff | Select-Object -First 400` when nothing is staged). If the diff is long, look at the most important files first.
4. Write the message in this format:
   - Subject line: imperative mood, at most 72 characters, no trailing period.
   - A blank line.
   - Body: short bullet points that explain what changed and why, not how.
5. Reply in the same language the user is writing in.

## Rules
- Do not run `git commit`, `git add` or any command that changes the repository unless the user explicitly asks for it.
- Mention anything suspicious you noticed (debug code, secrets, unrelated changes) after the message.
