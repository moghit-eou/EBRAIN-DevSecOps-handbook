# Why two gate models

TODO: why CVE-reporting tools (Trivy, OSV-Scanner) and rule-based tools
(OpenGrep, Hadolint) cannot share one gate model. CVSS score has no
equivalent in a linter or static analysis rule, and forcing one model onto
both would mean inventing a fake severity score for rule findings or
discarding real CVSS data.
