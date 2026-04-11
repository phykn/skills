# Refactor Skill Revision — Design

**Scope:** Revise `skills/refactor/SKILL.md` to (1) reframe the skill around two co-equal diagnostic lenses — Feynman reconstruction and MECE partition — and (2) remove duplicated passages that have accumulated across the file.

**Non-goals:** Changing the skill's hand-off contract (still plan-only, still delegates execution to `writing-plans` → `executing-plans`). Changing Step 4.5 falsification mechanics. Changing Finding Format fields beyond the `Category` values.

---

## Motivation

Two problems with the current skill:

1. **Feynman reconstruction is the only stated principle, but it only catches per-unit problems.** The skill occasionally mentions "MECE violations" in passing (Step 3A shallow pass) without promoting MECE to a first-class lens. As a result, structural findings — overlap between modules, gaps where a concern has no home, folder boundaries that don't reflect the problem partition — have no principled place in the skill and tend to be under-reported.

2. **The Feynman self-check is restated five times** (Overview, dedicated section, worked example's mechanical tests, subagent dispatch template, Failure Modes table). This is the single largest source of length and repetition in the file.

The user's stated goal: apply Feynman method to refactoring *and* end up with a MECE-organized folder/code structure. Both together.

## Design

### 1. Two co-equal lenses (Overview rewrite)

Replace the current Overview + "Core Principle" + "Feynman self-check" + "Worked example" block with a single Overview that establishes two lenses:

> This skill diagnoses code through two independent lenses. Neither subsumes the other; they catch different findings.
>
> **Lens ① — Feynman reconstruction (per-unit justification).**
> Question: *Why should this unit exist?*
> Catches: units with no reason (`dead-code`), unjustified complexity, `logic` that fails to enforce its claimed invariant, work with no purpose (`perf-waste`).
>
> **Lens ② — MECE partition (cross-unit structure).**
> Question: *Do these units divide the problem space without overlap or gap?*
> Applies at every granularity: folders, files, functions.
> Catches: unclear module boundaries (`structure`), names that do not reflect the partition (`naming`), the same concern in two cells (`duplication` = overlap), a concern with no cell (`gap`).
>
> Both lenses share one rule: **do not fetch intent from docs, tests, or comments. Reconstruct it.** The points where reconstruction fails are the findings.
>
> **Self-check (apply to every reconstruction, either lens):** "Is this a plain-English translation of what the code does, or an explanation of why it should exist / why this partition? If the former, the reconstruction has failed — the failure is itself the finding."

A short worked example for the Feynman lens (the existing `normalize_email` example, trimmed) stays. The "Mechanical tests you can apply when unsure" block is removed — it repeats the self-check in different words.

### 2. Process changes — lens asymmetry drives structure

Key asymmetry: **MECE needs a global view; Feynman is local.** Splitting MECE analysis across subagents would have each subagent see only its slice and miss overlap/gap between slices. This determines how Step 3A is organized.

**Step 3A (sweep mode) — renamed from "shallow → deep" to "MECE pass → Feynman pass":**

- **Pass 1 — MECE lens (main agent only).** Glob the tree, inspect folder/file naming, module boundaries. Produces two outputs:
  - MECE findings directly (`structure | naming | duplication | gap`) — these are first-class findings, not just triage signals.
  - The target list for Pass 2. Selection filters unchanged (size, complexity, git churn, random sampling) *plus* "files in boundary regions the MECE pass flagged as suspicious."
- **Pass 2 — Feynman lens (parallel subagents).** Each subagent performs Feynman reconstruction on its assigned targets. Subagents return `dead-code | logic | complexity | perf-waste` findings. Subagents do NOT perform MECE analysis — they lack the global view.

**Step 3B (targeted mode):** Main agent runs both lenses directly on the small scope. No subagents.

**Subagent dispatch template:** add a single sentence — "You run the Feynman lens only. MECE analysis is done by the main agent because it requires a global view. Do not write findings about module boundaries or files outside your scope." The existing self-check paragraph inside the template is compressed to a one-line reference back to the Overview.

### 3. Finding Format — Category grouping and new `gap`

```
Category (grouped by lens):
  Feynman lens: dead-code | logic | complexity | perf-waste
  MECE lens:    structure | naming | duplication | gap
```

`gap` is new and only the MECE lens can produce it — a concern the code base does not handle anywhere, identified by reconstructing the intended partition and noticing an empty cell.

### 4. Step 4.5 falsification — minor updates

- Add `gap` to the skip list (alongside `structure | naming | complexity`) — a missing concern cannot be falsified by execution.
- Drop the standalone "Who runs experiments" paragraph. Its content ("subagents do not run experiments") is now implied by Step 3A: subagents run Feynman only and return findings; experiments are a main-agent step.
- Compress the prose around the experiment table into bullets. Target: ~20 lines (from ~35).

### 5. Failure Modes — add an MECE row

Add to the existing table:

| Symptom | Why it is wrong |
|---------|-----------------|
| No `structure`, `duplication`, or `gap` findings after a sweep | MECE lens was skipped. A real codebase at sweep scope almost always has at least one boundary issue — zero MECE findings usually means the main agent only ran Feynman. |

Remove the "self-explanatory" row — it is the fifth restatement of the self-check.

### 6. Dedup targets (explicit list)

| Current location | Action |
|---|---|
| Overview L14 "All checks reduce to this one question" | Delete — contradicts the new two-lens framing |
| "Core Principle" section | Delete — merged into new Overview |
| "Feynman self-check" standalone section | Delete — merged into new Overview |
| "Worked example" → Mechanical tests 1/2/3 | Delete; keep example itself |
| Subagent template's self-check paragraph | Compress to one-line reference |
| Failure Modes "self-explanatory" row | Delete |
| Step 4.5 "Who runs experiments" paragraph | Delete |
| Step 4.5 prose | Compress to bullets |

## Expected size

Current: 216 lines. Target: ~150 lines. Self-check mentions: 5 → 1.

## Refactoring constraints

- `disable-model-invocation: false` and `allowed-tools` stay as they are (no tool changes).
- Finding Format fields unchanged except `Category` values.
- Step numbering (1 through 6, plus 4.5) stays stable so cross-references in other skills do not break.
- `docs/superpowers/specs/YYYY-MM-DD-refactor-<scope>-design.md` path for spec output unchanged.
- Still strictly plan-only — no execution of the refactor itself.

## Success criteria

- The Overview explains both lenses in under ~25 lines and never mentions the self-check more than once.
- A fresh reader can, from the Overview alone, predict which lens catches which Category value.
- Running the revised skill on a toy codebase produces at least one MECE finding (`structure`, `naming`, `duplication`, or `gap`) and at least one Feynman finding, with distinct reasoning chains.
- The subagent dispatch template makes it structurally impossible for a Feynman subagent to accidentally write a boundary-level finding outside its scope.
