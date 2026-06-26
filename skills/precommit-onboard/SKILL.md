---
name: precommit-onboard
description: >-
  Onboard or extend a repo's .pre-commit-config.yaml. Use when setting up
  pre-commit, adding linting hooks for the repo's primary language, adding
  gitleaks secret scanning, adding baseline pre-commit checks, wiring the
  zizmor hook when zizmor is installed, or SHA-freezing hook revs. Triggers:
  "set up pre-commit", "add pre-commit hooks", "extend pre-commit-config",
  "add gitleaks", "add linting hooks", "freeze pre-commit revs".
---

# Onboarding & extending .pre-commit-config.yaml

A repeatable playbook to create or extend a repo's `.pre-commit-config.yaml`.
Apply it end to end when onboarding a repo or adding hooks. Every step is
required unless marked optional.

## Goal state (definition of done)

1. Language linting hooks match the repo's **primary** language (detected, not assumed).
2. `gitleaks` is always present for static secret scanning.
3. A baseline of `pre-commit-hooks` general checks is present.
4. The `zizmor` hook is added **iff** `zizmor` is installed on the machine and there's something it audits (`.github/workflows/` or `.github/dependabot.yml`).
5. Every `rev:` is pinned to a **commit SHA** (not a mutable tag), via `pre-commit autoupdate --freeze`.
6. `pre-commit run --all-files` passes (or remaining findings are intentional).

## Prerequisites

- `pre-commit` installed: `brew install pre-commit` (or `pipx install pre-commit`, `uvx pre-commit`). Check `pre-commit --version`.
- `git` repo with at least one commit (pre-commit installs into `.git/hooks`).

## Step 1 — Detect the primary language

Count tracked source files by extension; the top non-doc extension is the primary language.

```bash
git ls-files | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20
```

Also check for manifest files that pin the language and toolchain:

```bash
ls go.mod package.json tsconfig.json pyproject.toml requirements.txt Cargo.toml \
   pom.xml build.gradle *.csproj Gemfile composer.json 2>/dev/null
```

Pick the dominant language. If two are close (e.g. a TS frontend + Python backend), add hooks for both. Note what you detected before writing config.

## Step 2 — Map language → linting hooks

Add the matching block(s). Each `rev` is a placeholder — Step 5 freezes them to SHAs.

| Language | Repo | Hooks |
| --- | --- | --- |
| Python | `astral-sh/ruff-pre-commit` | `ruff` (lint, `--fix`) + `ruff-format` |
| Go | `golangci/golangci-lint` | `golangci-lint` (or local `go vet`/`gofmt` hooks) |
| JS/TS | `pre-commit/mirrors-eslint` + `pre-commit/mirrors-prettier` | `eslint`, `prettier` |
| Rust | local hooks | `cargo fmt --check`, `cargo clippy` |
| Shell | `shellcheck-py/shellcheck-py` | `shellcheck` |
| YAML | `adrienverge/yamllint` | `yamllint` |
| Markdown | `igorshubovych/markdownlint-cli` | `markdownlint` |
| Terraform | `antonbabenko/pre-commit-terraform` | `terraform_fmt`, `terraform_validate` |
| .NET/C# | local hook | `dotnet format` |
| Dockerfile | `hadolint/hadolint` | `hadolint` |

Prefer the language's canonical formatter+linter. Bind versions to the repo manifest where the tool supports it (don't hardcode a toolchain version a hook can read from `go.mod`/`pyproject.toml`).

### Python example

```yaml
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0   # frozen in Step 5
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

### Go example

```yaml
  - repo: https://github.com/golangci/golangci-lint
    rev: v1.62.0   # frozen in Step 5
    hooks:
      - id: golangci-lint
```

### JS/TS example

```yaml
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0   # frozen in Step 5
    hooks:
      - id: prettier
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v9.15.0   # frozen in Step 5
    hooks:
      - id: eslint
```

## Step 3 — gitleaks (always)

Always add gitleaks for static secret scanning, regardless of language.

```yaml
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2   # frozen in Step 5
    hooks:
      - id: gitleaks
```

## Step 4 — Baseline checks (always)

Add the standard `pre-commit/pre-commit-hooks` baseline. These are cheap, language-agnostic guards.

```yaml
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0   # frozen in Step 5
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key
      - id: mixed-line-ending
