# Refactor Skill Revision Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Revise `skills/refactor/SKILL.md` to introduce MECE partition as a co-equal diagnostic lens alongside Feynman reconstruction, and remove the duplicated self-check passages that have accumulated across the file.

**Architecture:** Section-by-section edits to a single prose file. No code, no tests. Each task replaces one contiguous passage with the approved replacement text from the design spec. Verification is structural (line count, grep for removed patterns, markdown table sanity).

**Tech Stack:** Markdown. No build, no runtime. The skill is loaded by the Claude Code harness as plain text.

**Source spec:** `docs/superpowers/specs/2026-04-11-refactor-skill-revision-design.md`

**Baseline:** `skills/refactor/SKILL.md` currently has uncommitted modifications already staged in the working tree (falsification / Step 4.5). Those are considered the starting point — this plan edits on top of them. Total current line count: 216.

**Target line count after all tasks:** ~150.

**Language rule:** The skill file is English-only (per existing feedback memory). All new prose in tasks below is in English. Commit messages may be in either language.

---

## File Structure

Only one file is modified:

- Modify: `skills/refactor/SKILL.md` (currently 216 lines, target ~150)

No files are created or deleted.

---

## Task 1: Rewrite Overview as two co-equal lenses

**Files:**
- Modify: `skills/refactor/SKILL.md:10-37`

This task replaces the current Overview + "Core Principle" + "Feynman self-check" + "Worked example" + "Mechanical tests" block (lines 10–37 of the current file) with a single consolidated Overview that introduces both lenses.

- [ ] **Step 1: Read the target range to confirm current content**

Use the Read tool on `skills/refactor/SKILL.md` lines 10–37. Verify the first line of the range starts with `## Overview` and the range ends with the "Mechanical tests" numbered list (item 3 about naming the invariant).

- [ ] **Step 2: Replace the range with the new Overview**

Use Edit with these exact strings.

`old_string` (the full range from `## Overview` through the end of "Mechanical tests" block):

```
## Overview

This skill **plans** a refactor by diagnosing a codebase. It never modifies the main workspace — falsification experiments (Step 4.5) run in throwaway git worktrees. Execution of the refactor itself is delegated to `writing-plans` → `executing-plans`.

The diagnostic method is **Feynman-style reconstruction**: for each unit, try to explain in plain language *why this unit should exist*. Do not fetch intent from docs, tests, or comments — reconstruct it. Where the reconstruction fails is the finding. All checks (structure/MECE, naming, logic, quality, wasteful work) reduce to this one question.

### The Feynman self-check (non-negotiable)

Every time you attempt a reconstruction, explicitly self-check:

> **"Is this a plain-English translation of what the code does, or an explanation of why it should exist? If the former, the reconstruction has failed — and the failure is itself the finding."**

The most common slip is treating code as self-justifying ("this function sorts the list, therefore it exists to sort the list"). If your explanation would still hold for a pointless function, you have paraphrased, not reconstructed.

### Worked example

Function `normalize_email(s)` lowercases and strips whitespace.

- **Paraphrase (failed reconstruction):** "This function exists to normalize email strings by lowercasing and stripping whitespace."
  *Why it fails:* This is a restatement of the body. It would hold for any normalization function. It says nothing about *why the normalization must happen*.

- **Real reconstruction:** "Downstream code compares emails for account-uniqueness via byte equality. Without this call, `Alice@x.com` and `alice@x.com` would create two accounts for the same person. The invariant: after this function returns, two inputs that should be the same user are byte-equal. Delete this → duplicate-account bug returns at the signup path."
  *Why it works:* Names the concrete invariant, explains what breaks if removed, ties to a specific downstream caller.

**Mechanical tests you can apply when unsure:**
1. "If I delete this code, what specifically breaks, and where?"
2. "Would my sentence still make sense if I swapped this function for an unrelated one with the same signature?" — if yes, it is paraphrase.
3. "Can I name the invariant this code enforces in one sentence?" — if no, the reconstruction has not happened yet.
```

`new_string`:

```
## Overview

This skill **plans** a refactor by diagnosing a codebase. It never modifies the main workspace — falsification experiments (Step 4.5) run in throwaway git worktrees. Execution of the refactor itself is delegated to `writing-plans` → `executing-plans`.

The skill diagnoses code through **two independent lenses**. Neither subsumes the other; they catch different kinds of findings, and you must run both.

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
```

