# Reference: SAST (Static Application Security Testing) Pipeline

| Property | Value |
|---|---|
| Workflow file | `sast.yml` |
| Orchestrator | `ci/sast_scan.py` |
| Triggers | `pull_request`, `workflow_dispatch`, weekly schedule |
| Scans | Application source code only, not the Dockerfile, not dependencies |
| Tool | OpenGrep |
| Gate model | Rule-severity gate (see [gate-status-rule-severity.md](gate-status-rule-severity.md)) |

## Execution order

`run_opengrep()` runs the scanner **twice**, with the same base command
and different flags each time:

```mermaid
flowchart TD
    A["Checkout + Setup Python"] --> B["setup-tools.sh --install-tool opengrep,semgrep-rules"]
    B --> C["sast_scan.py"]
    C --> D["opengrep scan ... --sarif --output\n(report run, full findings)"]
    C --> E["opengrep scan ... --severity=ERROR --error\n(gate run, decides pass/fail)"]
    D --> F["Upload SARIF\ncategory: semgrep-app"]
    E --> G{"Exit code"}
    G -->|0| H["PASSED"]
    G -->|1| I["FAILED"]
    G -->|other| J["ERROR"]
    F --> K["Upload SARIF artifact\n(retained 30 days)"]
```

Run 1 always writes the complete SARIF report, regardless of severity, so
every finding is visible in the GitHub Security tab. Run 2 is the actual
gate: it only considers `ERROR`-severity findings and exits non-zero if
any are present. This split exists so that lower-severity findings are
still recorded and visible, without being able to block the build.

## Environment variables

| Variable | Purpose |
|---|---|
| `SEMGREP_CONFIG_RULESETS` | Space-separated list of OpenGrep/Semgrep rule packs to run. This is the one part of the SAST pipeline that's language-specific, adjust it to match the languages actually present in the repository |
| `OPENGREP_EXCLUDE` | Space-separated glob patterns excluded from the scan |
| `OPENGREP_SARIF_OUTPUT` | Output path for the SARIF report |

### Choosing `SEMGREP_CONFIG_RULESETS` for a new repository

Pick rule packs that match the languages actually in the repository, plus
the generic/cross-language packs. Two worked examples from the reference
implementations:

| Repository | Stack | `SEMGREP_CONFIG_RULESETS` |
|---|---|---|
| `platform-backend` | Java / Maven | `semgrep-rules/generic semgrep-rules/problem-based-packs semgrep-rules/bash semgrep-rules/java auto semgrep-rules/yaml semgrep-rules/package_managers p/default` |
| `platform-ui` | TypeScript / npm | `semgrep-rules/generic semgrep-rules/problem-based-packs semgrep-rules/bash semgrep-rules/javascript semgrep-rules/yaml semgrep-rules/package_managers p/default semgrep-rules/json` |

For a Python repository, the equivalent starting point would swap
`semgrep-rules/java` or `semgrep-rules/javascript` for `semgrep-rules/python`;
for Go, `semgrep-rules/go`. See
[why-opengrep-not-semgrep.md](../explanation/why-opengrep-not-semgrep.md#choosing-rulesets)
for the tradeoff between stacking specific packs and using `auto`.

### `OPENGREP_EXCLUDE` by repository

| Repository | `OPENGREP_EXCLUDE` |
|---|---|
| `platform-backend` | `*.sarif ci/ Dockerfile* .pre-commit-config.yaml docs/** README.md AGENTS.md` |
| `platform-ui` | `*.sarif ci/ Dockerfile* dist/** build/** node_modules/** .angular/**` |

Exclude build output and dependency directories (`node_modules/`,
`dist/`, `build/`, `target/`, `vendor/`) for any ecosystem, scanning
generated or vendored code wastes time and produces findings that can't be
fixed at the source.

## Running it locally

```bash
bash ci/setup-tools.sh --install-tool opengrep,semgrep-rules
python ci/sast_scan.py
```

This is identical for every ecosystem, OpenGrep scans source text, it has
no dependency on a build tool or package manager.

## Exit code to status mapping

| `opengrep` gate-run exit code | Orchestrator status |
|---|---|
| `0` | `PASSED` |
| `1` | `FAILED` (error-severity findings present) |
| anything else | `ERROR` (tool did not run correctly) |

If the expected SARIF output file doesn't exist after the run, the status
is forced to `ERROR` regardless of exit code, a missing report means the
tool didn't actually run, not that it found nothing.

See also: [why-opengrep-not-semgrep.md](../explanation/why-opengrep-not-semgrep.md).
