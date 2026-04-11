# skills

Custom skills for Claude Code.

## Installation

To use these globally, copy the directories under `skills/` into `~/.claude/skills/`:

```bash
cp -r skills/push skills/refactor skills/judge ~/.claude/skills/
```

For project-scoped use, place them under `.claude/skills/` in your project instead.

## Skills

### `/push`
Stage all changes, auto-generate a commit message, and push to the remote. Matches the repo's existing commit style from recent logs and keeps the message short and "why"-focused (in English).

### `/refactor`
**Plans** a refactor — does not modify code. Uses Feynman-style reconstruction: for each unit, try to explain in plain language *why it should exist*. The points where that explanation fails are the findings. Results are written to a spec document and handed off to `writing-plans`.

Not for specific bug fixes or hardware/library-level performance tuning.

### `/judge`
Stress-tests a tentative design or implementation decision through adversarial review. Two one-sided subagents — `naruhodo` (defense, steelman) and `mitsurugi` (prosecutor, attack) — gather sourced evidence while the main Claude (`taco`) judges across up to 3 stages and renders a verdict: **유지 / 조건부 유지 / 재검토 필요**. The full record is saved to `docs/trials/`.

Not for debugging, factual lookup, or zero-position brainstorming.
