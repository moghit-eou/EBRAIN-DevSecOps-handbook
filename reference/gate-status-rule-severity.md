# Gate status, rule-severity model

Applies to OpenGrep and Hadolint, in both Container Scanning (sast) and
SAST. These tools do not report CVSS, each tool's own severity threshold
decides the status directly.

| Status | Meaning | Blocks pipeline |
|---|---|---|
| PASSED | No error-severity findings | No |
| FAILED | Error-severity findings present | Yes |
| ERROR | Tool did not run correctly | Yes |
