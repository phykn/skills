# push skill — merge-to-main workflow

## Goal

Extend the `push` skill so that running it from a feature branch integrates the work into `main` and cleans up the feature branch, without requiring the user to think about branching. Running it from `main` behaves as it does today.

## Motivation

The user's mental model is "my work goes on main." Feature branches are treated as throwaway scratch space. Today the skill pushes whatever branch you're on (`git push -u origin HEAD`), which leaves feature branches lingering locally and on the remote and forces manual cleanup.

## Behavior

### Case 1 — currently on `main`

Unchanged from today:

1. `git status`, `git diff` (staged + unstaged), `git log --oneline -5` to understand changes and match commit style.
2. `git add -A`.
3. Commit with a 1–2 sentence English message focused on *why*, ending with the `Co-Authored-By` trailer.
4. `git push -u origin HEAD`.
5. Report result.

### Case 2 — currently on a feature branch

1. `git status`, `git diff`, `git log --oneline -5` on the feature branch.
2. `git add -A` and commit on the feature branch with the standard message format and trailer. (Skip if there are no uncommitted changes.)
3. Remember the feature branch name.
4. `git checkout main`.
5. `git pull --ff-only origin main` to bring `main` up to date.
6. `git merge <feature-branch>` — fast-forward if possible, otherwise a regular merge commit.
7. `git push origin main`.
8. Delete the feature branch locally: `git branch -d <feature-branch>`.
9. Delete the feature branch on the remote if it exists there: `git push origin --delete <feature-branch>`.
10. Report result, including which branch was cleaned up.

### No-op case

If there are no changes *and* the current branch is `main`, stop and inform the user (same as today).

If there are no changes but the current branch is a feature branch, still proceed with the merge/cleanup flow — the user's intent is "get my work onto main and clean up," which applies even if the last commit is already made.

## Safety rules

The skill must stop and report (never retry blindly) if any of the following happen:

- `git checkout main` fails (e.g., would overwrite untracked files).
- `git pull --ff-only` fails because `origin/main` has diverged from local `main`.
- `git merge` produces conflicts.
- `git push origin main` is rejected.
- `git branch -d` refuses because the feature branch is not fully merged into `main` (this should not happen after a successful merge, but if it does, it indicates something went wrong — do not force with `-D`).

In every failure case: leave the repository in whatever state git left it, report clearly what failed and what state things are in, and let the user decide.

Never delete a branch that has not been successfully merged into `main`. Never force-push. Never use `-D` or `--force`.

## Non-goals

- No rebase, squash, or other history-rewriting merge strategies. Plain `git merge` only.
- No stash-based workflows. Commit first, then switch branches.
- No handling of multiple feature branches. The skill only touches the branch that was current when it was invoked.
- No interaction with PRs.

## Files changed

- `skills/push/SKILL.md` — rewrite the body to describe both cases and the safety rules. Keep the frontmatter (`allowed-tools`, `description`) as-is aside from adjustments the new body requires.
