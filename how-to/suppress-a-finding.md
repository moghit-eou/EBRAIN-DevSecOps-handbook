# How to suppress a false positive

Applies to the CVSS-score tools only (Trivy, OSV-Scanner), shared across
Container Scanning and SCA since both point at the same two ignore files.
OpenGrep and Hadolint findings are not suppressed through a shared ignore
file, handle those at the rule or finding level instead.

## Trivy

Add an entry to your Trivy ignore file:

```yaml
vulnerabilities:
  - id: CVE-YYYY-NNNNN
    statement: "Explain why this is safe to ignore, be specific."
    expires: YYYY-MM-DD
```

Scope it further with `paths` or `purls` if the ignore should only apply to
a specific file or package, rather than every place the CVE ID shows up.

## OSV-Scanner

Add an entry to your OSV-Scanner ignore file:

```toml
[[IgnoredVulns]]
id = "GHSA-xxxx-xxxx-xxxx"
ignoreUntil = YYYY-MM-DD
reason = "Explain why this is safe to ignore, be specific."
```

## When NOT to suppress

TODO: don't suppress instead of upgrading when a fix is available. Always
include a reason and an expiry date, never suppress permanently.