# AGENTS.md

Agent guidelines for **ghaups**. `CLAUDE.md` is a symlink to this file. Edit
this file; never replace the symlink with a copy.

## What this is

Single-file Python CLI (`ghaups.py`) that rewrites `uses:` lines in GitHub
Actions workflow files to pinned SHA references, then scans each resolved SHA
with [Trivy](https://trivy.dev/docs/latest/getting-started/installation/) for
CRITICAL vulnerabilities. Also shipped as a Docker-based GitHub Action
(`action.yml`) and a reusable org-wide workflow (`org-pin-actions.yml`).

## Quick start

```bash
uv run ghaups.py .github/workflows/*.yml
```

## Key commands

```bash
uv run ghaups.py <files...>            # update + scan (default)
uv run ghaups.py --no-scan <files...>  # skip Trivy scan
uv run ghaups.py --no-update <files...># pin current version to SHA, no update check
uv run ghaups.py --log-level debug <files...>
```

There is no test suite, linter, or typecheck config, so there is nothing to run
for a single test. Verify changes by running the tool against a throwaway copy —
there is no dry-run mode and writes are in-place:

```bash
cp .github/workflows/release.yml /tmp/t.yml && uv run ghaups.py --no-scan --log-level debug /tmp/t.yml
```

## Architecture

- **Entrypoint:** `ghaups.py` — single module, no package structure, no tests.
- **Dependencies:** Python ≥3.11 and `requests`, managed by `uv` (`uv.lock`).
  The Docker action does **not** use uv — `Dockerfile` runs a plain
  `pip install requests`. A dependency change must be made in both places.
- **Cache:** `~/.ghaups_cache.json`, 1-hour TTL, keyed `owner/repo` → latest
  version + SHA. Loaded once per run and flushed at exit. Only the update path
  uses it; `--no-update` calls `get_sha_for_tag` directly and never hits cache.
- **GitHub Action:** `action.yml` + `Dockerfile` + `entrypoint.sh` — maps
  `INPUT_*` env vars to CLI args. `entrypoint.sh` passes `$INPUT_FILES` unquoted
  with no `nullglob`, so callers must expand globs themselves; an unmatched
  pattern reaches Python as a literal path and errors.
- **`pyproject.toml`'s `version` is vestigial** — nothing reads it, there is no
  `--version` flag, and it has already drifted from the released tags.

## Workflows

- `release.yml` — on a `MAJOR.MINOR.PATCH` tag push: creates the GitHub Release
  and force-moves the floating `MAJOR` / `MAJOR.MINOR` tags forward.
- `org-pin-actions.yml` — a reusable (`workflow_call`) workflow **for consumers**,
  not CI for this repo. Mints a GitHub App token, discovers repos the App is
  installed on, fans out over a matrix, and opens a `ghaups/pin-actions` PR to
  each. It pins `devopscoop/ghaups@<sha> # <version>` at its "Pin actions" step —
  **bump that self-reference after every release** or the org flow keeps running
  the old version.
- `claude.yml` (responds to `@claude` mentions) and `claude-code-review.yml`
  (automatic PR review). Both pin `--model claude-opus-5 --effort xhigh`. In the
  review job the `--allowed-tools` allowlist is load-bearing: without an
  `mcp__github_inline_comment__` entry the MCP server is never registered and the
  job silently posts nothing. Read the inline comments in both files before
  editing them.
- There is no CI that builds, tests, or lints this repo, and no
  `.pre-commit-config.yaml` is checked in — the `pre-commit` / `zizmor` policy
  below assumes a local setup, not a repo-committed one.

## Release

```bash
git tag 0.1.0
git push origin 0.1.0
```

Push the tag to every remote you have configured (`git remote -v` — this clone
has only `origin`). The GitHub push triggers `release.yml`, which creates the
Release and force-updates the `0` and `0.1` floating tags. Afterwards, bump the
`devopscoop/ghaups@<sha>` pin in `org-pin-actions.yml`.

## How it works

1. Parses `uses: owner/repo@ref` lines from workflow files.
1. Follows GitHub `/releases/latest` redirect to find latest version.
1. Resolves version tag → commit SHA via GitHub API.
1. Rewrites file with `owner/repo@<sha> #vX.Y.Z` format (pinned + commented).
1. Optionally runs `trivy repository --scanners vuln --severity CRITICAL --commit <sha> <repo_url>`.

## Behavioral quirks

- `--no-update` resolves the **current** tag to its SHA (does not look for newer
  versions). A 40-character ref is assumed to already be a SHA and is left alone.
- `--no-scan` and `--no-update` are independent flags; both can be combined.
- Logging splits: INFO/DEBUG → stdout, WARNING/ERROR → stderr.
- Trivy scans only CRITICAL severity (`--severity CRITICAL`; `HIGH` is present
  but commented out in the code, despite what README.md says).
- Scanning happens after **all** files are written, deduplicated by
  `(owner, repo, sha)`, so an action used in several files is scanned once.
- A `uses:` owner containing a `.` is skipped, which is how local `./...` and
  host-qualified refs are filtered out.
- Any Trivy failure — vulnerabilities, timeout, or Trivy not installed — counts
  toward the total and makes the run `sys.exit(1)` at the end.

## Common gotchas

- **Trivy must be installed separately** — not a Python dependency.
- GitHub API has rate limits; the 1-hour cache helps but hitting many actions may still fail.
- Only scans actions after resolving SHA (not unpinned tag refs).
- No dry-run mode; files are written in-place on update.
- **Never use `git commit --no-verify`** — pre-commit hooks (zizmor, etc.) enforce SHA pinning and other policies.
- **Never use a plain `git push --force`.** The single exception is moving the
  floating `0` / `0.1` tags in `.github/workflows/release.yml`, where re-pointing
  an existing tag has no non-force path. Anywhere a branch must be refreshed
  (e.g. the `ghaups/pin-actions` bot branch in `org-pin-actions.yml`), use
  `--force-with-lease` so a concurrent change can't be silently clobbered.

## Package manifests

This repo ships a `Brewfile` (macOS: `brew bundle`) and a `pkglist.txt` (Arch Linux) that install every CLI tool the repo uses (git, pre-commit, trivy, uv, zizmor). Keep them in sync with the code:

- When you add a tool, script, or a new external command (e.g. a new subprocess in ghaups.py), add the package to BOTH files, with a comment noting what uses it.
- When a tool stops being used, remove it from both files.
- Python library dependencies belong in pyproject.toml/uv.lock (managed by uv), NOT in the package manifests.
- Verify package names before adding them: `brew info <formula>` for Homebrew, and the official repos/AUR for Arch (e.g. Homebrew `gh` is Arch `github-cli`). If a package is AUR-only, note that in pkglist.txt's header instructions.
- Update the "Install required packages" subsection under Requirements in README.md if the tool list changes.
