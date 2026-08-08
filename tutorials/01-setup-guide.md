# Tutorial: Set Up the Security Tooling Locally

This tutorial walks through installing every scanner used by the three
pipelines, and running each pipeline's orchestrator script locally,
against a generic service. By the end you will have run Container
Scanning, SCA, and SAST once each and read a gate result.

This tutorial uses generic placeholders (`<your-image>`, `<your-service>`)
throughout. Replace them with your own values, don't copy names from
`platform-backend` here, see
[case-studies/platform-backend.md](../case-studies/platform-backend.md) for
a concrete worked example instead.

## Before you start

**Platform requirement:** this has only been tested on Linux x86_64 /
Ubuntu, matching the `ubuntu-latest` GitHub Actions runner. On Windows, use
WSL2 (a real Linux userspace). There is currently no working path for
macOS.

You will need:

- `bash`
- `curl`
- `sudo` access (the install script installs binaries to `/usr/local/bin`)
- Python 3.14.4 (the version pinned in the CI workflows)
- `git`
- Docker, if you intend to run the Container Scanning steps
- Either Maven (for a Maven/Java service) or npm (for an npm/JavaScript
  service), depending on what `<your-service>` is

## Step 1: Get the CI scripts

The scripts and workflow files live under `ci/` and `.github/workflows/`
(or an equivalent layout, adapted from the `platform-backend` and
`platform-ui` reference implementations):

```
ci/
├── setup-tools.sh
├── container_scan.py
├── sca_scan.py
├── sast_scan.py
├── parse_sarif.py
├── suppress_trivy.yaml
└── suppress_osv_scanner.toml
```

Copy these into `<your-service>/ci/`.

## Step 2: Install the scanners

`setup-tools.sh` installs whichever tools you ask for, each pinned to a
specific version with a SHA256 checksum it verifies before installing:

```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
```

You can install a subset instead. See
[reference/tool-installation-flags.md](../reference/tool-installation-flags.md)
for the full flag reference, including what each tool needs and where it
is used.

Confirm each tool is on your `PATH`:

```bash
trivy --version
osv-scanner --version
opengrep --version
hadolint --version
```

## Step 3: Run the SAST pipeline

SAST scans your source code with OpenGrep. No SBOM, no Docker image
required.

```bash
python ci/sast_scan.py
```

This runs OpenGrep twice: once to produce a full SARIF report, once as the
actual pass/fail gate (`--severity=ERROR --error`). Read the printed
summary block; a `FAILED` status means error-severity findings are
present. See
[reference/gate-status-rule-severity.md](../reference/gate-status-rule-severity.md)
for what each status means.

## Step 4: Run the SCA pipeline

SCA scans your **dependencies**, via a generated SBOM, not your source
code and not a container.

If `<your-service>` is a Maven project:

```bash
mvn dependency:resolve -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py
```

If `<your-service>` is an npm project:

```bash
npm ci
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py
```

Use `npm ci`, not `npm install`, before generating the SBOM. `npm ci`
installs strictly from `package-lock.json` and fails immediately if
`package.json` and `package-lock.json` are out of sync, instead of quietly
rewriting the lockfile. If it fails here, your lockfile is stale, run
`npm install` locally and commit the updated lockfile, don't work around
it in the pipeline.

Both commands generate an SBOM at `target/bom.json` and then run Trivy and
OSV-Scanner against it. Read the summary block for a `PASSED`, `WARNING`,
or `FAILED` status per tool, see
[reference/gate-status-cvss.md](../reference/gate-status-cvss.md) for what
each threshold means, and
[explanation/why-sboms.md](../explanation/why-sboms.md) for why scanning
the SBOM instead of your local dependency cache matters.

## Step 5: Run the Container Scanning pipeline

Container Scanning covers two different things: the Dockerfile itself
(SAST-style, via Hadolint and OpenGrep) and the built image (SCA-style, via
Trivy and OSV-Scanner).

```bash
docker build -t <your-image>:local .
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
python ci/container_scan.py --scan-type sast
python ci/container_scan.py --scan-type sca --image <your-image>:local
```

Both scan types can be run independently; run `sast` first if you want to
catch Dockerfile issues before spending time building and scanning the
image.

## Step 6: Merge and inspect the SARIF output

Each tool writes its own SARIF file. To merge them into one artifact (the
same thing `container-scan.yml` does after uploading each file
individually):

```bash
python ci/container_scan.py \
  --merge-sarif sca-trivy-container.sarif sca-osv-container.sarif \
                sast-opengrep-dockerfile.sarif sast-hadolint-dockerfile.sarif \
  --merge-output container-scan-merged.sarif
```

SARIF files are plain JSON. To browse one without a GitHub Security tab
(for example, one downloaded from workflow artifacts), open it in a SARIF
viewer such as
[Microsoft's SARIF Web Component](https://microsoft.github.io/sarif-web-component/).

## What you should have at this point

- All five scanners installed and verified.
- A SAST run against your source code, with a pass/fail result.
- An SCA run against a generated SBOM, with a per-tool pass/warn/fail
  result.
- A Container Scanning run covering both the Dockerfile and the built
  image.
- A merged SARIF artifact you can inspect locally.

## Next steps

- Wire this into GitHub Actions: see
  [reference/pipeline-container-scanning.md](../reference/pipeline-container-scanning.md),
  [reference/pipeline-sca.md](../reference/pipeline-sca.md), and
  [reference/pipeline-sast.md](../reference/pipeline-sast.md) for the
  exact workflow YAML this tutorial's commands are drawn from.
- Need to suppress a specific false positive instead of re-running
  everything? See
  [how-to/suppress-a-finding.md](../how-to/suppress-a-finding.md).
- Want to understand why the gate logic is split into two models instead
  of one? See
  [explanation/why-two-gate-models.md](../explanation/why-two-gate-models.md).

TODO: this tutorial assumes a Linux host with `sudo`. Add a note (or a
separate tutorial) once the dockerized-toolchain idea (running everything
via `docker run` against a published GHCR image, instead of
`setup-tools.sh`) is actually decided and implemented, it is currently an
open discussion, not available yet.