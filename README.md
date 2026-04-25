# skills

Custom skills for Claude Code. Each one exists because I kept repeating the same ad-hoc prompting and wanted a fixed workflow instead.

## Installation

Global:

```bash
cp -r skills/* ~/.claude/skills/
```

Project-scoped: copy into `.claude/skills/` instead.

## Skills

### `/upload-pypi`
Bumps the version in `pyproject.toml` (and any in-package version files), syncs README/changelog, verifies, then pauses for confirmation before tagging, pushing, and publishing to PyPI via `uv publish` or `twine`. Refuses to reuse versions, force-push tags, or guess credentials.
