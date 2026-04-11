---
name: refactor
description: Use when the user asks to refactor, clean up, reorganize, or improve code without a specific bug in mind. Plans and scopes a refactor via Feynman-style reconstruction — does not execute the refactor itself; hands off to writing-plans.
disable-model-invocation: false
allowed-tools: Bash Read Grep Glob Write Edit Agent
---

# Refactor (Diagnose and Plan)

The main workspace is never modified — falsification experiments (Step 4.5) run in throwaway git worktrees. Execution hands off to `writing-plans` → `executing-plans`.

Diagnosis runs through **two independent lenses**. Run both.

**Lens ① — Feynman reconstruction (per-unit justification).**
Question: *Why should this unit exist?*
Catches: units with no reason (`dead-code`), unjustified complexity, `logic` that fails to enforce its claimed invariant, work with no purpose (`perf-waste`).

**Lens ② — MECE partition (cross-unit structure).**
Question: *Do these units divide the problem space without overlap or gap?*
Applies at every granularity: folders, files, functions.
Catches: unclear module boundaries (`structure`), names that do not reflect the partition (`naming`), the same concern in two cells (`duplication` = overlap), a concern with no cell (`gap`).

Both lenses share one rule: **do not fetch intent from docs, tests, or comments. Reconstruct it.** The points where reconstruction fails are the findings.

**Self-check (apply to every reconstruction in either lens):**

> "Is this a plain-English translation of what the code does, or an explanation of why it should exist / why this partition? If the former, the reconstruction has failed — and the failure is itself the finding."

The most common slip is treating code as self-justifying ("this function sorts the list, therefore it exists to sort the list"). If your explanation would still hold for a pointless function, you have paraphrased, not reconstructed.

### Worked example (Feynman lens)

Function `normalize_email(s)` lowercases and strips whitespace.

- **Paraphrase (failed reconstruction):** "This function exists to normalize email strings by lowercasing and stripping whitespace." *Why it fails:* a restatement of the body. It would hold for any normalization function.

- **Real reconstruction:** "Downstream code compares emails for account-uniqueness via byte equality. Without this call, `Alice@x.com` and `alice@x.com` would create two accounts for the same person. The invariant: after this function returns, two inputs that should be the same user are byte-equal. Delete this → duplicate-account bug returns at the signup path." *Why it works:* names the concrete invariant, explains what breaks if removed, ties to a specific downstream caller.

## When to Use

- User asks to refactor, clean up, reorganize, or improve a codebase or module
- User asks for a structural/quality code review (not a specific bug)
- User says "this feels messy", "this needs cleanup", "let us plan a refactor"

## When NOT to Use

- **Specific bug fixing** — use `systematic-debugging`
- **Profile-driven performance optimization** — benchmark/profiler work is a different workflow
- **Hardware- or library-specific optimization** (SIMD, CUDA tuning, cache alignment)
- **New features** — use `brainstorming`

**Performance scope boundary**: "this code does unnecessary work" or "this algorithm is needlessly expensive for its purpose" IS in scope — those fall out of Feynman reconstruction naturally. "This hot path is slow under load" is out of scope — it needs measurement.

## Process

### Step 1: Determine scope

Infer from the invocation whether this is a **sweep** (full codebase) or a **targeted** refactor. If the user wrote `/refactor src/ml/training`, that is targeted — do not ask again. Ask ONLY when genuinely ambiguous.

### Step 2: Gather context signals

Read each of these as *signals*, not ground truth:
- `CLAUDE.md`, `README.md`, `docs/`
- Recent `git log` (progression + churn data for triage)
- Test file locations and a rough coverage signal

**Do NOT ask the user for intent at this stage.** Asking up-front invites post-hoc rationalization, which corrupts the reconstruction signal. You ask the user later, and only when a specific reconstruction conflict surfaces.

Discrepancies between signals (docs vs code, tests vs code, commits vs code) are themselves findings.

### Step 3A: Sweep mode — MECE pass → Feynman pass

MECE needs a global view (main agent only); Feynman is local (parallelizes across subagents).

