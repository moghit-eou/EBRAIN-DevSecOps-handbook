# Gate status, CVSS model

Applies to Trivy and OSV-Scanner, in both Container Scanning (sca) and SCA.

| Status | Meaning | Blocks pipeline |
|---|---|---|
| PASSED | Highest score below 5.0 | No |
| WARNING | Highest score 5.0 to 7.9 | No |
| FAILED | Highest score 8.0 or above | Yes |
| ERROR | Tool crashed or SARIF missing | Yes |