- [ ] **Step 3: Verify self-check mentions are now down to 1 in the Overview region**

Use Grep on `skills/refactor/SKILL.md` with pattern `self-check|paraphrase` and `output_mode: "content"`. Expected: the only hits inside the Overview/Worked example region are the single self-check blockquote and the "paraphrased" line immediately after it. "Mechanical tests" and its three numbered items must be gone.

- [ ] **Step 4: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): rewrite Overview as two co-equal lenses"
```

---

## Task 2: Rename Step 3A passes to MECE lens → Feynman lens

**Files:**
- Modify: `skills/refactor/SKILL.md` — the Step 3A block

This task changes Step 3A so that Pass 1 is explicitly the MECE lens (main agent only, producing first-class MECE findings) and Pass 2 is the Feynman lens (parallel subagents). The selection filters are preserved; the structural shift is just framing Pass 1 as a findings source, not only a triage stage.

- [ ] **Step 1: Read the current Step 3A block**

Use the Read tool on `skills/refactor/SKILL.md`, locating `### Step 3A: Sweep mode`. Read through the end of the Step 3A block (stops at `### Step 3B`).

- [ ] **Step 2: Replace the Step 3A block**

Use Edit with these strings.

`old_string`:

```
### Step 3A: Sweep mode — shallow pass → deep pass

**Shallow pass (main agent):**
- `Glob` the tree, inspect headers and top-level structure
- Collect structural findings: folder/file naming, MECE violations, obvious layout issues
- Select deep-pass targets using **multiple filters, not just cosmetics**:
  - **Size** (lines of code)
  - **Complexity** (nesting depth, branch count, file length)
  - **Git churn** (files changed frequently recently — unstable = risk)
  - **Random sampling** — also pick a few apparently-clean files. Clean names can hide rotten internals; cosmetic triage alone misses these.

**Deep pass (parallel subagents):**
- Dispatch the selected targets to subagents in parallel (use `dispatching-parallel-agents`)
- Each subagent receives the dispatch template below and performs Feynman reconstruction
- Each returns a findings report
```

`new_string`:

```
### Step 3A: Sweep mode — MECE pass → Feynman pass

The two lenses have different scope requirements: MECE needs a global view, Feynman is local. So the MECE lens runs in the main agent (which sees the whole tree) and the Feynman lens parallelizes across subagents.

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
```

- [ ] **Step 3: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): Step 3A — MECE pass then Feynman pass"
```

---

## Task 3: Update Step 3B for two-lens targeted mode

**Files:**
- Modify: `skills/refactor/SKILL.md` — the Step 3B block

- [ ] **Step 1: Replace the Step 3B block**

Use Edit.

`old_string`:

```
### Step 3B: Targeted mode — direct deep pass

If the scope fits in the main agent's context, read the range directly and perform Feynman reconstruction yourself. Do not spawn subagents — overhead is not worth it for small scopes.
```

`new_string`:

```
### Step 3B: Targeted mode — both lenses, main agent

If the scope fits in the main agent's context, read the range directly and run both lenses yourself (MECE first, then Feynman). Do not spawn subagents — overhead is not worth it for small scopes, and running both lenses in one agent removes the global-view problem entirely.
```

- [ ] **Step 2: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): Step 3B runs both lenses in main agent"
```

---

## Task 4: Update subagent dispatch template

**Files:**
- Modify: `skills/refactor/SKILL.md` — the subagent dispatch template block

This task compresses the in-template self-check paragraph to a one-line reference, and adds a sentence scoping subagents to the Feynman lens only.

- [ ] **Step 1: Replace the dispatch template**

Use Edit.

`old_string`:

