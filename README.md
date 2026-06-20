# .github — the fleet paved path

Central, reusable CI and security workflows plus canonical config for every
repo in the fleet. Repos consume the workflows; they do not copy them.

## Reusable workflows

| Workflow                          | Purpose                                         | Frozen check     |
| --------------------------------- | ----------------------------------------------- | ---------------- |
| `.github/workflows/security-baseline.yml` | harden-runner, gitleaks, zizmor, dependency-review | `security-passed` |
| `.github/workflows/ci-python-uv.yml`      | uv sync, ruff, pyright, pytest+cov, optional bandit | `ci-passed`       |
| `.github/workflows/ci-node-pnpm.yml`      | pnpm lint, tsc, test, build, audit              | `node-ci-passed`  |
| `.github/workflows/ci-go.yml`             | go vet, golangci-lint, race tests, govulncheck  | `go-ci-passed`    |
| `.github/workflows/ci-dotnet.yml`         | dotnet build (warnaserror), test, vuln scan     | `dotnet-ci-passed`|
| `.github/workflows/ci-static-shell.yml`   | shellcheck/shfmt, markdownlint, lychee, opt pytest | `static-ci-passed`|

## How to consume — the `@v1` floating-tag model

Call a workflow from a consumer repo and pin it to the floating major tag `v1`:

```yaml
# .github/workflows/ci.yml in a consuming repo
name: ci
on:
  push:
    branches: [main]
  pull_request:

jobs:
  security:
    uses: itotallyforgot/.github/.github/workflows/security-baseline.yml@v1

  python:
    uses: itotallyforgot/.github/.github/workflows/ci-python-uv.yml@v1
    with:
      cov-floor: 80
      python-versions: '["3.11", "3.12"]'
```

The double `.github/.github/` is correct: the first is the repository name, the
second is the workflows directory.

### What `v1` means

- Consumers pin to `@v1`. The owner **force-moves** the `v1` tag to each new
  compatible release, so every consumer picks up fixes without editing their
  caller. `v1.0.0` (and later `v1.x.y`) are immutable point releases for anyone
  who wants to pin exactly.
- Inside these workflows, every third-party `uses:` is pinned to a **full
  40-char commit SHA** with a trailing `# vX.Y.Z` comment. Floating tags are a
  consumer-side convenience here, not a supply-chain shortcut — the actions we
  depend on are SHA-locked. Dependabot (`github-actions` ecosystem, see
  `templates/dependabot.yml`) keeps those SHAs current.
- A breaking change to a workflow's inputs or to a frozen check name ships as
  `v2`; `v1` keeps working.

## The frozen aggregate check contract

Every workflow ends in one aggregate job whose **name is a frozen API**:

```
security-baseline.yml -> security-passed
ci-python-uv.yml      -> ci-passed
ci-node-pnpm.yml      -> node-ci-passed
ci-go.yml             -> go-ci-passed
ci-dotnet.yml         -> dotnet-ci-passed
ci-static-shell.yml   -> static-ci-passed
```

Branch-protection rulesets reference these names as required status checks, so
**they are never renamed** (a rename is a `v2` break). The aggregate job
`needs:` every real job, runs `if: always()`, and **fails if any needed job
failed or was skipped** — a path-gated or condition-gated skip cannot turn the
gate green by skipping. Jobs that are intentionally optional (for example
`dependency-review`, which only runs on `pull_request`, or `bandit`, which is
opt-in) are tolerated as skipped **only** when the corresponding input or event
explicitly disables them; otherwise a skip still fails the gate.

Require the right checks per repo archetype — see `rulesets/README.md`.

## Templates (copy, don't call)

- `templates/scorecard.yml` — scheduled OSSF Scorecard for public repos.
  Scorecard needs repo-level permissions a reusable workflow can't inherit
  cleanly, so copy it to `.github/workflows/scorecard.yml`.
- `templates/dependabot.yml` — canonical Dependabot config. `github-actions`
  weekly is always on; uncomment the ecosystem blocks the repo uses.

## Hardening invariants (all workflows)

- Top-level `permissions: {}`; each job is granted only what it needs
  (`contents: read`, plus `security-events: write` only on SARIF-upload jobs).
- Every `actions/checkout` sets `persist-credentials: false`.
- Every third-party action is pinned to a full commit SHA.

## Layout

```
.github/workflows/   reusable workflows (workflow_call)
templates/           copy-me files (scorecard, dependabot)
rulesets/            branch-protection floor + archetype→checks mapping
CODEOWNERS           workflow + catch-all ownership
SECURITY.md          how to report vulnerabilities
```
