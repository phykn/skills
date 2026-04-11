---
name: push
description: Stage all changes, auto-generate a commit message, and push to remote
disable-model-invocation: false
allowed-tools: Bash(git:*), Read, Grep
---

Push the user's work to `main`. The flow branches on whether you're already on `main` or on a feature branch.

## Step 1 — Understand current state

Run these in parallel to see what you're working with:

- `git status`
- `git diff` (shows unstaged)
- `git diff --staged`
- `git log --oneline -5`
- `git branch --show-current`

Match the repo's commit style from recent log entries.

## Step 2 — Branch on current branch

### Case A: already on `main`

1. If there are no changes (nothing staged, nothing unstaged, nothing untracked worth committing), stop and tell the user there is nothing to push.
2. `git add -A`.
3. Commit with a 1–2 sentence message in **English** focused on *why*, ending with:
   `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
4. `git push -u origin HEAD`.
5. Report the result. Done.

### Case B: on a feature branch (anything other than `main`)

Remember the current branch name — call it `$FEATURE`.

1. **Commit any pending work on `$FEATURE`.** If `git status` shows staged or unstaged changes:
   - `git add -A`
   - Commit with the same 1–2 sentence English message format and `Co-Authored-By` trailer as Case A.
   If there are no pending changes, skip this step.

2. **Switch to `main`.**
   - `git checkout main`
   - If this fails (e.g., would overwrite untracked files), **stop** and report the error. Do not retry, do not force.

3. **Update `main` from the remote.**
   - `git pull --ff-only origin main`
   - If this fails because `origin/main` has diverged from local `main`, **stop** and report. Let the user resolve it.

4. **Merge `$FEATURE` into `main`.**
   - `git merge $FEATURE`
   - Fast-forward is fine; a regular merge commit is fine. Do **not** use `--squash`, `--rebase`, or `--no-ff`.
   - If merge produces conflicts, **stop** and report the conflicted files. Leave the repo in the conflicted state for the user.

5. **Push `main`.**
   - `git push origin main`
   - If the push is rejected, **stop** and report. Do not force-push.

6. **Delete `$FEATURE` locally.**
   - `git branch -d $FEATURE`
   - This uses lowercase `-d`, which refuses if the branch is not fully merged. After a successful merge in step 4 it should always succeed. If it refuses, **stop** and report — do **not** escalate to `-D`.

7. **Delete `$FEATURE` on the remote if it exists there.**
   - Check: `git ls-remote --heads origin $FEATURE`
   - If the output is non-empty: `git push origin --delete $FEATURE`
   - If the output is empty, skip — the branch was never pushed.

8. Report the result, including: what was committed, that `main` was pushed, and that `$FEATURE` was cleaned up (locally and/or remotely).

## Safety rules (apply to both cases)

- Never force-push. Never use `git push --force` or `--force-with-lease`.
- Never use `git branch -D` or `git reset --hard` to work around a failure.
- Never delete a branch that has not been successfully merged into `main`.
- On any failure, stop and report the exact command that failed and what state the repo is in. Do not retry blindly.
- Do not skip hooks (`--no-verify`). If a pre-commit hook fails, fix the underlying issue or report it.
