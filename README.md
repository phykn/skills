# skills

Custom skills for Claude Code. Each one exists because I kept repeating the same ad-hoc prompting and wanted a fixed workflow instead.

## Installation

Global:

```bash
cp -r skills/* ~/.claude/skills/
```

Project-scoped: copy into `.claude/skills/` instead.

## Skills

### `/push`
Stages all changes, writes a short English commit message matching the repo's recent style, and pushes to the tracking remote. On failure, diagnoses instead of retrying.

### `/refactor`
**Plans** a refactor — does not modify code. Uses Feynman-style reconstruction: for each unit, try to explain *why it should exist*. Where that explanation fails is the finding. Findings use six named fields and three independent axes (Impact / Confidence / Effort) — never a single priority score. After user selection, writes a spec and hands off to `writing-plans`.

Not for specific bug fixes, profile-driven perf work, or hardware/library-level tuning.

### `/judge`
Stress-tests a tentative decision through adversarial review. Two one-sided subagents — `naruhodo` (steelman) and `mitsurugi` (attack) — gather sourced, tiered evidence while `taco` (the main Claude) judges across up to 3 auto-terminating stages and renders **keep / keep-with-conditions / reconsider**. Full record saved to `docs/trials/`. User-facing output mirrors the user's language.

Requires a tentative position + uncertainty + a decision. Not for debugging, factual lookup, or zero-position brainstorming.

### `/shut-up`
Cuts redundancy, restatement, and defensive cruft from prose, code, and Claude's own responses. One rule: if removing it does not lose information the reader needs, remove it.