```
### Subagent dispatch template (for sweep deep pass)

When dispatching a subagent, paste this prompt with the bracketed parts substituted, and **append the Finding Format block (from the next section) verbatim** at the marked spot:

> You are performing a Feynman-style refactoring diagnosis of `[region path]`.
>
> For each significant unit (file, class, function), explain in plain language **why this unit should exist** — not what it does, why it has to exist.
>
> **Self-check every reconstruction:** Is this a plain-English translation of what the code does, or an explanation of why it should exist? If the former, the reconstruction has failed — and that failure IS the finding.
>
> Context signals (read as signals, not truth): `[paths to docs/README/tests]`
>
> Record each finding using the Finding Format below. Observation and Reconstruction MUST be separate fields — blending them destroys auditability. Do NOT propose implementation plans. Do NOT run experiments or modify code — the main agent will handle falsification in a later step. Do NOT group findings by single priority; use the three axes (Impact, Confidence, Effort).
>
> [APPEND FINDING FORMAT BLOCK HERE]
>
> Return a markdown report with all findings.
```

`new_string`:

```
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
```

- [ ] **Step 2: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): scope subagent dispatch to Feynman lens"
```

---

## Task 5: Update Finding Format Category (group by lens, add `gap`)

**Files:**
- Modify: `skills/refactor/SKILL.md` — the Finding Format block

- [ ] **Step 1: Replace the Category line**

Use Edit.

`old_string`:

```
- **Category**: structure | naming | logic | complexity | dead-code | duplication | perf-waste
```

`new_string`:

```
- **Category**: one of the two lens groups:
  - Feynman lens: `dead-code | logic | complexity | perf-waste`
  - MECE lens: `structure | naming | duplication | gap`
```

- [ ] **Step 2: Verify `gap` appears exactly where expected**

Use Grep with pattern `\bgap\b` on `skills/refactor/SKILL.md`, `output_mode: "content"`. Expected hits: Overview (Lens ② bullet), Step 3A Pass 1 categories list, Finding Format Category line, Step 4.5 skip list (added in Task 6), Failure Modes row (added in Task 7). Before Task 6/7 run, only the first three are expected.

- [ ] **Step 3: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): group Finding categories by lens, add gap"
```

---

## Task 6: Update Step 4.5 — skip list, drop "Who runs experiments", compress prose

**Files:**
- Modify: `skills/refactor/SKILL.md` — the Step 4.5 block

This task does three things in one Edit because the affected passages are contiguous.

- [ ] **Step 1: Replace the Step 4.5 prose tail**

Use Edit.

`old_string`:

```
**Update the finding based on the result:**
- Experiment confirms → Confidence: high, attach `Verification` field
- Experiment contradicts → either drop the finding or rewrite it as a new finding ("I thought X but the test shows Y")
- Cannot write an experiment (e.g. falsifier not expressible) → Confidence stays where the reconstruction left it, mark `Verification: not falsifiable by execution`

**Skip this step for** `structure | naming | complexity` — these cannot be tested by execution. Rely on reconstruction alone.

**Who runs experiments:** The main agent only. Subagents in Step 3A perform reconstruction and return findings; they do NOT run experiments. Centralizing experiments in the main agent avoids worktree contention, makes the experiment budget (see below) enforceable, and keeps subagents cheap.

**Experiment budget:** Run at most 10 experiments per invocation by default. If more falsifiable findings exist, rank by `Impact × Confidence` and experiment on the top 10. Mark the rest with `Verification: skipped (budget)`.

**Per-experiment attempt cap: 3.** Each experiment (write the falsifier, run the test) gets at most 3 attempts. If it is not producing a clean signal within 3 tries — the test setup is broken, the environment is uncooperative, the falsifier keeps needing rewrites — stop and mark the finding `Verification: inconclusive after 3 attempts`. Do NOT fall into a debugging rabbit hole trying to make one experiment work; move on to the next finding. The goal of this step is signal, not exhaustive verification.

**Guardrail: coverage check.** Before trusting a `dead-code` experiment, confirm the file is actually exercised by the test suite. A "dead code" finding on an uncovered file is meaningless — downgrade Confidence to low and mark `Verification: uncovered by tests`.
```

`new_string`:

