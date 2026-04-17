---
name: refactor
description: Use when the user asks to refactor, clean up, reorganize, or improve code without a specific bug in mind.
disable-model-invocation: false
allowed-tools: Bash Read Grep Glob Write Edit Agent
---

# Refactor

"Fix my code." Ask ≤2 upfront questions (scope, test command — only if undetectable). Then diagnose, auto-triage, apply, test, commit. No further questions. Flags: `dry-run` (stop after triage), `allow-red-baseline`, `include-behavior-change`.

## Axes

| Axis | Values | Meaning |
|---|---|---|
| Category | `delete` \| `extract` \| `inline` \| `move` \| `rename` \| `replace` \| `correct` \| `add` | Operation |
| Lens | Unit \| Partition | Per-unit vs cross-unit |
| Stage | 1 \| 2 | Filesystem-path change = 1, else 2 |

`delete`/`extract`/`inline`/`move`/`rename`/`replace` preserve behavior. `correct` (invariant fix) and `add` (missing abstraction) change behavior — both require a new test proving the invariant before the fix.

Same smell → different operations: duplication → `delete` (drop one copy) or `extract` (pull the common piece). The choice is the finding.

## Lenses

- **Unit** — hold one unit in working memory, reconstruct *why it should exist*. Parallelizable.
- **Partition** — hold sibling units in working memory, check they tile the space *without overlap or gap*. Main agent only.

Reconstruct intent from code alone — **do not read docs, tests, or comments for intent**. Reconstruction failures are the findings.

**Self-check:** If your explanation still holds for a pointless function or partition, you paraphrased. The paraphrase is the finding.

## Worked example

`src/ml/data/{augment,preprocess}/`.

- *Paraphrase:* "`augment/` augments, `preprocess/` preprocesses" — restates names, fails.
- *Reconstruction:* "`preprocess/` = deterministic one-shot transforms; `augment/` = stochastic runtime perturbations." Check: `preprocess/normalize.py` has random jitter; `augment/crop.py` is deterministic → split is by name, not responsibility → `move` (Partition, Stage 1).

## When NOT to Use

- Specific bug fixing.
- Profile-driven performance optimization.
- Hardware/library-specific optimization (SIMD, CUDA, cache alignment).
- New features.

Exception: "unnecessary work" surfaces as `delete`/`replace` during Unit reconstruction — in scope.

## Process

### 1. Setup

Infer scope (`/refactor src/ml/training` = targeted; no path = sweep). Pick tier, state in one line:

| Tier | Trigger | Lenses | Subagents | Experiments |
|---|---|---|---|---|
| `light` | ≤3 files or <500 LOC | Unit only | no | 2 |
| `standard` | 4–15 files, or sweep <50 files | both | no | 3 |
| `deep` | sweep ≥50 files, or user says "deep" | both | parallel | 10 |

Default: lowest tier scope allows.

Read `CLAUDE.md`, `README.md`, `docs/`, `git log`, test locations as *signals* — not intent. Discrepancies are findings.

Detect the test command (`package.json`, `pyproject.toml`, `Cargo.toml`, `Makefile`, `pytest.ini`). **Run baseline — must be green.** Else abort with first-failure summary; user re-invokes with `allow-red-baseline`.

Ask scope only if neither path nor sweep intent is inferable. Ask test command only if none detectable. Bundle in one message. After this, ask nothing more.

### 2. Diagnose + verify

Partition lens first (global, main agent). Then Unit:
- `deep`: dispatch Unit targets to parallel subagents.
- `standard`: Unit sequentially in main agent.
- Targeted: both in main agent.

Unit target selection: file size, nesting depth, git churn, plus random sampling (clean names hide rotten internals).

Each finding = exactly one operation. Track internally: location, lens, category, stage, one-line observation, one-line reconstruction-failure-point, direction, Impact/Confidence/Effort, verification, and (for `correct`/`add`) pre-commit test name.

Confirm falsifiable operations in a throwaway worktree:

| Category | Experiment |
|---|---|
| `delete` | Delete, run tests. Pass → safe. Require test coverage — else downgrade Confidence. |
| `extract` (duplication) | Replace one copy with a call to the other. Pass → equivalent. |
| `inline` | Replace callers with body. Pass → no dependency on separation. |
| `correct` | Write a test violating the claimed invariant. Fail → real; Pass → drop the finding. |
| `replace` (perf claim) | Benchmark at n=10/100/1000. |

Skip experiments for `move`/`rename`/`add` and `replace` without perf claim.

Budget per tier (see table). Rank by `Impact × Confidence`, top N. Cap 3 attempts per experiment. Discard worktree after each.

### 3. Triage + apply

Three buckets (automatic):

- **Include**: behavior-preserving findings above threshold.
- **Report**: `correct`/`add` — behavior change, listed not applied. `include-behavior-change` flag moves these to Include; each gets its invariant test applied first, fix commits only if that test goes red → green.
- **Skip** (silent): Impact low AND Confidence low, OR `Verification: contradicted`, OR uncovered by tests.

Conflicts: keep higher `Impact × Confidence`. Tie: keep Stage 1.

Headline (user-facing — use plain words, not internal bucket names):

```
Scope: <x>. Tier: <t>.
Fixing now: <N> (<s1> file moves, <s2> code edits). Flagged (would change behavior, not touching): <B>. Ignored (low confidence or not covered by tests): <M>.
Starting.
```

Include and Report both empty → "Nothing worth fixing.", terminate. `dry-run` → stop here.

**Stage 1** (filesystem moves/renames): batch atomically. Full suite. Green → commit. Red → revert batch, mark `reverted`, stop.

**Stage 2**, per finding in decreasing Impact order (Include only):

1. Apply.
2. Test touched region. Green → commit (`refactor(<scope>): <summary> (F-xx)`).
3. Red → one retry: revert, reshape, re-run. Green → commit; Red → revert, mark `reverted (2 attempts)`, continue.

**Final full suite.** Green → finish. Red → flag the cross-cutting regression with the failing test name. Do NOT auto-revert; user decides.

**Final report (user-facing — use plain words):**

```
Fixed: <K> of <N>, in <C> commits. Had to undo: <R> (<ids>) — tests broke after the change.
Tests: were green before → <still green ✓ | now red at <first-failure>. A regression slipped past per-file tests — inspect or revert via git.>
Flagged but NOT fixed: <B> (<ids>) — these would change behavior, so left alone. Rerun with `include-behavior-change` to apply.
Skipped as low priority: <M>.
```

Omit the "Flagged" line when B = 0. Omit the "Had to undo" clause when R = 0.

## Failure Modes

| Symptom | Fix |
|---|---|
| Observation and Reconstruction say the same thing | Split: fact vs interpretation. |
| Category recorded as a smell name (`dead-code`, `duplication`, `logic`, `gap`, `structure`, `naming`, `complexity`, `perf-waste`) | Reclassify: `dead-code`→`delete`; `duplication`→`delete`/`extract`; `logic`→`correct`; `gap`→`add`; `structure`→`move`; `naming`→`rename`; `complexity`→`extract`/`replace`; `perf-waste`→`delete`/`replace`. |
| After a sweep, zero Partition findings | Partition was skipped. Re-run. |
| After 10+ units scanned, zero Unit findings | Unit applied cosmetically. Re-apply self-check. |
