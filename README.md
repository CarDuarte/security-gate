# Reusable Security Gate

A reusable GitHub Actions workflow that any project can call to scan for security
breaches, leaked secrets, vulnerable dependencies, and insecure infrastructure code.
It adapts automatically to the languages present in the calling repository.

## What it runs

| Job | Tool | What it catches | Gate behavior |
|---|---|---|---|
| `secrets-scan` | [Gitleaks](https://github.com/gitleaks/gitleaks) (pinned binary) | Secrets/credentials in the full git history | **Any** finding fails (a leaked secret is always critical) |
| `sast` | [Semgrep](https://semgrep.dev) (official image) | Code vulnerabilities — OWASP Top 10, injection, crypto misuse, etc. | Fails at/above `fail-on-severity` |
| `sca` | [OSV-Scanner](https://github.com/google/osv-scanner) | Known CVEs in dependencies (npm, pip, Go, Maven, Cargo, Composer, NuGet, RubyGems, …) | Fails when CVSS ≥ threshold; unrated vulns fail conservatively |
| `iac` | [Checkov](https://www.checkov.io) | Terraform, CloudFormation, Kubernetes, Helm, Dockerfile misconfigurations | Runs only when IaC is detected; any failed check fails |
| `gate` | — | Aggregates everything into one required status check | Fails if any enabled scan failed |

Language support is automatic: Semgrep's rulesets and OSV-Scanner's manifest detection
cover Python, JavaScript/TypeScript, Go, Java, Ruby, PHP, Rust, C#, and more — the
`detect` job reports what it found in the run summary.

Scanners are installed from **pinned official releases or official images**, not
marketplace wrapper actions (the `trivy-action`/`setup-trivy` wrappers were compromised
in a March 2026 supply-chain attack — this repo deliberately avoids that class of risk).

## Usage

In the calling repository, create `.github/workflows/security.yml`:

```yaml
name: Security

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write   # lets scan results appear in the repo's Security tab
  actions: read

jobs:
  security-gate:
    uses: CarDuarte/security-gate/.github/workflows/reusable-security-gate.yml@main
    with:
      fail-on-severity: high
    secrets:
      SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}   # optional
```

Then mark **Security gate result** as a required status check in branch protection.

> For stronger supply-chain guarantees, reference this workflow by commit SHA
> instead of `@main` once it stabilizes.

## Inputs

| Input | Default | Description |
|---|---|---|
| `scan-path` | `.` | Path within the caller repo to scan |
| `fail-on-severity` | `high` | Minimum severity that fails the gate: `critical`, `high`, `medium`, `low` |
| `semgrep-rulesets` | `p/default p/owasp-top-ten p/secrets` | Semgrep registry rulesets (ignored when `SEMGREP_APP_TOKEN` is set — the platform's policies are used instead) |
| `semgrep-custom-rules` | `semgrep-rules` | Path in the caller repo to extra Semgrep rules; used only if it exists |
| `gitleaks-config` | `gitleaks.toml` | Path in the caller repo to a custom Gitleaks config; used only if it exists |
| `enable-secrets-scan` | `true` | Toggle the Gitleaks job |
| `enable-sast` | `true` | Toggle the Semgrep job |
| `enable-sca` | `true` | Toggle the OSV-Scanner job |
| `enable-iac` | `true` | Toggle the Checkov job (it also auto-skips when no IaC is detected) |
| `upload-sarif` | `true` | Upload SARIF to GitHub code scanning (degrades gracefully if the repo lacks Advanced Security) |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `SEMGREP_APP_TOKEN` | No | If set, Semgrep runs in `semgrep ci` mode using your Semgrep AppSec Platform policies |

## Severity threshold mapping

`fail-on-severity` maps onto each tool's native scale:

| Threshold | Semgrep rules that fail | Dependency CVSS that fails |
|---|---|---|
| `critical` | ERROR | ≥ 9.0 |
| `high` | ERROR | ≥ 7.0 |
| `medium` | ERROR + WARNING | ≥ 4.0 |
| `low` | all | any |

Gitleaks findings and Checkov failed checks are not severity-filtered — a leaked
credential or misconfigured resource fails the gate regardless of threshold.

## Reports

Every scan uploads its SARIF report:

- **Security tab → Code scanning** in the caller repo (categories `security-gate-*`),
  when the repo has code scanning available.
- **Workflow run artifacts** (`security-gate-*-report`) in all cases.

## Customizing per repo

- **Custom Semgrep rules:** add a `semgrep-rules/` directory (or point
  `semgrep-custom-rules` elsewhere) with your own `.yml` rules — e.g. org-specific
  checks like CSRF validation on internal frameworks.
- **False-positive secrets:** add a `gitleaks.toml` with an `[allowlist]` section.
- **Ignoring a dependency vuln:** add an `osv-scanner.toml` in the caller repo with
  `[[IgnoredVulns]]` entries (id + reason), which OSV-Scanner picks up automatically.
- **Skipping a Checkov check:** annotate the resource with
  `#checkov:skip=CKV_XXX: <reason>`.
