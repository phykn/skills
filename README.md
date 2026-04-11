# skills

Custom skills for Claude Code — an opinionated set of workflows that extend Claude Code beyond its defaults. Each skill is a self-contained `SKILL.md` with YAML frontmatter that Claude Code auto-discovers and invokes by slash command or description match. They exist to turn points where I kept repeating the same ad-hoc prompting into fixed, reusable workflows.

## Installation

Global:

```bash
cp -r skills/push skills/refactor skills/judge ~/.claude/skills/
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