**Pass 1 — MECE lens (main agent only):**
- `Glob` the tree, inspect folder/file naming and module boundaries
- Produce MECE findings directly, not just triage signals. Categories: `structure | naming | duplication | gap`
- Produce the Feynman-pass target list using the filters below, **plus** any files the MECE pass flagged as sitting in a suspicious boundary region:
  - **Size** (lines of code)
  - **Complexity** (nesting depth, branch count, file length)
  - **Git churn** (files changed frequently recently — unstable = risk)
  - **Random sampling** — also pick a few apparently-clean files. Clean names can hide rotten internals; cosmetic triage alone misses these.

**Pass 2 — Feynman lens (parallel subagents):**
- Dispatch the selected targets to subagents in parallel (use `dispatching-parallel-agents`)
- Each subagent receives the dispatch template below and performs Feynman reconstruction on its slice
- Each returns a findings report containing only Feynman-lens categories (`dead-code | logic | complexity | perf-waste`). Subagents do NOT produce MECE findings — they lack the global view.

### Step 3B: Targeted mode — both lenses, main agent

If the scope fits in the main agent's context, read the range directly and run both lenses yourself (MECE first, then Feynman). Do not spawn subagents — overhead is not worth it for small scopes.

### Step 4: Aggregate findings

- Merge subagent reports (sweep mode)
- Deduplicate overlapping findings
- Assign three independent axes to each finding:
  - **Impact** (low / med / high) — how much code or behavior is affected
  - **Confidence** (low / med / high) — how sure you are this is a real problem
  - **Effort** (S / M / L) — rough cost to fix
- **Do NOT compute a single priority score** — user weights axes differently; one number erases the distinction.
- Detect **conflicts**: findings whose suggested directions are mutually exclusive. Mark them explicitly so the user knows to pick at most one.

### Step 4.5: Falsify findings with real experiments

For findings in these categories, **do not trust your own reconstruction — run an experiment**:

| Category | Experiment |
|---|---|
| `dead-code` | Delete the code, run the relevant tests. Pass → dead confirmed. |
| `duplication` | Replace one implementation with a call to the other, run tests. Pass → duplication confirmed. |
| `unused-return` | Change the function to return `None` (or equivalent), run tests. Pass → return value unused. |
| `logic` (when reconstruction claims "this enforces X") | Write a test that violates X. Fail → claim confirmed. Pass → reconstruction is wrong; this itself becomes a new finding. |
| `perf-waste` (algorithmic) | Benchmark with n=10, 100, 1000 to confirm the complexity claim. |

**How to run experiments safely:**

1. Create an isolated git worktree (use `using-git-worktrees`) so the main workspace is never touched.
2. Make the modification in the worktree.
3. Run only the relevant tests (`pytest path/to/test_x.py`, not the full suite), detected from Step 2's context scan.
4. Record the command and output.
5. Discard the worktree.

**Update the finding based on the result:**
- Confirms → Confidence: high, attach `Verification` field with command + output
- Contradicts → drop the finding, or rewrite it as a new finding ("I thought X but the test shows Y")
- Not expressible as an experiment → keep reconstruction's Confidence, mark `Verification: not falsifiable by execution`

**Skip Step 4.5 entirely for** `structure | naming | complexity | gap` — these cannot be falsified by execution. (Experiments are a main-agent step; subagents never run them, per Step 3A.)

**Budget:** At most 10 experiments per invocation. Rank falsifiable findings by `Impact × Confidence` and experiment on the top 10. Mark the rest `Verification: skipped (budget)`.

**Per-experiment attempt cap: 3.** If one experiment isn't producing a clean signal within 3 tries, stop and mark `Verification: inconclusive after 3 attempts`. Goal is signal, not exhaustive verification.

**Coverage guardrail.** Before trusting a `dead-code` experiment, confirm the file is actually exercised by the test suite. A "dead code" finding on an uncovered file is meaningless — downgrade Confidence to low and mark `Verification: uncovered by tests`.

### Step 5: Present findings → user selection

See "Presentation and Selection" below.

### Step 6: Write the spec document

If the user selects nothing, terminate cleanly.

Otherwise, write to `docs/superpowers/specs/YYYY-MM-DD-refactor-<scope>-design.md` in the target project (create the directory if needed, NOT in `~/.claude/`) with these contents:

