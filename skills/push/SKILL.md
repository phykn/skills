---
name: push
description: Stage all changes, auto-generate a commit message, and push to remote
disable-model-invocation: false
allowed-tools: Bash(git *) Read Grep
---

Push all changes to the remote repository:

1. Run `git status` and `git diff` (staged + unstaged) to understand the changes.
2. Run `git log --oneline -5` to match the repo's commit message style.
3. Stage all changed files with `git add -A`.
4. Generate a concise commit message summarizing the changes (1-2 sentences, focus on "why").
   - Always write the commit message in **English**.
   - Keep it short and concise.
   - End the message with: `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
5. Commit with the generated message.
6. Push to the current branch's remote tracking branch (or `origin HEAD` if none is set).
7. Report the result to the user.

If there are no changes to commit, inform the user and stop.
If the commit or push fails, diagnose the error and report it — do NOT retry blindly.
