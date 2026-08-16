# Reference: Python Orchestrators

The three pipelines (SAST, SCA, Container Scanning) are each driven by a
small Python script under `ci/`: `sast_scan.py`, `sca_scan.py`, and
`container_scan.py`. This page shows their functions, how SCA scanning
and SAST scanning are each handled, and where the three files overlap.

## Functions

### `sast_scan.py`

```python
def run_opengrep():
    base_cmd = ["opengrep", "scan"] + \
        [f"--config {config}" for config in SEMGREP_CONFIG_RULESETS] + \
        [f"--exclude={pattern}" for pattern in OPENGREP_EXCLUDE]

    report_cmd = (base_cmd + ["--sarif", "--output", OPENGREP_SARIF_OUTPUT])
    subprocess.run(" ".join(report_cmd).split())

    gate_cmd = (base_cmd + ["--severity=ERROR", "--error"])
    return subprocess.run(" ".join(gate_cmd).split()).returncode
```

Runs OpenGrep twice with the same base command: once to write a full
report, once with `--severity=ERROR --error` to check the gate. Only the
gate run's exit code matters for pass or fail.

### `sca_scan.py`

```python
def run_trivy():
    cmd = ["trivy", "sbom", SBOM_PATH, "--format", "sarif",
           "--ignorefile", TRIVY_IGNOREFILE, "--output", TRIVY_SARIF_OUTPUT]
    return subprocess.run(cmd).returncode

def run_osv_scanner():
    cmd = ["osv-scanner", "scan", "source", "--lockfile", SBOM_PATH,
           "--config", OSV_IGNOREFILE, "--format", "sarif",
           "--output-file", OSV_SARIF_OUTPUT]
    exit_code = subprocess.run(cmd).returncode
    if exit_code == 1:
        return 0  # 1 means "found something," not "tool failed"
    return exit_code

def merge_sarifs():
    merged = {"$schema": "...", "version": "2.1.0", "runs": []}
    for path in (TRIVY_SARIF_OUTPUT, OSV_SARIF_OUTPUT):
        if os.path.exists(path):
            with open(path) as f:
                merged["runs"].extend(json.load(f, strict=False).get("runs", []))
    with open(SCA_MERGED_SARIF_OUTPUT, "w") as f:
        json.dump(merged, f)
```

`run_trivy()` scans the SBOM file directly. `run_osv_scanner()` does the
same but always turns exit code `1` into `0`, since the real pass or fail
decision comes later, from `evaluate()`, not from this exit code.
`merge_sarifs()` concatenates both tools' SARIF `runs` arrays into one
file, for the combined artifact. It plays no part in the gate decision.

### `container_scan.py`

`run_trivy()` and `run_osv_scanner()` here are the same as in
`sca_scan.py`, just pointed at an image instead of an SBOM file:

```python
def run_trivy():
    cmd = ["trivy", "image", IMAGE_NAME, "--format", "sarif",
           "--ignorefile", TRIVY_IGNOREFILE, "--output", TRIVY_SCA_SARIF_OUTPUT]
    return subprocess.run(cmd).returncode

def run_osv_scanner():
    cmd = ["osv-scanner", "scan", "image", IMAGE_NAME,
           "--config", OSV_IGNOREFILE, "--format", "sarif",
           "--output-file", OSV_SCA_SARIF_OUTPUT]
    exit_code = subprocess.run(cmd).returncode
    if exit_code == 1:
        return 0
    return exit_code
```

Two more functions run against the `Dockerfile` instead of the image,
one linter and one SAST scanner:

```python
def run_hadolint():
    cmd = ["hadolint", "Dockerfile", "--failure-threshold", "error", "--format", "sarif"]
    with open(HADOLINT_SAST_SARIF_OUTPUT, "w") as f:
        result = subprocess.run(cmd, stdout=f)
    return result.returncode

def run_opengrep():
    base_cmd = ["opengrep", "scan", "--include=Dockerfile", "-q"] + \
        [f"--config {config}" for config in SEMGREP_CONFIG_RULESETS]
    report_cmd = base_cmd + ["--sarif", "--output", OPENGREP_SAST_SARIF_OUTPUT]
    subprocess.run(" ".join(report_cmd).split())
    gate_cmd = base_cmd + ["--severity=ERROR", "--error"]
    return subprocess.run(" ".join(gate_cmd).split()).returncode
```

