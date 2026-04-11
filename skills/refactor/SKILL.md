---
name: refactor
description: Use when the user asks to refactor, clean up, reorganize, or improve code without a specific bug in mind. Plans and scopes a refactor via Feynman-style reconstruction — does not execute the refactor itself; hands off to writing-plans.
disable-model-invocation: false
allowed-tools: Bash(git log:*) Bash(git status:*) Bash(find:*) Bash(wc:*) Read Grep Glob Write Edit Agent
---

# Refactor (Diagnose and Plan)

## Overview

This skill **plans** a refactor by diagnosing a codebase. It does NOT modify code. Execution is delegated to `writing-plans` → `executing-plans`.

The diagnostic method is **Feynman-style reconstruction**: for each unit, try to explain in plain language *why this unit should exist*. Where that explanation fails — where you can only describe what the code does, not why it should exist — that is the finding.

## Core Principle

**Do not fetch intent from any source. Reconstruct it. The points where reconstruction fails in plain language are the findings.**

All checks (structure/MECE, naming, logical soundness, quality, wasteful work) reduce to one question:

> *Can this unit be explained in plain language in terms of why it should exist?*

### The Feynman self-check (non-negotiable)

Every time you attempt a reconstruction, you MUST explicitly self-check:

> **"Is this explanation a plain-English translation of what the code does, or is it an explanation of why this code should exist? If it is the former, the reconstruction has failed — and the failure is itself the finding."**

This is the single most important guardrail. The most common slip is treating code as self-justifying ("this function sorts the list, therefore it exists to sort the list"). That is paraphrase, not reconstruction. If your explanation would still hold for a completely pointless function, you have paraphrased.

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

### Step 3A: Sweep mode — shallow pass → deep pass

**Shallow pass (main agent):**
- `Glob` the tree, inspect headers and top-level structure
- Collect structural findings: folder/file naming, MECE violations, obvious layout issues
- Select deep-pass targets using **multiple filters, not just cosmetics**:
  - **Size** (lines of code)
  - **Complexity** (nesting depth, branch count, file length)
  - **Git churn** (files changed frequently recently — unstable = risk)
  - **Random sampling** — pick a few apparently-clean files too. This is critical: a file with clean names and a clean layout can hide rotten internals. Pure cosmetic triage would miss those, so sampling plugs the hole.

**Deep pass (parallel subagents):**
- Dispatch the selected targets to subagents in parallel (use `dispatching-parallel-agents`)
- Each subagent receives the dispatch template below and performs Feynman reconstruction
- Each returns a findings report

### Step 3B: Targeted mode — direct deep pass

If the scope fits in the main agent's context, read the range directly and perform Feynman reconstruction yourself. Do not spawn subagents — overhead is not worth it for small scopes.

### Step 4: Aggregate findings

- Merge subagent reports (sweep mode)
- Deduplicate overlapping findings
- Assign three independent axes to each finding:
  - **Impact** (low / med / high) — how much code or behavior is affected
  - **Confidence** (low / med / high) — how sure you are this is a real problem
  - **Effort** (S / M / L) — rough cost to fix
- **Do NOT compute a single priority score.** The user weights axes differently depending on their situation. Presenting one number erases the distinction.
- Detect **conflicts**: findings whose suggested directions are mutually exclusive. Mark them explicitly so the user knows to pick at most one.

### Step 5: Present findings → user selection

See "Finding Format and Presentation" below.

### Step 6: Write the spec document

After the user selects a non-empty subset, write the spec to:

`docs/superpowers/specs/YYYY-MM-DD-refactor-<scope>-design.md`

(In the target project, NOT in `~/.claude/`.)

Commit the spec to git with a message like `docs: refactor diagnosis for <scope>`. Then invoke `writing-plans` with the spec as input.

If the user selects nothing (empty selection), terminate cleanly without writing a spec. Do NOT pressure the user to pick something.

### Subagent dispatch template (for sweep deep pass)

When dispatching a subagent for Feynman reconstruction of a region, use this prompt verbatim (substituting the bracketed parts):

> You are performing a Feynman-style refactoring diagnosis of `[region path]`.
>
> Your task: for each significant unit (file, class, function) in this region, try to explain in plain language **why this unit should exist** — not what it does, why it has to exist.
>
> **Critical self-check (do this for EVERY reconstruction):** Is my explanation a plain-English translation of what the code does, or is it an explanation of why this code should exist? If it is the former, the reconstruction has failed — and the failure is itself the finding.
>
> Record findings in the format specified in `~/.claude/skills/refactor/SKILL.md` under "Finding Format". Each finding MUST have Observation (facts only, no interpretation) separate from Reconstruction attempt (interpretation). The failure point — where the reconstruction broke down — IS the reason the finding exists.
>
> Context signals (read as signals, not truth): `[paths to docs/README/tests relevant to this region]`
>
> Do NOT propose implementation plans. One or two sentences of "suggested direction" is the maximum. Detailed planning is a later phase.
>
> Do NOT group findings by "priority" (high/med/low). Use the three independent axes: Impact, Confidence, Effort.
>
> Return a markdown report with all findings.

## Finding Format

