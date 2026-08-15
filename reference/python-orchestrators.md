# Reference: Python Orchestrators

The three pipelines (SAST, SCA, Container Scanning) are each driven by a
small, dependency-light Python script living under `ci/`. This page
documents `sast_scan.py`, `sca_scan.py`, and `container_scan.py`
themselves, purely as Python: their structure, their functions, and how
they compose. For what invokes them (the GitHub Actions workflow files)
see [pipeline-sast.md](pipeline-sast.md), [pipeline-sca.md](pipeline-sca.md),
and [pipeline-container-scanning.md](pipeline-container-scanning.md). For
the shared SARIF-parsing gate logic all three call into, see
[gate-status-cvss.md](gate-status-cvss.md) and
[gate-status-rule-severity.md](gate-status-rule-severity.md), which cover
`parse_sarif.py`.

## Shared shape

All three orchestrators are single-file scripts with no third-party
dependencies, only the standard library (`subprocess`, `os`, `sys`,
`logging`, `json`), plus a same-package import of `parse_sarif` for the
two that need a gate decision. Each follows the same pattern:

1. A small set of ANSI color constants (`GREEN`, `RED`, `RESET`, `BOLD`,
   `YELLOW`) for readable CI log output.
2. `logging.basicConfig(format='%(message)s')`, a bare message format
   with no timestamp, since the CI runner already timestamps every log
   line, a second timestamp from Python would just be noise.
3. A block of module-level configuration read from environment variables
   via `os.getenv(NAME, default)`, so the same script runs unmodified
   across `platform-backend` and `platform-ui`, only the environment
   differs.
4. One `run_<tool>()` function per external tool, each building an
   argument list and calling `subprocess.run(cmd).returncode`.
5. A summary block that maps tool exit codes and/or gate evaluations to a
   `PASSED` / `WARNING` / `FAILED` / `ERROR` status, prints a formatted
   summary, and calls `sys.exit(1)` if anything failed.

None of the three scripts parses CLI flags for its core scan logic
(`container_scan.py` is the one exception, see below); ecosystem or
scan-type selection happens entirely through environment variables and,
for `container_scan.py`, one `argparse` router.

## `sast_scan.py`

The simplest of the three. One function, `run_opengrep()`, runs OpenGrep
**twice** with the same base command and different flags:

- **Report run:** `--sarif --output $OPENGREP_SARIF_OUTPUT`, writes every
  finding regardless of severity, for the GitHub Security tab.
- **Gate run:** `--severity=ERROR --error`, considers only `ERROR`-level
  findings, and its exit code is what `main()` maps to
  `PASSED` / `FAILED` / `ERROR`.

Configuration read from the environment: `SEMGREP_CONFIG_RULESETS`
(space-separated rule packs, `.split()` into a list),
`OPENGREP_EXCLUDE` (space-separated glob patterns, also split into a
list), and `OPENGREP_SARIF_OUTPUT` (a single path).

`main()`'s status logic: exit code `0` → `PASSED`; `1` → `FAILED`; anything
else → `ERROR`. If the expected SARIF file is missing from disk after the
run, `main()` forces the status to `ERROR` regardless of exit code, since
a missing report means the tool didn't actually run.

## `sca_scan.py`

Structurally the most representative of the three. Three functions:

- `run_trivy()` — runs `trivy sbom $SBOM_PATH ...`, returns the raw exit
  code.
- `run_osv_scanner()` — runs `osv-scanner scan source --lockfile
  $SBOM_PATH ...`. Exit code `1` from OSV-Scanner means "vulnerabilities
  found," which this function deliberately remaps to `0` so a finding
  doesn't short-circuit the pipeline before the SARIF-based gate runs;
  any other non-zero code passes through unchanged.
- `merge_sarifs()` — reads both tools' SARIF files (skipping and warning
  on any that don't exist), concatenates their `runs` arrays into one
  SARIF document, and writes it to `SCA_MERGED_SARIF_OUTPUT`. This merged
  file is an artifact for humans; it is not what the gate evaluates.

`main()` builds two dicts, `tools` (name → function) and `sarif_files`
(name → expected output path), then:

1. Calls every tool function, flags `ERROR` if a tool exits non-zero but
   still produced a SARIF file (an inconsistent state worth flagging on
   its own).
2. Calls `merge_sarifs()`.
3. For each tool not already flagged `ERROR`, calls `parse_sarif.evaluate()`
   on its SARIF file and maps the result to `PASSED` / `WARNING` /
   `FAILED` per the CVSS thresholds (see
   [gate-status-cvss.md](gate-status-cvss.md)).
4. Prints a per-tool summary and exits `1` if any tool's status is not
   `PASSED`.

## `container_scan.py`

The largest of the three, because it does the job of both of the others
against a Docker image and a `Dockerfile` instead of source or an SBOM.
It's the only one with an `argparse` CLI (`-s/--scan-type`, `-i/--image`,
`--merge-sarif`, `--merge-output`), since a single invocation needs to
pick between two unrelated code paths:

- `handle_sca()` — the same shape as `sca_scan.py`'s `main()` body
  (Trivy + OSV-Scanner, CVSS-score gate), but against `trivy image
  $IMAGE_NAME` and `osv-scanner scan image $IMAGE_NAME` instead of an
  SBOM file.
- `handle_sast()` — runs Hadolint and OpenGrep against the `Dockerfile`
  (`opengrep scan --include=Dockerfile`), and maps each tool's own exit
  code directly to a status (no shared gate function here, since both
  tools' exit codes are already a direct pass/fail signal, see
  [gate-status-rule-severity.md](gate-status-rule-severity.md)).
- `merge_sarifs(sarif_paths, output_path)` — a generalized version of
  `sca_scan.py`'s merge function, taking an arbitrary list of SARIF paths
  instead of a fixed pair, since `container-scan.yml` needs to merge four
  files (two SCA, two SAST) rather than two.

`main()` parses arguments, dispatches to `handle_sast()` or `handle_sca()`
based on `--scan-type`, optionally overrides the module-level
`IMAGE_NAME` via `global IMAGE_NAME` if `--image` was passed, and, if
`--merge-sarif` paths were given, calls `merge_sarifs()` as a separate
step at the end. This is why `container-scan.yml` invokes the script
three times in one job (`--scan-type sast`, `--scan-type sca`, then
`--merge-sarif ...`), see
[pipeline-container-scanning.md](pipeline-container-scanning.md#execution-order).

## What's deliberately not in any of these scripts

None of the three scripts knows anything about the calling CI system.
There is no GitHub-specific API call, no `GITHUB_*` environment variable
read anywhere in any of them, every input arrives as a generic
environment variable or CLI flag. That's what makes it possible to run
any of them identically outside GitHub Actions, on any CI runner with
`python3` and the relevant scanner binaries on `PATH`, or directly on a
developer machine for local triage.
