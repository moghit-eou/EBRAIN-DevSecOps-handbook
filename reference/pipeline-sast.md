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

## Workflow steps

`sast.yml` is the shortest of the three pipelines: it has no
ecosystem-specific step at all, only a language-specific *value*
(`SEMGREP_CONFIG_RULESETS`), so the workflow file itself never changes
between repositories, only the `env:` block does:

| # | Step | Ecosystem-specific? | Notes |
|---|---|---|---|
| 1 | `actions/checkout` | No | Standard checkout |
| 2 | `actions/setup-python` | No | The orchestrator (`sast_scan.py`) is Python |
| 3 | `setup-tools.sh --install-tool opengrep,semgrep-rules` | No | Installs OpenGrep and clones the pinned `semgrep-rules` ruleset |
| 4 | `python ci/sast_scan.py` (report run) | No | Writes the full SARIF report, every finding, any severity |
| 5 | `python ci/sast_scan.py` (gate run) | No | Same scan, `--severity=ERROR --error`, decides pass/fail |
| 6 | `upload-sarif` + `upload-artifact` | No | One category (`semgrep-app`), plus the retained artifact |

There is no dependency-install step and no cache-restore step in this
pipeline, OpenGrep reads source files directly; it never needs a resolved
dependency tree the way the SCA pipeline's SBOM generation does.

## Caching

There is nothing to cache in the sense the SCA pipeline means it, no
dependency store, no `node_modules`, no `.m2`, this pipeline never installs
or resolves dependencies. The one thing worth caching, if `sast.yml` runs
frequently (every PR, on a large monorepo), is the `semgrep-rules` clone
itself, so `setup-tools.sh` doesn't re-clone the same pinned community
ruleset on every run:

```yaml
- name: Cache semgrep-rules checkout
  uses: actions/cache@v6
  with:
    path: <path used by setup-tools.sh to clone semgrep-rules, e.g. .cache/semgrep-rules>
    key: ${{ runner.os }}-semgrep-rules-${{ env.SEMGREP_RULES_REF }}
    restore-keys: ${{ runner.os }}-semgrep-rules-
```

Key this on the pinned commit/ref (`SEMGREP_RULES_REF` in
`setup-tools.sh`), not a lockfile, since there is no lockfile here, the
"version" being cached is the ruleset's own pinned git ref. This is
optional: it only matters for scan-time performance, it has no effect on
gate correctness, unlike the SCA pipeline's dependency cache where an
incorrectly-keyed cache is a correctness bug (see
[pipeline-sca.md](pipeline-sca.md#caching)).

## Generic GitHub Actions template

Unlike the SCA and Container Scanning pipelines, this template needs no
ecosystem substitution at all, only the ruleset value changes per
repository:

```yaml
name: Static Application Security Testing (SAST)

on:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'

jobs:
  sast:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    env:
      # --- the only line that changes per repository/language ---
      SEMGREP_CONFIG_RULESETS: >-
        semgrep-rules/generic semgrep-rules/problem-based-packs semgrep-rules/bash
        semgrep-rules/<your-language> semgrep-rules/yaml semgrep-rules/package_managers
        p/default
      # -------------------------------------------------------------
      OPENGREP_EXCLUDE: >-
        *.sarif ci/ Dockerfile* dist/** build/** node_modules/** target/** vendor/**
      OPENGREP_SARIF_OUTPUT: sast-opengrep-app.sarif

    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v6
        with:
          python-version: '3.14.4'

      - name: Setup tools
        run: bash ci/setup-tools.sh --install-tool opengrep,semgrep-rules

      - name: Run SAST scan
        run: python ci/sast_scan.py

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.OPENGREP_SARIF_OUTPUT }}
          category: semgrep-app

      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: sast-scan-sarif-report
          path: ${{ env.OPENGREP_SARIF_OUTPUT }}
          retention-days: 30
```

> **Note on Action versions:** pin every `uses:` to a full commit SHA in a
> real workflow rather than the floating `@v7`-style tags shown here for
> readability, see [platform-ui.md](../case-studies/platform-ui.md) for a
> real, SHA-pinned example.
>
> **Not limited to Java or TypeScript.** `semgrep-rules/java` and
> `semgrep-rules/javascript` are what the two reference implementations
> use; OpenGrep ships packs for Python, Go, Ruby, PHP, C/C++, Rust, Kotlin,
> Terraform, and many more, swap in whichever pack (or packs) matches your
> repository's actual language(s). Nothing else in this file changes.

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
