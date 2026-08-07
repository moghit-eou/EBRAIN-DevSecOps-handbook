# Setup guide
 
Goal: three independent, parallel pull request gates, one for Container
Scanning, one for SCA, one for SAST. This tutorial walks through Container
Scanning end to end, since it is the pipeline with a complete reference
implementation so far. SCA and SAST follow the same shape.
 
## Step 1: install the tools
 
Each pipeline installs only the tools it needs via `--install-tool`:
 
```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
```
 
Tool versions and their release-asset checksums are pinned inside the
script itself, not passed here, see
[Tool installation flags](../reference/tool-installation-flags.md).
 
## Step 2: build what you're scanning
 
Container Scanning scans a built image, not source. Build it first:
 
```bash
docker build -t <your-image>:local .
``` 
## Step 3: set your severity thresholds

TODO: CVSS thresholds for the CVSS gate, error-severity flags for the
rule-severity gate. Point to reference/gate-status-cvss.md and
reference/gate-status-rule-severity.md.

## Step 4: verify each pipeline blocks a known bad PR

TODO: one test case per pipeline, since they gate independently.
