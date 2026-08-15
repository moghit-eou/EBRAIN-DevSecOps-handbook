# Reference: Rule-Severity Gate

Applies to: **OpenGrep, Hadolint**, the SAST pipeline, and the SAST half
of the Container Scanning pipeline.

These tools don't report a CVSS score, there is no CVE to score, the
finding is a rule/pattern match against source code or a Dockerfile, not a
known-vulnerability lookup. Severity comes from the **rule itself**, not
from an external database.

## Rule severity levels

Both tools classify each rule at one of three levels. Only `ERROR` blocks
the pipeline by default, in this setup, `WARN` and `INFO` are recorded in
the SARIF output and visible in the Security tab, but never fail the gate:

| Level | Meaning | Blocks the gate (default config)? |
|---|---|---|
| `ERROR` | High-confidence, security-relevant pattern match (e.g. a hardcoded credential, an insecure Dockerfile instruction) | **Yes** |
| `WARN` | Lower-confidence or best-practice finding, worth surfacing but not necessarily a real vulnerability | No |
| `INFO` | Informational only, style or minor-practice notes | No |

The gate flags checked by the orchestrators (`--severity=ERROR --error`
for OpenGrep, `--failure-threshold error` for Hadolint) only look at
`ERROR`-level findings. `WARN` and `INFO` findings still appear in the full
SARIF report uploaded to the Security tab, they're visible, just not
blocking. To change what counts as blocking, see
[adjust-severity-gate.md](../how-to/adjust-severity-gate.md#rule-severity-gate-opengrep-hadolint).

## Gate outcome

| Status | Condition | Blocks the pipeline? |
|---|---|---|
| `PASSED` | No `ERROR`-severity findings | No |
| `FAILED` | One or more `ERROR`-severity findings | **Yes** |
| `ERROR` | Tool did not run correctly, or its SARIF output is missing | **Yes** |

Unlike the CVSS-score gate, there's no `WARNING` state here, a rule either
matched at blocking severity or it didn't; there's no numeric scale to sit
in between.

## Where the threshold can be changed

Each tool's own severity flag decides the gate directly, there's no shared
parsing layer equivalent to `parse_sarif.py` for this gate model. See
[adjust-severity-gate.md](../how-to/adjust-severity-gate.md#rule-severity-gate-opengrep-hadolint).