1. **Scope** — what was diagnosed
2. **Context summary** — what sources were read
3. **Full body of each selected finding**
4. **Refactoring constraints** — tests must still pass, public API stable, etc.
5. **Success criteria** — "after the refactor, the same Feynman reconstruction should no longer get stuck at the failure point"

Commit with a message like `docs: refactor diagnosis for <scope>`, then invoke `writing-plans` with the spec as input.

### Subagent dispatch template (for Feynman pass)

When dispatching a subagent, paste this prompt with the bracketed parts substituted, and **append the Finding Format block (from the next section) verbatim** at the marked spot:

> You run the **Feynman lens only** on `[region path]`. MECE analysis (structure, boundaries, overlap, gap) requires a global view and is done by the main agent — do NOT write findings about module boundaries or files outside your scope.
>
> For each significant unit (file, class, function), explain in plain language **why this unit should exist** — not what it does, why it has to exist. Apply the self-check from the skill Overview to every reconstruction.
>
> Context signals (read as signals, not truth): `[paths to docs/README/tests]`
>
> Record each finding using the Finding Format below. Observation and Reconstruction MUST be separate fields — blending them destroys auditability. Do NOT propose implementation plans. Do NOT run experiments or modify code — the main agent handles falsification in a later step. Use the three axes (Impact, Confidence, Effort); do not group findings by single priority.
>
> Valid Category values for your output: `dead-code | logic | complexity | perf-waste`.
>
> [APPEND FINDING FORMAT BLOCK HERE]
>
> Return a markdown report with all findings.

## Finding Format

Each finding has this exact structure:

```markdown
### F-{id}: {one-line summary}
- **Location**: `path/to/file.py:42` (or directory for structural findings)
- **Category**: one of the two lens groups:
  - Feynman lens: `dead-code | logic | complexity | perf-waste`
  - MECE lens: `structure | naming | duplication | gap`
- **Observation**: What is visible in the code. Facts only, NO interpretation. Example: "This function is called from three places, all of which discard its return value."
- **Reconstruction attempt**: "This unit exists to do X, because Y..." — a plain-language why-it-should-exist.
- **Failure point**: Where and why the reconstruction broke down. THIS is the reason it is a finding.
- **Suggested direction**: One or two sentences of direction. NOT an implementation. Detailed planning is deferred to `writing-plans`.
- **Axes**: Impact: low|med|high, Confidence: low|med|high, Effort: S|M|L
- **Verification** *(optional)*: Experiment command + observed result (see Step 4.5). Or `not falsifiable by execution`.
- **Conflicts** *(optional)*: `F-05 proposes the opposite direction — mutually exclusive`
```

## Presentation and Selection

First, show a summary table for scanning:

```
ID    | Location                          | Category  | Imp  | Conf | Eff | Summary
F-01  | src/ml/training/trainer.py        | logic     | high | high | M   | loss schedule diverges from config
F-02  | src/ml/data/                      | structure | med  | high | S   | augment/ and preprocess/ boundary unclear
F-03  | src/ml/model/lora/wrapper.py:88   | dead-code | low  | high | S   | unused fallback branch
```

Then the user requests details or makes a selection using:
- **ID list**: "show F-01, F-03" / "F-01, F-03, F-07"
- **Filter**: "all high-impact", "all S-effort", "all logic"
- **Everything**: "show all"
- **Empty selection**: valid — user may look and decide to fix nothing. Terminate cleanly (see Step 6).

## Failure Modes — STOP and Restart

In-flight checklist. If any of these is true of a finding you just wrote, the reconstruction slipped — stop and redo.

| Symptom | Why it is wrong |
|---------|-----------------|
| "Observation" and "Reconstruction" contain the same information | They must be separable: one is fact, one is interpretation. Blended, neither you nor the user can audit the reasoning. |
| A finding written as a prose paragraph instead of the named fields | Prose blends observation and interpretation. The named fields exist to force them apart. |
| After a sweep, zero `structure`, `duplication`, or `gap` findings | The MECE lens was skipped. A real sweep-scoped codebase almost always has at least one boundary issue — zero MECE findings usually means the main agent only ran Feynman. |
