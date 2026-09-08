# Changelog

## Unreleased — 2026-09-08 security review

A review of `reusable-security-gate.yml` produced the findings below. Each was
checked against the file and, where it mattered, against the upstream tool source.

### Findings

| # | Finding | Verdict | Resolution |
|---|---|---|---|
| 1 | README claimed all scanners were pinned. Only Gitleaks had a fixed version; OSV-Scanner used `releases/latest`, the Semgrep image had no tag, Checkov was `pipx install checkov`. No checksums anywhere. | **Confirmed** | All four pinned; Gitleaks and OSV-Scanner also checksum-verified; Semgrep pinned by tag + digest. README rewritten to describe what is actually done. |
| 2 | Third-party actions used mutable tags (`actions/checkout@v7`, `upload-sarif@v4`). Moving tags were exactly the trivy-action attack vector the README cites. | **Confirmed, and one more:** `actions/upload-artifact@v4` was also unpinned (and two majors behind). | All `uses:` lines pinned to full commit SHAs with the tag in a comment. `upload-artifact` bumped to v7.0.1. |
| 3 | Checkov report lost on failure: the step runs under `bash -e`, Checkov exits 1 on findings, so the `mv results.sarif checkov.sarif` never ran. | **Confirmed, but the real bug was earlier.** With `-o cli -o sarif --output-file-path console,.` Checkov maps paths positionally and tries to write the SARIF to the literal path `.`. That raises `IsADirectoryError`, which Checkov logs and ignores. `results.sarif` was **never** written, pass or fail, and the `\|\| true` on the `mv` hid it. The Checkov SARIF has never reached code scanning. | `--output-file-path console,checkov.sarif`; the `mv` is gone. Checkov writes the file before exiting non-zero, so the `always()` upload steps find it. |
| 4 | `${{ inputs.* }}` interpolated directly into `run:` scripts (about twenty sites, including an unquoted `for r in ${{ inputs.semgrep-rulesets }}`). Low practical severity in a reusable workflow, since inputs come from the caller's own workflow file, but wrong for a security-tooling repo. | **Confirmed** | Every input and every `needs.*.outputs` value now goes through `env:` and is read as a quoted shell variable. Word-splitting that is intentional (rulesets, severity flags) uses `read -ra` into arrays. |
| 5a | No per-job `permissions:`; every job inherited `security-events: write`. | **Confirmed, with a constraint** | `detect` drops to `contents: read`, `gate` to none. The scan jobs deliberately stay unset: a called workflow may only reduce the caller's grant, and requesting `security-events: write` when the caller withheld it fails the job instead of degrading gracefully. The README now documents the exact minimal caller grant. |
| 5b | No `concurrency` group. | **Confirmed** | Belongs in the caller; added to `examples/security.yml` and the README example. |
| 5c | Does `actions/checkout` work inside the `semgrep/semgrep` container (Node actions fail on musl/Alpine)? | **Not an issue** | The image is Alpine 3.23, runs as root, and ships git and bash. The GitHub runner provides a musl Node build for Alpine containers, and this is Semgrep's own documented GitHub Actions setup. Left as is, with a comment. |

Additional issues found during the review:

| # | Finding | Resolution |
|---|---|---|
| 6 | `scan-path` was documented as applying to everything, but Gitleaks scans the whole repo history and `semgrep ci` scans the whole checkout (it also ignores `semgrep-rulesets` and `semgrep-custom-rules`). | Documented in the input descriptions, README, and inline comments. |
| 7 | OSV-Scanner ran twice (one output format per run) and the second run's failure was swallowed by `\|\| true`. | Still runs twice (unavoidable with OSV-Scanner v2), now commented, and the SARIF run emits a `::warning::` with its exit code if it fails. |
| 8 | Stack detection piped `find`/`grep` into `grep -q`. Under `pipefail` an early exit on the consumer side can surface as a failure on the producer side and turn a match into a miss. | Detection helpers use `$(... \| head -n1)` inside `[ -n ... ]` so only the test's exit status matters. Relevant now that `pipefail` is on everywhere. |
| 9 | Download commands did not force HTTPS/TLS and had no retry. | `curl --proto '=https' --tlsv1.2 --retry 3` on both downloads. |
| 10 | Scanner binaries were extracted into the workspace root. | Installed under `$RUNNER_TEMP` and added to `PATH` via `$GITHUB_PATH`. |
| 11 | Version pins were scattered (one in `env`, others inline). | All versions and checksums live in the top-level `env` block, except the Semgrep image which must stay on the `container:` line because the `env` context is unavailable there. |
| 12 | The default `run:` shell was `bash -e` without `pipefail`. | `defaults.run.shell: bash` (which GitHub runs as `bash -eo pipefail`) for every job, including the Semgrep container. |

### Changes

- `.github/workflows/reusable-security-gate.yml`
  - Pin `actions/checkout` to `3d3c42e5aac5ba805825da76410c181273ba90b1` (v7.0.1),
    `github/codeql-action/upload-sarif` to `cdf488f595d80d6e07e03d4674febd5ab45fa938` (v4.37.9),
    `actions/upload-artifact` to `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` (v7.0.1).
  - Pin Gitleaks 8.30.1 with SHA-256, OSV-Scanner 2.5.1 with SHA-256, Semgrep
    `1.176.0@sha256:12672acd…`, Checkov 3.3.16.
  - Fix Checkov SARIF output path; remove the `mv`.
  - Move all `${{ inputs.* }}` / `${{ needs.* }}` references out of `run:` into `env:`.
  - `permissions: contents: read` on `detect`, `permissions: {}` on `gate`.
  - `defaults.run.shell: bash`; hardened `curl`; scanners installed to `$RUNNER_TEMP`.
  - Pipefail-safe stack detection; OSV SARIF run warns instead of silently failing.
  - Comments documenting the supply-chain model and the `scan-path` exceptions.
- `.github/dependabot.yml` (new): weekly grouped Dependabot PRs for the SHA-pinned
  actions. Scanner binaries and the Semgrep image are outside Dependabot's reach and
  stay manual.
- `examples/security.yml`: add `concurrency`, annotate the permission block and the
  `@main` reference.
- `README.md`: accurate pinning description, new **Supply chain** section with
  bump instructions, **Permissions** section, `scan-path` caveats, link to this file.

### Considered and not done

- **Checkov `github_actions` framework.** It would lint the caller's own workflows
  (unpinned actions, `pull_request_target` misuse). Every caller has workflows, so it
  would change `has_iac` semantics and add noise for existing callers. Worth adding
  as an opt-in input later.
- **Single OSV-Scanner run.** Would require generating SARIF from the JSON output
  ourselves. Not worth the maintenance for one extra API call.

## e74dfe8 — initial release

Complete reusable security gate workflow: Gitleaks, Semgrep, OSV-Scanner, Checkov,
and an aggregating `gate` job.
