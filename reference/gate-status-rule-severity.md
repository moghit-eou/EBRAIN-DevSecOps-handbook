# Reference: Rule-Severity Gate
Todo: mention the types of rules ( error , warn , info ..)
---
Applies to: **OpenGrep, Hadolint**, the SAST pipeline, and the SAST half
of the Container Scanning pipeline.

These tools don't report a CVSS score, there is no CVE to score, the
finding is a pattern match against source code or a Dockerfile
instruction. Each tool decides pass/fail directly from its own severity
flag and exit code, `parse_sarif.evaluate()` is not used here.

## Thresholds

| Status | Condition | Blocks the pipeline? |
|---|---|---|
| `PASSED` | No error-severity findings | No |
| `FAILED` | Error-severity findings present | **Yes** |
| `ERROR` | Tool did not run correctly | **Yes** |

There is no `WARNING` state in this model, unlike the CVSS-score gate.
Either a tool passes, or it finds an error-severity issue and fails, or it
didn't run correctly.

## Per-tool flags that decide the gate

| Tool | Flag(s) | Effect |
|---|---|---|
| OpenGrep | `--severity=ERROR --error` | Only counts `ERROR`-severity findings; exits non-zero if any exist |
| Hadolint | `--failure-threshold error` | Only counts `error`-level findings; exits non-zero if any exist |

## OpenGrep's two-run pattern

`run_opengrep()` runs the same base command twice, once to always produce
the full SARIF report (all severities, for visibility in the Security
tab), once purely as the pass/fail gate (`ERROR` severity only). The gate
result comes from the second run's exit code, not from inspecting the
first run's SARIF output.

## Used by

| Orchestrator | Uses this model for |
|---|---|
| `sast_scan.py` | OpenGrep against source code |
| `container_scan.py --scan-type sast` | Hadolint and OpenGrep against the `Dockerfile` |

`container_scan.py --scan-type sca` and `sca_scan.py` use the CVSS-score
model instead, see [gate-status-cvss.md](gate-status-cvss.md). The two
models are never mixed for the same tool. See
[why-two-gate-models.md](../explanation/why-two-gate-models.md).
