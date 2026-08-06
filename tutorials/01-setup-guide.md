# Setup guide

Goal: three independent, parallel pull request gates, one for Container
Scanning, one for SCA, one for SAST.

## Step 1: install the tools

TODO: generalize setup-tools.sh, the --install-tool flag list, and SHA256
pinning approach.

## Step 2: wire up the three workflow files

TODO: one workflow per pipeline, all triggered on pull_request,
workflow_dispatch, and a schedule.

## Step 3: set your severity thresholds

TODO: CVSS thresholds for the CVSS gate, error-severity flags for the
rule-severity gate. Point to reference/gate-status-cvss.md and
reference/gate-status-rule-severity.md.

## Step 4: verify each pipeline blocks a known bad PR

TODO: one test case per pipeline, since they gate independently.
