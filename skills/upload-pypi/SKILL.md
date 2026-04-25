---
name: upload-pypi
description: Bump version in pyproject.toml, sync README, git push, and publish to PyPI
disable-model-invocation: false
allowed-tools: Bash, Read, Edit, Grep, Glob
---

Publish the current Python project to PyPI. Run the steps in order. On any failure, **stop** and report.

## Step 1 — Bump version in `pyproject.toml`

1. Read `pyproject.toml` (or `setup.py` / `setup.cfg`). Find `version` under `[project]` or `[tool.poetry]`.
2. Ask the user for the bump (`patch` / `minor` / `major`) unless specified.
3. Edit only the version field.
4. Grep for matching version strings across the package: `__init__.py`, `_version.py` / `version.py`, and `docs/conf.py` (Sphinx `release`/`version`). Update all matches to stay in sync.

## Step 2 — Update `README.md`

Update any old-version references: install snippets (`pip install pkg==X.Y.Z`), badges, "Latest version" lines. If a `CHANGELOG.md` exists, add a section for the new version using `git log $(git describe --tags --abbrev=0 --match 'v*')..HEAD` so non-release tags (e.g. `nightly-*`) cannot become the base. Skip if nothing references the version — do not invent changes.

## Step 3 — Verify

First, in parallel:

- `grep -rn "<old_version>" pyproject.toml README.md src/` — should be empty for version fields.
- Confirm build tool: `uv --version` or `python -m build --version`. If neither, stop and ask.

Then, sequentially:

- **Run the full test suite** (`pytest` or the project's command). Wait for it to finish. Do not skip, xfail, or delete tests to force a green run. If the project has no tests at all, stop and get explicit user confirmation before continuing. A failing test halts the release — do not proceed to the confirmation pause.
- Show old → new version and changed files. **Pause for user confirmation before Step 4** — PyPI uploads are not reversible.

## Step 4 — Git push

1. `git status` / `git diff` to confirm only expected files changed.
2. `git add` the specific files — never `git add -A`.
3. Commit `release: vX.Y.Z`.
4. `git tag vX.Y.Z`.
5. `git push origin HEAD && git push origin vX.Y.Z`.

## Step 5 — Build and upload

1. `rm -rf dist/ build/ *.egg-info`
2. Build: `uv build` or `python -m build`.
3. Check `dist/` has sdist + wheel with the new version.
4. Upload — pick the method by checking credentials in order:
   - If `~/.pypirc` exists: use `twine upload dist/*` (twine reads `~/.pypirc` automatically; `uv publish` does **not**).
   - Else if `UV_PUBLISH_TOKEN` is set: use `uv publish`.
   - Else if `TWINE_USERNAME`/`TWINE_PASSWORD` are set: use `twine upload dist/*`.
   - Otherwise: stop and ask the user to configure credentials.
   - Never echo or print `UV_PUBLISH_TOKEN`, `TWINE_PASSWORD`, or `~/.pypirc` contents — when surfacing an auth error, redact the token value and report only the variable name.

On failure:
- **Duplicate version**: bump again; PyPI will not accept the same version twice.
- **Auth**: stop and ask the user to configure `~/.pypirc` / `TWINE_*` / `UV_PUBLISH_TOKEN`. Never guess credentials.
- **`uv publish` credential error with `~/.pypirc` present**: `uv publish` does not read `~/.pypirc`. Fall back to `twine upload dist/*`.

## Step 6 — Report

New version, PyPI URL (`https://pypi.org/project/<package>/<version>/`), tag pushed, any follow-up.

## Safety rules

- Never upload without explicit user go-ahead after Step 3.
- Never upload with failing or skipped tests. No exceptions.
- Never reuse a version number.
- Never force-push a release tag.
- Never `git add -A` during release.
- Never commit or echo PyPI tokens.
