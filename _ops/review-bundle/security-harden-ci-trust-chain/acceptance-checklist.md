# Acceptance checklist — security-harden-ci-trust-chain

- [✓] Dependabot automation cannot fall back to an immediate merge — the sweep only enables gated squash auto-merge and fails otherwise.
- [✓] Reusable security jobs fail on scanner findings — Bandit and zizmor no longer suppress nonzero exits.
- [✓] Private repositories can run the baseline without paid GHAS features — dependency review and SARIF upload are independently configurable.
- [✓] Fork pull requests cannot upload SARIF — upload steps require a same-repository pull request head.
- [✓] Mutable tooling references are removed from the changed workflows — actions and the optional TruffleHog image use immutable digests.
- [✓] The central repository has a required-check candidate — `validate-passed` aggregates zizmor and gitleaks.
- [✓] Local static validation passes — `zizmor --no-config --collect all --strict-collection --min-severity high .` reported no findings.
- [⚠] Foreign-model adversarial review — skipped by explicit user direction; this is a documented delivery-gate waiver.
