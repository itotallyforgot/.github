# Consuming the secure-CI paved path

This repository hosts the organization's reusable CI/security workflows. A
consuming repository wires a thin **caller** workflow that delegates to these
reusables. This page is the contract: copy the patterns verbatim.

## Model

- Reusables are consumed through the floating **`@v1`** tag. The owner moves
  `v1` forward on each central release, so a fix lands fleet-wide without a PR
  to every repo.
- Third-party actions are SHA-pinned **inside** the reusables. Consumers never
  pin the reusables by SHA — they pin by the `@v1` ref (see `.github/zizmor.yml`).
- Each CI reusable emits one **frozen aggregate check** whose name is a stable
  required-status-check surface. Never reference internal job names in a
  ruleset — only the aggregate.

| Reusable                | Aggregate check     | For |
|-------------------------|---------------------|-----|
| `ci-python-uv.yml`      | `ci-passed`         | uv-managed Python (`pyproject.toml` + `uv.lock`) |
| `ci-node-pnpm.yml`      | `node-ci-passed`    | pnpm Node/TS (`package.json` + `pnpm-lock.yaml`) |
| `ci-go.yml`             | `go-ci-passed`      | Go modules (`go.mod` + `go.sum`) |
| `ci-dotnet.yml`         | `dotnet-ci-passed`  | .NET |
| `ci-static-shell.yml`   | `static-ci-passed`  | shell / static repos |
| `security-baseline.yml` | `security-passed`   | every repo (gitleaks, zizmor, harden-runner, dependency-review) |

## 1. CI caller — `.github/workflows/ci.yml`

```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: read
jobs:
  ci:
    permissions:
      contents: read
      security-events: write   # REQUIRED even if SARIF upload is gated off (see Footguns)
    uses: itotallyforgot/.github/.github/workflows/ci-python-uv.yml@v1
    with:
      working-directory: .     # one caller job per project dir in a monorepo
```

Node/TS → `ci-node-pnpm.yml@v1`; Go → `ci-go.yml@v1`; .NET → `ci-dotnet.yml@v1`;
static → `ci-static-shell.yml@v1`. Inspect each reusable's `inputs:` and pass
only what it declares.

## 2. Security caller — `.github/workflows/security.yml`

```yaml
name: security
on:
  push: { branches: [main] }
  pull_request:
permissions:
  contents: read
jobs:
  baseline:
    permissions:
      contents: read
      security-events: write
    uses: itotallyforgot/.github/.github/workflows/security-baseline.yml@v1
    # NO `secrets: inherit` — the baseline needs no caller secrets.
```

Repo-specific scanners (semgrep, trivy-fs, pip-audit, bandit, smoke/eval suites)
stay as **repo-local jobs** in this file: SHA-pin every action,
`persist-credentials: false` on checkout, and grant the minimum `permissions:`
per job (`security-events: write` only when uploading SARIF).

## 3. zizmor policy — `.github/zizmor.yml`

Copy [`templates/zizmor.yml`](templates/zizmor.yml) to **`.github/zizmor.yml`**.
Without the policy, every `@v1` caller is reported as an `unpinned-uses` finding.

Location matters: zizmor (1.26+) auto-discovers `.github/zizmor.yml` (and a root
`zizmor.yml`), but **not** a root `.zizmor.yml` dotfile — a `.zizmor.yml` at the
repo root is silently ignored. The security baseline injects this policy at scan
time when no config is committed, so CI passes either way, but committing it to
`.github/zizmor.yml` is what makes a local `zizmor` run correct.

## 4. Scorecard

`security-baseline.yml` does not run Scorecard. Copy
[`templates/scorecard.yml`](templates/scorecard.yml) to
`.github/workflows/scorecard.yml` as a standalone workflow.

## Footguns

- **`security-events: write` is mandatory on every caller job.** GitHub
  validates nested-job permissions at startup, before any `if:` gate. A missing
  grant fails the whole workflow with `startup_failure` and zero jobs created —
  even when the SARIF-uploading step is disabled.
- **To pick up a moved `@v1`, push a fresh commit** (`git commit --allow-empty`).
  `gh run rerun --failed` re-runs against the reusable's *originally resolved
  commit* and will not re-resolve the tag.
- **A pre-merge code-scanning check can show a stale "fail" with zero real
  alerts**, because code-scanning cannot diff against a base branch that has no
  scanning yet. Confirm the true state with
  `gh api repos/<owner>/<repo>/code-scanning/alerts?tool_name=zizmor&state=open`
  (an empty array is clean). It resolves on merge.

## Making CI load-bearing

After a consumer's CI is green on the default branch, add a
`required_status_checks` rule to its branch ruleset, wired to the **aggregate
check contexts only** (format `<caller-job-name> / <aggregate>`), e.g.
`ci / ci-passed` and `baseline / security-passed`. Pin each to the GitHub
Actions app (`integration_id: 15368`) so no other check producer can satisfy
the gate.
