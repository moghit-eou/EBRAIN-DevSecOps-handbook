
# How to Suppress a False Positive or Accepted-Risk Finding for SCA

*Note: The vulnerabilities, dates, and packages shown in the configurations below are just examples.*

- This applies to the **CVSS-score tools** (Trivy, OSV-Scanner) only.
Suppression is shared between the Container Scanning and SCA pipelines,
since both point at the same two ignore files. 
- Only Suppressing findings in the event of false positives or noisy alerts that do not reflect actual risk.
## 1. Suppress a Trivy finding

Edit `ci/suppress_trivy.yaml`:

```yaml
vulnerabilities:
  # Non-reachable code path
  - id: CVE-2026-54515
    statement: "Vulnerable code path is not reachable: affected function is dead code in our build."
    expires: 2026-09-30

  # Low severity, accepted risk
  - id: CVE-2025-11111
    statement: "Low severity; affects an optional dev-only dependency not shipped in production images. Risk accepted, see SEC-5678."
    expires: 2026-10-15

  # Scoped ignore: paths limits it to specific files, purls limits it to
  # specific packages by PURL. Without either, the ignore applies to every
  # file/package where this id shows up.
  - id: CVE-2024-33333
    paths:
      - "test/fixtures/legacy-bundle.jar"
    purls:
      - "pkg:maven/org.example/legacy-lib"
    statement: "Only present in test fixtures; not part of the shipped artifact."
    expires: 2026-11-01

```

Always include a `statement` explaining why, and an `expires` date so the
suppression doesn't silently persist forever. Full reference:
[Trivy filtering and ignore files](https://trivy.dev/docs/latest/configuration/filtering/#trivyignoreyaml).

## 2. Suppress an OSV-Scanner finding

Edit `ci/suppress_osv_scanner.toml`:

```toml
[[IgnoredVulns]]
id = "GHSA-5jmj-h7xm-6q6v"
ignoreUntil = 2026-09-30
reason = "Vulnerable function is never called."

[[IgnoredVulns]]
id = "GHSA-9jx5-6pgf-crrp"
ignoreUntil = 2026-10-15
reason = "Low severity DoS in a dev-only tool, not present in production build. Risk accepted by security team."

```

Only use the accepted-risk pattern for LOW/MEDIUM severity findings with
limited impact. Full reference:
[OSV-Scanner: ignore vulnerabilities by ID](https://google.github.io/osv-scanner/configuration/#ignore-vulnerabilities-by-id).

