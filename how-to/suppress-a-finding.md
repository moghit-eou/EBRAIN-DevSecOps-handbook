# How to suppress a false positive

Applies to the CVSS-score tools only (Trivy, OSV-Scanner), shared across
Container Scanning and SCA since both point at the same two ignore files.
OpenGrep and Hadolint findings are not suppressed through a shared ignore
file, handle those at the rule or finding level instead.

## Trivy

TODO: generalized suppress_trivy.yaml example, including scoping an ignore
to specific paths or packages instead of ignoring everywhere.

## OSV-Scanner

TODO: generalized suppress_osv_scanner.toml example.

## When NOT to suppress

TODO: don't suppress instead of upgrading when a fix is available. Always
include a reason and an expiry date.
