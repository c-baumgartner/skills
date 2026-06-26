---
name: zizmor-harden
description: >-
  Onboard or rework GitHub Actions workflows with zizmor security scanning.
  Use when setting up CI hardening, SHA-pinning actions, fixing zizmor
  findings, configuring dependabot cooldown, adding a zizmor CI gate, or
  wiring the zizmor pre-commit hook. Triggers: "harden the workflows", "run
  zizmor", "pin actions to SHAs", "add a zizmor check", "secure GitHub Actions".
---

# Hardening GitHub Actions with zizmor

A repeatable playbook to audit and harden a repo's GitHub Actions setup with
[zizmor](https://docs.zizmor.sh). Apply it end to end when onboarding a repo or
reworking CI. Every step below is required unless marked optional.

## Goal state (definition of done)

1. Every `uses:` is pinned to a full commit SHA with a `# vX.Y.Z` comment.
2. `zizmor .github/` AND `zizmor --persona auditor .github/` both report **no findings** (inline-ignore the rare false positive, with a reason).
3. `dependabot.yml` is scanned by zizmor and has a `cooldown` (if the file exists).
4. A CI job runs zizmor on every PR.
5. `.pre-commit-config.yaml` includes the zizmor hook.

## Prerequisites

- `zizmor` installed locally: `brew install zizmor` (or `uvx zizmor`, `pipx install zizmor`). Check `zizmor --version`.
- `gh` CLI authenticated (used to resolve SHAs and look up action metadata).
- Optional: `actionlint` for an orthogonal workflow-syntax check.

## Step 1 — Inventory

Find everything zizmor audits: workflows, dependabot config, and any composite/local actions.

```bash
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
ls .github/dependabot.yml .github/dependabot.yaml 2>/dev/null
find . -name action.yml -o -name action.yaml 2>/dev/null   # composite/local actions
```

## Step 2 — Baseline scan (both personas)

```bash
zizmor .github/                    # default persona: low/medium/high actionable findings
zizmor --persona auditor .github/  # auditor: also informational + low-confidence (concurrency, anonymous-definition, cooldown, ...)
```

Exit code is non-zero when findings exist (e.g. 13). Fix default-persona findings first, then re-run with `--persona auditor` and fix the rest. Read each finding's `audit documentation` URL when unsure.

## Step 3 — SHA-pin every action

Unpinned `@v4` / `@main` refs are mutable. Pin to the immutable commit SHA, keeping a version comment. Resolve the SHA robustly (works for both lightweight and annotated tags):

```bash
gh api repos/OWNER/REPO/commits/TAG --jq '.sha'   # e.g. repos/actions/checkout/commits/v6.0.2
```

Result form:

```yaml
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
```

When bumping an action across a major version, read its release notes for breaking changes to the inputs you use:

```bash
gh api repos/OWNER/REPO/releases/tags/TAG --jq '.body'
```

If the bump is to clear a Node.js runtime deprecation, confirm the new tag targets the newer runtime:

```bash
gh api repos/OWNER/REPO/contents/action.yml --jq '.content' | base64 -d | grep -i 'using:'   # want node24 (or composite)
```

## Step 4 — Apply the canonical fixes

Map each finding class to its fix:

| zizmor finding | Fix |
| --- | --- |
| `unpinned-uses` | Pin to commit SHA + `# vX.Y.Z` (Step 3). |
| `artipacked` | Add `with: { persist-credentials: false }` to `actions/checkout` (unless a later step pushes with the persisted token). |
| `excessive-permissions` | Set top-level `permissions: { contents: read }`; grant write scopes per-job, only on the job that needs them. |
| `cache-poisoning` | On release/publish workflows, set `cache: false` on `setup-*` actions so a poisoned cache can't reach published artifacts. |
| `template-injection` | Never interpolate `${{ ... }}` directly inside a `run:` block. Pass values via `env:` and reference shell vars (`"$MY_VAR"`). |
| `secrets-outside-env` | Add `environment: <name>` to the job referencing the secrets, so branch/reviewer protection rules can gate them. Repo-level secrets still resolve; flag that protection rules must be configured in repo Settings to realize the benefit. |
| `concurrency-limits` | Add a `concurrency:` block. PR checks: `group: <wf>-${{ github.ref }}`, `cancel-in-progress: true`. Release/deploy: key by tag/ref, `cancel-in-progress: false` (never abort mid-publish). |
| `anonymous-definition` | Give every job a `name:`. |
| `superfluous-actions` | Prefer a first-party tool over a third-party action when it covers the used features. E.g. replace `softprops/action-gh-release` with `gh release create "$TAG" assets/* --generate-notes --prerelease="$PRE" --latest="$LATEST"` (gh is preinstalled). **If the job has no `actions/checkout` step (e.g. a publish job that only downloads artifacts), gh has no local git repo** — set `env: { GH_REPO: ${{ github.repository }} }` so gh targets the repo via the API, and do NOT pass `--verify-tag` (it shells out to git and fails). Only keep the action if you need features gh lacks (draft, discussions, changelog templating, asset rename). |
| `dependabot-cooldown` | See Step 6. |

Notes:
- Prefer `go-version-file: go.mod` (and equivalents for other languages) over a hardcoded version, so CI tracks the manifest and never drifts.
- **Inline ignore must sit on the line zizmor points at** (the `uses:`/`run:` line), not the line above. Combine with the version comment: `# v2.5.0 # zizmor: ignore[rule-name]`. Always add a short reason.

## Step 5 — Dedicated zizmor CI job

If no workflow runs zizmor, add a job (to the PR-check workflow, or a standalone `.github/workflows/zizmor.yml`). Pin the action to a SHA like any other.

```yaml
  zizmor:
    name: Workflow Security Audit (zizmor)
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@<sha> # vX.Y.Z
        with:
          persist-credentials: false
      - name: Run zizmor
        uses: zizmorcore/zizmor-action@<sha> # vX.Y.Z
        with:
          persona: auditor          # enforce the strictest bar once the repo is clean at it
          advanced-security: false  # set true ONLY if GitHub Advanced Security (SARIF upload) is enabled on the repo
          annotations: true         # surface findings as inline PR annotations (mutually exclusive with advanced-security: true)
          online-audits: false      # offline-only keeps the gate deterministic; set true for stale-action/network audits
```

Resolve the latest `zizmorcore/zizmor-action` tag + SHA with the Step 3 commands. Only use `persona: auditor` after the repo already passes auditor locally; otherwise it will block PRs on pre-existing informational findings.

## Step 6 — Dependabot cooldown + ecosystems

If `.github/dependabot.yml` exists, ensure each `updates:` entry has a `cooldown` (delays adopting freshly published versions so a compromised release can be yanked first). Add a `github-actions` ecosystem so the SHA pins from Step 3 stay maintained.

```yaml
version: 2
updates:
  - package-ecosystem: "gomod"   # or npm, pip, cargo, ...
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
```

If there is no `dependabot.yml`, creating one is optional (out of scope unless asked) — but if it exists, it must pass zizmor's auditor persona.

## Step 7 — Pre-commit hook

Create or update `.pre-commit-config.yaml` to add the zizmor hook (`zizmorcore/zizmor-pre-commit`). Pin `rev` to a **commit SHA**, not a mutable tag — same immutability rationale as Step 3. `pre-commit` does this for you with `--freeze`, which rewrites each `rev` to the tag's commit SHA and leaves the tag as a trailing comment.

Resolve the SHA for the tag matching your local `zizmor --version`:

```bash
gh api repos/zizmorcore/zizmor-pre-commit/commits/v1.26.1 --jq '.sha'
```

```yaml
repos:
  - repo: https://github.com/zizmorcore/zizmor-pre-commit
    rev: e3eebf65325ccc992422292cb7a4baee967cf815  # frozen via `pre-commit autoupdate --freeze`; v1.26.1
    hooks:
      - id: zizmor
```

If the file already exists, merge the repo entry rather than overwriting. Then freeze and verify:

```bash
pre-commit autoupdate --freeze --repo https://github.com/zizmorcore/zizmor-pre-commit   # pins rev to a SHA + tag comment
pre-commit install                                                                      # if not already installed
pre-commit run zizmor --all-files                                                       # verify the hook runs clean
```

Prefer running `pre-commit autoupdate --freeze` over hand-writing the SHA, so the SHA-to-tag mapping is authoritative. The hand-written example above is a fallback when `pre-commit` isn't installed locally yet.

## Step 8 — Verify

```bash
zizmor .github/ && zizmor --persona auditor .github/    # both: "No findings to report"
actionlint .github/workflows/*.yml                       # optional; ignore pre-existing custom self-hosted runner-label warnings
```

Commit workflow changes under a `ci:` prefix. Each commit message should state which finding classes were resolved and why any inline-ignore was kept.

## Pitfalls

- `advanced-security: true` (the action default) uploads SARIF and **requires GitHub Advanced Security** on the repo — it fails on repos without it. Use `advanced-security: false` + `annotations: true` unless GHAS is confirmed enabled.
- `online-audits: true` needs network + a token and can be flaky/rate-limited in CI. Keep it `false` for a deterministic gate; run online audits ad hoc.
- After SHA-pinning, a major-version bump can silently change inputs — always read release notes (Step 3).
- `secrets-outside-env` is only fully resolved when the named environment has protection rules configured in repo Settings; adding the `environment:` key alone satisfies zizmor but is cosmetic until rules exist. Say so explicitly.
- An auditor-persona CI gate will block PRs on informational findings — only enable it once the repo is already clean at that bar.
- zizmor and actionlint are static — they do not execute the workflow. A change that passes both can still fail at runtime (e.g. `gh release create` needing repo/git context). When you rework a release/publish workflow, watch the next real run (`gh run watch`) before considering it done.