```

## Step 5 — zizmor hook (conditional)

Add the zizmor hook **only if** zizmor is installed locally AND the repo has something zizmor audits — GitHub Actions workflows OR a `dependabot.yml` (zizmor scans both). Check:

```bash
command -v zizmor && ls .github/workflows/*.y*ml .github/dependabot.y*ml 2>/dev/null
```

If zizmor is installed and either path exists, add the hook. Resolve the tag matching the local `zizmor --version`:

```yaml
  - repo: https://github.com/zizmorcore/zizmor-pre-commit
    rev: v1.26.1   # frozen in Step 5
    hooks:
      - id: zizmor
```

Skip it if neither condition holds (not installed = hook can't run; no workflows and no dependabot.yml = nothing to scan). For deeper Actions hardening beyond the hook, see the `zizmor-harden` skill.

## Step 6 — Freeze every rev to a SHA

Mutable tags (`v8.21.2`) can be moved; a commit SHA can't. Pin every `rev` to its SHA with `--freeze`, which rewrites each `rev` to the tag's commit SHA and leaves the tag as a trailing comment.

```bash
pre-commit autoupdate --freeze
```

Result form:

```yaml
  - repo: https://github.com/gitleaks/gitleaks
    rev: 77c3c6a34b2577d71ac76e0d800f95e992ae6e83  # frozen via `pre-commit autoupdate --freeze`; v8.21.2
    hooks:
      - id: gitleaks
```

`autoupdate` also bumps to the latest release before freezing. To freeze at the current pinned version without bumping, freeze a single repo and re-run per repo:

```bash
pre-commit autoupdate --freeze --repo https://github.com/gitleaks/gitleaks
```

If `pre-commit` isn't installed, resolve SHAs by hand as a fallback:

```bash
gh api repos/gitleaks/gitleaks/commits/v8.21.2 --jq '.sha'
```

## Step 7 — Install & verify

```bash
pre-commit install                  # wire into .git/hooks
pre-commit run --all-files          # run every hook over the whole tree
```

Fix or intentionally accept findings. First run may reformat files (ruff/prettier/end-of-file-fixer) — re-stage and re-run until clean.

Commit the config under a `ci:`/`chore:` prefix. State which language was detected and which hooks were added.

## Merging into an existing config

- If `.pre-commit-config.yaml` already exists, **merge** repo entries — never overwrite.
- Don't duplicate a repo block; add missing hook `id`s to the existing one.
- Re-run `pre-commit autoupdate --freeze` after merging so new entries are SHA-pinned too.
- Preserve any existing `default_language_version`, `exclude`, or `ci:` keys.

## Full example (Python repo, post-freeze)

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: cef0300fd0fc4d2a87a85fa2093c6b283ea36f4b  # frozen via `pre-commit autoupdate --freeze`; v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key
      - id: mixed-line-ending
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: 8b76f04e7e5a9cd259e9d1db7799599355f97cdf  # frozen via `pre-commit autoupdate --freeze`; v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/gitleaks/gitleaks
    rev: 77c3c6a34b2577d71ac76e0d800f95e992ae6e83  # frozen via `pre-commit autoupdate --freeze`; v8.21.2
    hooks:
      - id: gitleaks
  - repo: https://github.com/zizmorcore/zizmor-pre-commit
    rev: e3eebf65325ccc992422292cb7a4baee967cf815  # frozen via `pre-commit autoupdate --freeze`; v1.26.1
    hooks:
      - id: zizmor
```

## Pitfalls

- `autoupdate --freeze` **bumps to latest** before freezing — a major bump can change lint rules and suddenly fail clean code. To hold a version, freeze per-repo or hand-resolve the SHA.
- Hooks that fix in place (`ruff --fix`, `prettier`, `end-of-file-fixer`) exit non-zero on the first run when they change files. That's expected — re-stage and re-run.
- The zizmor hook is static — it doesn't replace a CI zizmor gate. Add the hook for fast local feedback; keep the CI job for enforcement (see `zizmor-harden`).
- `check-added-large-files` defaults to 500 kB; raise with `args: ['--maxkb=1024']` only if the repo legitimately tracks large assets.
- Don't add language hooks for files that aren't actually in the repo — detect first (Step 1), don't assume.
