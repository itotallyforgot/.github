# Rulesets

Wave-0 branch-protection floor for the fleet, plus the archetype mapping that
decides which frozen aggregate checks a given repo must require.

## Files

- `ruleset-public.json` — floor for public repos.
- `ruleset-private.json` — floor for private repos.

Both are identical in rule content today; they are split so public-only policy
(if it diverges later) has a home. Each carries the **full menu** of frozen
aggregate status checks. The `_comment` key documents intent and is stripped
before `POST`.

## The floor (every repo)

- `pull_request` required (`required_approving_review_count: 0` — solo
  self-merge stays possible, but direct pushes to the default branch are
  blocked).
- `non_fast_forward` — no force-push to the default branch.
- `deletion` — the default branch cannot be deleted.
- `bypass_actors: []` — nobody bypasses, including admins.

## Frozen aggregate status checks

Each reusable workflow ends in a single aggregate job whose name is a frozen
API. Rulesets reference these names. **Never rename them.**

| Workflow              | Frozen check     |
| --------------------- | ---------------- |
| security-baseline.yml | `security-passed` |
| ci-python-uv.yml      | `ci-passed`       |
| ci-node-pnpm.yml      | `node-ci-passed`  |
| ci-go.yml             | `go-ci-passed`    |
| ci-dotnet.yml         | `dotnet-ci-passed`|
| ci-static-shell.yml   | `static-ci-passed`|

## Archetype → required checks

Prune the `required_status_checks` array down to exactly the rows below before
applying a ruleset to a repo. A listed check that never runs in that repo
stays **pending forever** and blocks every merge.

| Archetype                 | Required aggregate checks                          |
| ------------------------- | -------------------------------------------------- |
| python-uv-lib             | `security-passed`, `ci-passed`                     |
| python-ts-webapp          | `security-passed`, `ci-passed`, `node-ci-passed`   |
| go                        | `security-passed`, `go-ci-passed`                  |
| python-csharp-mods        | `security-passed`, `ci-passed`, `dotnet-ci-passed` |
| python-docs-tooling       | `security-passed`, `static-ci-passed`              |
| shell-python-bench        | `security-passed`, `static-ci-passed`              |
| static-site               | none beyond the floor                              |
| profile-readme            | none beyond the floor                              |
| empty-stub                | none beyond the floor                              |

For the "none beyond the floor" archetypes, drop the `required_status_checks`
rule entirely and keep only `pull_request`, `non_fast_forward`, and `deletion`.

## Applying

```sh
# Strip the _comment key and any checks not in the archetype, then:
gh api -X POST repos/<owner>/<repo>/rulesets --input pruned-ruleset.json
```

Status-check contexts use the job NAME (the frozen aggregate), not the workflow
file name. `strict_required_status_checks_policy` is left `false` so a branch
does not have to be up to date with the base before merge; flip it to `true`
per repo if you want that.
