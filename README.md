# Reusable Security Gate

A reusable GitHub Actions workflow that any project can call to scan for security
breaches, leaked secrets, vulnerable dependencies, and insecure infrastructure code.
It adapts automatically to the languages present in the calling repository.

## What it runs

| Job | Tool | What it catches | Gate behavior |
|---|---|---|---|
| `secrets-scan` | [Gitleaks](https://github.com/gitleaks/gitleaks) | Secrets/credentials in the full git history | **Any** finding fails (a leaked secret is always critical) |
| `sast` | [Semgrep](https://semgrep.dev) | Code vulnerabilities — OWASP Top 10, injection, crypto misuse, etc. | Fails at/above `fail-on-severity` |
| `sca` | [OSV-Scanner](https://github.com/google/osv-scanner) | Known CVEs in dependencies (npm, pip, Go, Maven, Cargo, Composer, NuGet, RubyGems, …) | Fails when CVSS ≥ threshold; unrated vulns fail conservatively |
| `iac` | [Checkov](https://www.checkov.io) | Terraform, CloudFormation, Kubernetes, Helm, Dockerfile misconfigurations | Runs only when IaC is detected; any failed check fails |
| `gate` | — | Aggregates everything into one required status check | Fails if any enabled scan failed |

Language support is automatic: Semgrep's rulesets and OSV-Scanner's manifest detection
cover Python, JavaScript/TypeScript, Go, Java, Ruby, PHP, Rust, C#, and more — the
`detect` job reports what it found in the run summary.

## Usage

In the calling repository, create `.github/workflows/security.yml`
(a ready-to-copy version lives in [`examples/security.yml`](examples/security.yml)):

```yaml
name: Security

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: security-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  security-events: write   # lets scan results appear in the repo's Security tab
  actions: read            # needed by the SARIF upload on private repos

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

### Permissions

The `permissions` block above is the complete set the gate needs. A called workflow
can only *reduce* the caller's grant, so this repo does not request permissions on
the scan jobs (asking for `security-events: write` when the caller withheld it would
fail the job). The `detect` and `gate` jobs do drop to `contents: read` and none
respectively. If you set `upload-sarif: false` you can remove `security-events: write`
and `actions: read`.

## Inputs

| Input | Default | Description |
|---|---|---|
| `scan-path` | `.` | Path within the caller repo to scan. Applies to Semgrep (scan mode), OSV-Scanner and Checkov. Gitleaks always scans the whole repository history, and `semgrep ci` (logged-in mode) always scans the whole checkout. |
| `fail-on-severity` | `high` | Minimum severity that fails the gate: `critical`, `high`, `medium`, `low` |
| `semgrep-rulesets` | `p/default p/owasp-top-ten p/secrets` | Semgrep registry rulesets (ignored when `SEMGREP_APP_TOKEN` is set — the platform's policies are used instead) |
| `semgrep-custom-rules` | `semgrep-rules` | Path in the caller repo to extra Semgrep rules; used only if it exists (ignored when `SEMGREP_APP_TOKEN` is set) |
| `gitleaks-config` | `gitleaks.toml` | Path in the caller repo to a custom Gitleaks config; used only if it exists |
| `enable-secrets-scan` | `true` | Toggle the Gitleaks job |
| `enable-sast` | `true` | Toggle the Semgrep job |
| `enable-sca` | `true` | Toggle the OSV-Scanner job |
| `enable-iac` | `true` | Toggle the Checkov job (it also auto-skips when no IaC is detected) |
| `upload-sarif` | `true` | Upload SARIF to GitHub code scanning (degrades gracefully if the repo lacks Advanced Security or the permission) |

Inputs are never interpolated into shell scripts. Each `run:` step receives them
through `env:` and reads them as quoted variables, so a value containing shell
metacharacters cannot change what the step executes.

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

OSV-Scanner only accepts one output format per run, so the `sca` job runs it twice
(JSON for the CVSS gate, SARIF for code scanning). The SARIF run is best-effort and
emits a warning if it fails.

## Supply chain

The `trivy-action`/`setup-trivy` marketplace wrappers were compromised in a
March 2026 supply-chain attack via moving tags. This repo avoids that class of risk:

| Component | How it is pinned | Where |
|---|---|---|
| `actions/checkout`, `actions/upload-artifact`, `github/codeql-action/upload-sarif` | Full commit SHA, with the release tag in a trailing comment | every `uses:` line |
| Gitleaks | Fixed version **and** SHA-256 of the release tarball, verified before extraction | `env.GITLEAKS_VERSION` / `env.GITLEAKS_SHA256` |
| OSV-Scanner | Fixed version **and** SHA-256 of the release binary, verified before use | `env.OSV_SCANNER_VERSION` / `env.OSV_SCANNER_SHA256` |
| Semgrep | Official image at a release tag **and** its image digest | `jobs.sast.container.image` |
| Checkov | Fixed version from PyPI (`pipx install checkov==X`) | `env.CHECKOV_VERSION` |

Downloads use `curl --proto '=https' --tlsv1.2` so they cannot be downgraded to
plain HTTP.

### Bumping a version

1. **Gitleaks:** pick the release on
   [gitleaks/gitleaks/releases](https://github.com/gitleaks/gitleaks/releases), open
   `gitleaks_<ver>_checksums.txt`, copy the line for `gitleaks_<ver>_linux_x64.tar.gz`.
2. **OSV-Scanner:** pick the release on
   [google/osv-scanner/releases](https://github.com/google/osv-scanner/releases), open
   `osv-scanner_SHA256SUMS`, copy the line for `osv-scanner_linux_amd64`.
3. **Semgrep:** find the tag on [Docker Hub](https://hub.docker.com/r/semgrep/semgrep/tags)
   and copy the index digest (the `sha256:` shown for the tag, not for a single
   architecture).
4. **Checkov:** pick the version from [PyPI](https://pypi.org/project/checkov/).
5. **Actions:** resolve the release tag to its commit, e.g.
   `gh api repos/actions/checkout/git/ref/tags/v7.0.1 -q .object.sha`
   (if the object type is `tag`, dereference it once more via `git/tags/<sha>`).

Dependabot is configured in [`.github/dependabot.yml`](.github/dependabot.yml) to keep
the action SHAs current: once a week it opens a single grouped PR that bumps the
SHA-pinned `uses:` lines and rewrites the version comment. It cannot update the
scanner binaries or the Semgrep image, so steps 1–4 above stay manual.

## Customizing per repo

- **Custom Semgrep rules:** add a `semgrep-rules/` directory (or point
  `semgrep-custom-rules` elsewhere) with your own `.yml` rules — e.g. org-specific
  checks like CSRF validation on internal frameworks.
- **False-positive secrets:** add a `gitleaks.toml` with an `[allowlist]` section.
- **Ignoring a dependency vuln:** add an `osv-scanner.toml` in the caller repo with
  `[[IgnoredVulns]]` entries (id + reason), which OSV-Scanner picks up automatically.
- **Skipping a Checkov check:** annotate the resource with
  `#checkov:skip=CKV_XXX: <reason>`.

## Changes

See [CHANGELOG.md](CHANGELOG.md) for the review findings behind the current hardening
and the history of changes.
