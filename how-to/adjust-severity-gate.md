# How to Adjust the Severity Gate

This covers the two different gate models used by these pipelines. Confirm
which one applies to the tool you're adjusting before you start, see
[explanation/why-two-gate-models.md](../explanation/why-two-gate-models.md)
if you're not sure why there are two.

## CVSS-score gate (Trivy, OSV-Scanner)

The thresholds live in `ci/parse_sarif.py`, inside `evaluate()`:

```python
return EvaluationResult(
        gate_failed=max_score >= 8,
        gate_warn=5 <= max_score < 8,
    )
```

This is currently **hardcoded**, not exposed as a config option. To change
the fail or warn threshold, edit these two lines directly in
`parse_sarif.py`. This file is shared by `sca_scan.py` and
`container_scan.py` (the SCA half), so a change here affects both
pipelines identically, they cannot currently be tuned independently.

After editing, re-run the affected pipeline and confirm the new threshold
is reflected in the printed summary:

```bash
python ci/sca_scan.py
```

See [reference/gate-status-cvss.md](../reference/gate-status-cvss.md) for
what the current default thresholds mean before changing them.

TODO: exposing these thresholds as a CLI flag or environment variable
instead of a hardcoded constant is not yet implemented. If you need
per-repository thresholds, that's the gap to close, there is currently no
supported way to do it without editing `parse_sarif.py` directly.

## Rule-severity gate (OpenGrep, Hadolint)

These tools don't go through `parse_sarif.py` for the pass/fail decision,
each tool's own severity flag decides directly. Adjust the gate by
changing the flag passed to the tool, not by editing a shared file.

**OpenGrep**, in `sast_scan.py` or `container_scan.py`:

```python
gate_cmd = (base_cmd + ["--severity=ERROR", "--error"])
```

Change `--severity=ERROR` to a different OpenGrep severity level to change
what counts as a blocking finding. See OpenGrep's own documentation for
the full list of severity levels it supports.

**Hadolint**, in `container_scan.py`:

```python
cmd = [
    "hadolint", "Dockerfile",
    "--failure-threshold", "error",
    "--format", "sarif",
]
```

Change `--failure-threshold error` to a different Hadolint threshold
(for example `warning`) to change what blocks the pipeline.

See
[reference/gate-status-rule-severity.md](../reference/gate-status-rule-severity.md)
for what the current defaults mean.

