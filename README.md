# skills

Custom skills for Claude Code. Each one exists because I kept repeating the same ad-hoc prompting and wanted a fixed workflow instead.

## Installation

Global:

```bash
cp -r skills/* ~/.claude/skills/
```

Project-scoped: copy into `.claude/skills/` at the project root instead.

## Skills

### `/upload-pypi`
Bumps the version in `pyproject.toml` (and any in-package version files), updates README references and the changelog when present, verifies, then pauses for confirmation before tagging, pushing, and publishing to PyPI via `uv publish` or `twine`. Will not reuse a version number, force-push a release tag, `git add -A`, or guess credentials.