Each finding has this exact structure:

```markdown
### F-{id}: {one-line summary}
- **Location**: `path/to/file.py:42` (or directory for structural findings)
- **Category**: structure | naming | logic | complexity | dead-code | duplication | perf-waste
- **Observation**: What is visible in the code. Facts only, NO interpretation. Example: "This function is called from three places, all of which discard its return value."
- **Reconstruction attempt**: "This unit exists to do X, because Y..." — a plain-language why-it-should-exist.
- **Failure point**: Where and why the reconstruction broke down. THIS is the reason it is a finding.
- **Suggested direction**: One or two sentences of direction. NOT an implementation. Detailed planning is deferred to `writing-plans`.
- **Axes**: Impact: low|med|high, Confidence: low|med|high, Effort: S|M|L
- **Conflicts** *(optional)*: `F-05 proposes the opposite direction — mutually exclusive`
```

### Why Observation and Reconstruction are separated

**Observation** is irrefutable fact. **Reconstruction** is interpretation. Separating them lets the user say "the observation is correct but the interpretation is wrong", preserving honest review even when you misread. If you blend them into one paragraph, the user cannot audit your reasoning — and you cannot audit it yourself.

## Presentation and Selection

Present findings in TWO stages.

### Stage 1 — Summary table (for scanning)

```
ID    | Location                          | Category  | Imp  | Conf | Eff | Summary
F-01  | src/ml/training/trainer.py        | logic     | high | high | M   | loss schedule diverges from config
F-02  | src/ml/data/                      | structure | med  | high | S   | augment/ and preprocess/ boundary unclear
F-03  | src/ml/model/lora/wrapper.py:88   | dead-code | low  | high | S   | unused fallback branch
```

### Stage 2 — Detail on demand

User requests detail by:
- **ID list**: "show F-01, F-03"
- **Filter**: "show all high-impact", "show all S-effort", "show all logic"
- **Everything**: "show all"

### Selection modes

- Explicit IDs: "F-01, F-03, F-07"
- Filter: "all high-impact", "all S-effort", "all logic"
- **Empty selection is valid** — user can look and decide to fix nothing right now. Terminate cleanly without writing a spec.

## Output: the spec document

After non-empty selection, write:

`docs/superpowers/specs/YYYY-MM-DD-refactor-<scope>-design.md` (in target project)

**Contents:**
1. **Scope** — what was diagnosed
2. **Context summary** — what sources were read
3. **Full body of each selected finding**
4. **Refactoring constraints** — tests must still pass, public API stable, etc.
5. **Success criteria** — specifically: "after the refactor, the same Feynman reconstruction should no longer get stuck at the failure point"

Commit the spec. Invoke `writing-plans` with the spec as input.

## Rationalizations to Watch For

These are shortcuts that agents (including you) take to avoid the actual Feynman reconstruction. If you catch yourself thinking any of these, STOP and restart the reconstruction properly.

| Rationalization | Reality |
|-----------------|---------|
| "Priority is obvious — just group by high/med/low" | That collapses three independent axes into one. A high-impact low-confidence finding is very different from a high-impact high-confidence one. Use the three axes separately. |
| "A finding is a paragraph describing a problem" | No. A finding has six named fields (Location, Category, Observation, Reconstruction, Failure point, Direction) + axes. Prose paragraphs blend observation and interpretation; the reader cannot audit blended reasoning. |
| "I see a problem, I should suggest the concrete fix" | No. One or two sentences of direction only. Full fix plans come from `writing-plans`, not from this skill. |
| "Why-does-this-exist is the same as what-does-this-do" | These are completely different questions. Paraphrase of behavior = failed reconstruction. Only catches real issues when sources visibly disagree — misses most silent problems. |
| "The user said refactor X, let me propose the file splits" | No. Diagnose first, present findings, get a subset selection, THEN hand off to `writing-plans`. Jumping to implementation skips every quality gate this skill exists to enforce. |
| "I should ask the user what the intent is" | Not up-front. Asking first invites post-hoc rationalization. Only ask when a specific reconstruction conflict makes the question concrete. |
| "The code is self-explanatory; no reconstruction needed" | Then the reconstruction is trivial — write it out. "Self-explanatory" almost always means "I paraphrased in my head and moved on". Do it on paper. |
| "This file has 500 lines; I should read it sequentially" | Not in sweep mode. Triage first using size, complexity, churn, random sampling. Sequential reading exhausts context and produces uniformly shallow findings. |

## Red Flags — STOP and Restart

If you notice any of the following while producing findings, you have slipped. Stop, go back to the Feynman self-check, and redo the reconstruction:

- Your "Reconstruction" field reads like a paraphrase of the code
- Your "Observation" and "Reconstruction" fields contain the same information
- You computed a single priority score despite the three-axis rule
- You jumped from findings directly to implementation code
- You asked the user "what is the intent of this code?" before attempting your own reconstruction
- You are reading files sequentially in sweep mode instead of triaging first
- You grouped findings as "high priority / medium priority / low priority"
- Your finding is written as a prose paragraph without the six named fields

All of these mean the same thing: **go back to the Feynman self-check and do the reconstruction honestly.**