`run_hadolint()` is a Dockerfile linter, it checks for bad Dockerfile
practices, not for vulnerable code patterns. It is not a SAST tool.
`run_opengrep()` is the actual SAST tool here, the same rule-based static
analysis used in `sast_scan.py`, just scoped to the `Dockerfile` with
`--include=Dockerfile`. The pipeline runs both together and gates on
both, but they are two different kinds of check.

`merge_sarifs()` here takes a list of paths instead of a fixed pair, so
it can combine any number of SARIF files, not just two:

```python
def merge_sarifs(sarif_paths, output_path):
    merged = {"$schema": "...", "version": "2.1.0", "runs": []}
    for path in sarif_paths:
        if os.path.exists(path):
            with open(path) as f:
                merged["runs"].extend(json.load(f, strict=False).get("runs", []))
    with open(output_path, "w") as f:
        json.dump(merged, f)
```

## How SCA scanning is handled

This same sequence runs inside `sca_scan.py`'s `main()`, and again inside
`container_scan.py` (there it scans an image instead of an SBOM, the
steps are otherwise identical):

```
for each tool (trivy, osv-scanner):
    run the tool, get exit_code
    if exit_code is not 0 but the tool still wrote a SARIF file:
        mark that tool ERROR (inconsistent, worth flagging on its own)

merge_sarifs()   # combined artifact only, not used for the gate

for each tool not already marked ERROR:
    if its SARIF file is missing: mark ERROR
    else: evaluate() the SARIF file
        CVSS >= 8.0 found  -> FAILED
        5.0 <= CVSS < 8.0  -> WARNING
        nothing >= 5.0     -> PASSED

print the summary
if any tool is FAILED or ERROR: exit with code 1
```

## How SAST scanning is handled

In `sast_scan.py`, only OpenGrep runs, so this logic applies once:

```
run OpenGrep, get exit_code
if exit_code is 0: status = PASSED
if exit_code is 1: status = FAILED
otherwise: status = ERROR

if the SARIF file was not written: status = ERROR (tool did not run)

print the summary
if status is ERROR or FAILED: exit with code 1
```

In `container_scan.py`, the same exit code mapping applies, but to two
tools instead of one, Hadolint (linting) and OpenGrep (SAST), run in a
loop:

```
for each tool (hadolint, opengrep):
    run the tool, get exit_code
    if exit_code is 0: status = PASSED
    if exit_code is 1: status = FAILED
    otherwise: status = ERROR

print the summary
if any tool is not PASSED: exit with code 1
```

Hadolint and OpenGrep are gated together here for practical reasons, both
are cheap checks against the same `Dockerfile`, but they are not the same
kind of tool. Only OpenGrep is SAST; Hadolint is a linter.

## How `container_scan.py` routes between them

`container_scan.py` is the only one of the three scripts with a command
line interface. Its `main()` picks between the SCA logic and the SAST
logic above based on a flag, instead of a script only ever doing one
thing:

```
parse CLI flags: --scan-type (sast or sca), --image, --merge-sarif, --merge-output

if --scan-type is sast: run the SAST scanning steps above
if --scan-type is sca:
    if --image was given, use it as the image name
    run the SCA scanning steps above

if --merge-sarif paths were given: call merge_sarifs() and stop
```

This is why `container-scan.yml` calls this one script three times in
the same job: once with `--scan-type sast`, once with `--scan-type sca`,
once with only `--merge-sarif` and no `--scan-type`.

In short, `container_scan.py` is not a fully separate script. It reuses
`sca_scan.py`'s SCA logic and `sast_scan.py`'s SAST logic, plus a linter
call and a more general `merge_sarifs()`. If you change shared behavior,
for example how OSV-Scanner's exit code is handled, check all three
files, since the same logic exists in more than one place.