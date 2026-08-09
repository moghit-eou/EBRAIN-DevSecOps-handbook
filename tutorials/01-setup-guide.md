# Tutorial: Set Up and Run the Security Pipelines Locally

This tutorial walks through installing the scanners and running all three
security pipelines (Container Scanning, SCA, SAST) against a repository on
your own machine, start to finish. It uses generic placeholders
(`<your-service>`, `<your-image>`) instead of any specific repository name,
so it applies to any Maven or npm project set up the same way.

**Platform requirement:** this tutorial has only been verified on Linux
x86_64 (Ubuntu), matching the `ubuntu-latest` GitHub Actions runner. On
Windows, use WSL2. There is currently no working path on native macOS.
*Note: Future iterations of this pipeline will fully Dockerize the orchestrator and toolchain to guarantee native, frictionless execution across all operating systems.* See [tool-installation-flags.md](https://www.google.com/search?q=../reference/tool-installation-flags.md)
for current compatibility details.

## What you'll need

* A Linux x86_64 environment (native Ubuntu or WSL2).
* `bash`, `curl`, `sudo`, `git`.
* Docker (for the Container Scanning pipeline).
* Either Maven (for a Maven/Java project) or Node.js and npm (for an
npm/JavaScript project), matching your project's ecosystem.
* The `ci/` scripts from your repository: `setup-tools.sh`,
`container_scan.py`, `sca_scan.py`, `sast_scan.py`, `parse_sarif.py`.

## Step 1: check out the repository

```bash
git clone <your-repo-url>
cd <your-service>
```

## Step 2: Run the SAST pipeline

SAST scans your source code only. It doesn't need Docker or your
dependency manager.

```bash
bash ci/setup-tools.sh --install-tool opengrep,semgrep-rules
python ci/sast_scan.py

```

This installs OpenGrep and clones the pinned `semgrep-rules` community
ruleset, then runs OpenGrep against your source tree twice: once to write
a full SARIF report, once as the actual pass/fail gate. See
[pipeline-sast.md](https://www.google.com/search?q=../reference/pipeline-sast.md) for what each step does.

Expect a `sast-opengrep-app.sarif` file (or your project's configured
output name) in the working directory when this finishes.

## Step 3: Run the SCA pipeline

SCA scans your application's declared dependencies, via a generated SBOM,
not your source code and not a container image.

**If your project is Maven-based:**

```bash
mvn dependency:resolve -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py

```

**If your project is npm-based:**

```bash
npm ci
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py

```

Use `npm ci`, not `npm install`, before generating the SBOM. `npm ci`
installs strictly from `package-lock.json`, wipes `node_modules` first for
a clean install, and fails immediately if `package.json` and
`package-lock.json` are out of sync, instead of silently rewriting the
lockfile. If it fails, regenerate the lockfile locally with
`npm install` and commit the result, don't work around the failure inside
the pipeline.

This installs Trivy and OSV-Scanner, generates a CycloneDX SBOM at
`target/bom.json`, then scans that SBOM (not your local dependency cache)
with both tools. See [why-sboms.md](https://www.google.com/search?q=../explanation/why-sboms.md) for why
the SBOM, and not `~/.m2` or `node_modules` directly, is what gets
scanned.

## Step 4: Run the Container Scanning pipeline

Container Scanning covers two different things in one pipeline: the
Dockerfile itself (SAST-style linting) and the built image (SCA-style CVE
scanning).

```bash
docker build -t <your-image>:local .
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
python ci/container_scan.py --scan-type sast
python ci/container_scan.py --scan-type sca --image <your-image>:local

```

The `--scan-type sast` run scans the `Dockerfile` with Hadolint and
OpenGrep. The `--scan-type sca` run scans the built image with Trivy and
OSV-Scanner. Both use the same `container_scan.py` CLI, just with a
different `--scan-type`.

### Orchestrator CLI Reference

You can view all available arguments for the container orchestrator at any time using the `--help` flag. This exposes advanced pipeline functionality, such as merging multiple SARIF reports into a single consolidated artifact.

```bash
python ci/container_scan.py --help

```

**Output:**

```text
usage: sec-orchestrator [-h] [-s {sast,sca}] [-i IMAGE] [--merge-sarif SARIF_FILE [SARIF_FILE ...]] [--merge-output MERGE_OUTPUT]

Agnostic DevSecOps Container scanning Pipeline Orchestrator

options:
  -h, --help            show this help message and exit
  -s, --scan-type {sast,sca}
                        Specify the security methodology to execute (e.g., sast, sca)
  -i, --image IMAGE     Target Docker image reference
  --merge-sarif SARIF_FILE [SARIF_FILE ...]
                        List of SARIF files to merge into one report
  --merge-output MERGE_OUTPUT
                        Output path for the merged SARIF file

```

## Step 5: Read the results

Each tool writes its own SARIF file. All three orchestrator scripts print
a summary block at the end of their run, for example:

```text
========== SCA PIPELINE SUMMARY ==========
[trivy]: PASSED
[osv-scanner]: WARNING (findings between 5.0 and 8.0)
============================================

```

If you don't have the GitHub Security tab available (a local run, or a
SARIF file downloaded from workflow artifacts), open it in a SARIF viewer
such as
[Microsoft's SARIF Web Component](https://microsoft.github.io/sarif-web-component/)
instead of reading the raw JSON.

To understand what PASSED, WARNING, FAILED, and ERROR mean for each tool,
see [gate-status-cvss.md](https://www.google.com/search?q=../reference/gate-status-cvss.md) and
[gate-status-rule-severity.md](https://www.google.com/search?q=../reference/gate-status-rule-severity.md).

## Next steps

* If a finding is a false positive, see
[suppress-a-finding.md](https://www.google.com/search?q=../how-to/suppress-a-finding.md).
* If you hit a Maven registry rate limit while resolving dependencies, see
[troubleshoot-maven-rate-limit.md](https://www.google.com/search?q=../how-to/troubleshoot-maven-rate-limit.md).
* If SBOM generation itself fails, see
[troubleshoot-sbom-generation-errors.md](https://www.google.com/search?q=../how-to/troubleshoot-sbom-generation-errors.md).