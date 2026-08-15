# How to Add a New Scanner to a Pipeline

This walks through wiring a new tool into an existing pipeline, following
the same pattern already used by Trivy, OSV-Scanner, OpenGrep, and
Hadolint. There is no dedicated plugin system, adding a tool means
touching three files: `setup-tools.sh`, the relevant orchestrator script,
and the workflow YAML.

## 1. Add installation to `setup-tools.sh`

Every existing tool follows the same shape: a pinned version, a pinned
SHA256 checksum (with a `# renovate:` marker so Renovate bumps both
together), downloaded via `download_and_verify`, gated behind
`should_install "<tool-name>"`:

```bash
# renovate: datasource=github-release-attachments depName=<org>/<tool>
NEWTOOL_VERSION="${NEWTOOL_VERSION:-vX.Y.Z}"
NEWTOOL_SHA256="${NEWTOOL_SHA256:-<sha256-of-the-release-asset>}"

if should_install "newtool"; then
  echo "[setup-tools] Installing NewTool ${NEWTOOL_VERSION}"
  download_and_verify \
    "https://github.com/<org>/<tool>/releases/download/${NEWTOOL_VERSION}/<asset-name>" \
    "${TMP_DIR}/newtool" \
    "${NEWTOOL_SHA256}"
  sudo install -m 0755 "${TMP_DIR}/newtool" /usr/local/bin/newtool
  newtool --version
  echo "NewTool installed OK"
fi
```

Do not skip the SHA256 pin. Every existing tool in this setup verifies its
download before installing it; a new tool should follow the same practice.

## 2. Add a runner function to the orchestrator

Each orchestrator (`sca_scan.py`, `sast_scan.py`, `container_scan.py`)
follows the same shape: a `run_<tool>()` function that builds a command
list, runs it via `subprocess.run(...)`, and returns the exit code. Which
orchestrator you add to depends on what the new tool scans, source code,
dependencies, or a container image, not on the tool's name.

```python
def run_newtool():
    cmd = [
        "newtool", "scan",
        "--format", "sarif",
        "--output", NEWTOOL_SARIF_OUTPUT,
    ]
    return subprocess.run(cmd).returncode
```

Then register it in the `tools` dict alongside the existing entries, so it
participates in the same summary-printing and gate-evaluation loop.

## 3. Decide which gate model it uses

- If the tool reports a CVSS-style `security-severity` score in its SARIF
  output (like Trivy and OSV-Scanner), it can reuse
  `parse_sarif.evaluate()` unchanged, see
  [gate-status-cvss.md](../reference/gate-status-cvss.md).
- If it reports its own pass/fail via exit code and a severity flag (like
  OpenGrep and Hadolint), don't route it through `parse_sarif.evaluate()`,
  let its own exit code decide the status directly, see
  [gate-status-rule-severity.md](../reference/gate-status-rule-severity.md).

Mixing the two models for one tool is what the upstream OWASP PR
([#107](https://github.com/OWASP/DevSecOpsGuideline/pull/107)) specifically
argues against, keep them separate.

## 4. Wire the SARIF upload in the workflow YAML

```yaml
- name: Upload NewTool SARIF to GitHub Security tab
  id: upload_newtool
  if: always()
  uses: github/codeql-action/upload-sarif@<pinned-sha>
  with:
    sarif_file: ${{ env.NEWTOOL_SARIF_OUTPUT }}
    category: newtool-<scan-type>
```

Use `if: always()` so the SARIF still uploads even if an earlier step in
the job failed, that's how existing steps ensure a partial pipeline
failure doesn't hide results. Give it a unique `category`, GitHub's Code
Scanning upload treats duplicate categories from the same job as
conflicting uploads and will reject the second one.

## 5. Add it to the merge step, if the pipeline merges SARIF files

`container_scan.py --merge-sarif` and the merge logic in `sca_scan.py`
simply concatenate the `runs` array from each SARIF file into one merged
artifact. Add the new tool's output path to that list.