```
**Update the finding based on the result:**
- Confirms → Confidence: high, attach `Verification` field with command + output
- Contradicts → drop the finding, or rewrite it as a new finding ("I thought X but the test shows Y")
- Not expressible as an experiment → keep reconstruction's Confidence, mark `Verification: not falsifiable by execution`

**Skip Step 4.5 entirely for** `structure | naming | complexity | gap` — these cannot be falsified by execution. (Experiments are a main-agent step; subagents never run them, per Step 3A.)

**Budget:** At most 10 experiments per invocation. Rank falsifiable findings by `Impact × Confidence` and experiment on the top 10. Mark the rest `Verification: skipped (budget)`.

**Per-experiment attempt cap: 3.** If a single experiment isn't producing a clean signal within 3 tries, stop and mark `Verification: inconclusive after 3 attempts`. Do not debug-rabbit-hole one experiment — move on. Goal is signal, not exhaustive verification.

**Coverage guardrail.** Before trusting a `dead-code` experiment, confirm the file is actually exercised by the test suite. A "dead code" finding on an uncovered file is meaningless — downgrade Confidence to low and mark `Verification: uncovered by tests`.
```

- [ ] **Step 2: Verify "Who runs experiments" no longer appears**

Use Grep with pattern `Who runs experiments` on `skills/refactor/SKILL.md`. Expected: no matches.

- [ ] **Step 3: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): compress Step 4.5, add gap to skip list"
```

---

## Task 7: Update Failure Modes table

**Files:**
- Modify: `skills/refactor/SKILL.md` — the Failure Modes table

- [ ] **Step 1: Replace the Failure Modes table**

Use Edit.

`old_string`:

```
| Symptom | Why it is wrong |
|---------|-----------------|
| "Observation" and "Reconstruction" contain the same information | They must be separable: one is fact, one is interpretation. Blended, neither you nor the user can audit the reasoning. |
| A finding written as a prose paragraph instead of the named fields | Prose blends observation and interpretation. The named fields exist to force them apart. |
| "The code is self-explanatory; no reconstruction needed" | Then the reconstruction is trivial — write it out. "Self-explanatory" usually means "I paraphrased in my head and moved on". |
```

`new_string`:

```
| Symptom | Why it is wrong |
|---------|-----------------|
| "Observation" and "Reconstruction" contain the same information | They must be separable: one is fact, one is interpretation. Blended, neither you nor the user can audit the reasoning. |
| A finding written as a prose paragraph instead of the named fields | Prose blends observation and interpretation. The named fields exist to force them apart. |
| After a sweep, zero `structure`, `duplication`, or `gap` findings | The MECE lens was skipped. A real sweep-scoped codebase almost always has at least one boundary issue — zero MECE findings usually means the main agent only ran Feynman. |
```

- [ ] **Step 2: Commit**

```bash
git add skills/refactor/SKILL.md
git commit -m "refactor(refactor-skill): Failure Modes — add MECE row, drop self-explanatory"
```

---

## Task 8: Final verification

**Files:**
- Read-only: `skills/refactor/SKILL.md`

- [ ] **Step 1: Line count check**

Run:

```bash
wc -l skills/refactor/SKILL.md
```

Expected: between 140 and 165 lines. If outside that range, investigate (did a replacement drop too much, or leave stale content?). Target is ~150.

- [ ] **Step 2: Self-check mention count**

Use Grep with pattern `self-check` on `skills/refactor/SKILL.md`, `output_mode: "count"`. Expected: 2 or fewer occurrences (one in the Overview self-check block, optionally one in the subagent template reference). Absolutely not 5 as in the original.

- [ ] **Step 3: Category value sanity**

Use Grep with pattern `dead-code \| logic` and `structure \| naming \| duplication \| gap` on `skills/refactor/SKILL.md`, `output_mode: "content"`. Expected: both patterns appear — the first from the subagent template and Finding Format Feynman group, the second from the Overview Lens ② bullet and Finding Format MECE group.

- [ ] **Step 4: Deleted-content sanity**

Use Grep with pattern `Mechanical tests you can apply|Who runs experiments|self-explanatory` on `skills/refactor/SKILL.md`. Expected: zero matches. Any match means a deletion was missed.

- [ ] **Step 5: Report back**

Report: final line count, final self-check count, any anomalies. No commit — this task is read-only verification.

---

## Self-review complete

Spec coverage: every bullet of the spec's "설계" section maps to a task (Overview → T1; Step 3A → T2; Step 3B → T3; subagent template → T4; Finding Format → T5; Step 4.5 → T6; Failure Modes → T7; dedup targets are distributed across T1/T4/T6/T7). Success criteria are verified by T8.

No placeholders. Every edit shows full old_string and new_string. Category values are consistent across tasks (`dead-code | logic | complexity | perf-waste` / `structure | naming | duplication | gap`).
