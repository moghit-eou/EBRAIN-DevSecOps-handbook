# Tutorial: Integrate the Security Pipelines Into Any Repository

This tutorial walks through wiring all three security pipelines (Container
Scanning, SCA, SAST) into a new repository, start to finish, regardless of
language or build tool. It replaces separate Maven-specific and npm-specific
setup guides: everything below is identical across ecosystems except the
single SCA step marked **"look up your ecosystem"**, which points at
[reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md).

**Platform requirement:** verified on Linux x86_64 (Ubuntu), matching the
`ubuntu-latest` GitHub Actions runner. On Windows, use WSL2. There is
currently no working path on native macOS.
*Note: full Dockerization of the toolchain, to guarantee frictionless
execution across all operating systems, is an idea under discussion with
the project's docs mentor. It is not a decided direction, see
[tool-installation-flags.md](../reference/tool-installation-flags.md).*

## What you'll need

* A Linux x86_64 environment (native Ubuntu or WSL2).
* `bash`, `curl`, `sudo`, `git`.
* Docker (for the Container Scanning pipeline).
* Whatever your project's ecosystem needs to resolve dependencies (Maven,
  Gradle, npm, or Python's pip/Poetry). See
  [reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md) if
  you're unsure which commands apply.
* The `ci/` scripts from the reference implementation:
  `setup-tools.sh`, `container_scan.py`, `sca_scan.py`, `sast_scan.py`,
  `parse_sarif.py`.

## Step 1: check out the repository

```bash
git clone <your-repo-url>
cd <your-service>
```

## Step 2: run the SAST pipeline

SAST scans your source code only. It doesn't need Docker or your dependency
manager, and this step is **identical regardless of ecosystem**, only the
OpenGrep ruleset list changes per language:

```bash
bash ci/setup-tools.sh --install-tool opengrep,semgrep-rules
python ci/sast_scan.py
```

This installs OpenGrep and clones the pinned `semgrep-rules` community
ruleset, then runs OpenGrep against your source tree twice, once to write a
full SARIF report, once as the actual pass/fail gate. See
[pipeline-sast.md](../reference/pipeline-sast.md) for what each run does,
and set `SEMGREP_CONFIG_RULESETS` to match your language, see the ruleset
table there for the Java and JavaScript examples already in production; the
same pattern extends to other languages by picking the matching
`semgrep-rules/<lang>` pack.

Expect a `sast-opengrep-*.sarif` file (or your project's configured output
name) in the working directory when this finishes.

## Step 3: run the SCA pipeline — look up your ecosystem

SCA scans your application's declared dependencies, via a generated SBOM,
not your source code and not a container image. The orchestrator
(`sca_scan.py`), the two scanning tools (Trivy, OSV-Scanner), and the gate
are **identical for every ecosystem**. The only thing that changes is how
the SBOM gets generated and what gets cached, both are one lookup away:

Open [reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md), find
your ecosystem's drop-down, and copy its **resolve** and **generate SBOM**
commands into the three-line pattern below:

```bash
<resolve-dependencies-command>          # from the matrix
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem <value>   # <value> from the matrix
python ci/sca_scan.py
```

For example, Maven and npm (both shipped today) look like this:

```bash
# Maven
mvn dependency:resolve -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py

# npm
npm ci
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py
```

Gradle and Python follow the same three-line shape, but `--sbom-ecosystem`
doesn't have a `gradle` or `python` case yet, run their SBOM command
directly instead of through `setup-tools.sh` until that gap closes, see
[the matrix's gap note](../reference/ecosystem-matrix.md#closing-the-gap-wiring-a-new-ecosystem-into-setup-toolssh).

Raw JavaScript with no package manager has no SBOM path at all, use the
`trivy fs` substitute documented in
[the matrix's raw-JS section](../reference/ecosystem-matrix.md#raw-javascript-no-packagejson-no-lockfile)
instead of this step.

**Whichever ecosystem you're on:** never substitute a "looser" install
command (`npm install` instead of `npm ci`, an unpinned `pip install`
without a lockfile) to work around a resolution failure inside the
pipeline. Fix the lockfile/manifest and commit it. See
[why-sboms.md](../explanation/why-sboms.md) for why this matters, an SBOM
built from an unresolved or loosely-resolved dependency tree reproduces the
exact 300+-false-positive failure mode that's the entire reason this
pipeline scans an SBOM instead of a raw dependency cache in the first
place.

## Step 4: run the Container Scanning pipeline

Container Scanning covers two things in one pipeline: the Dockerfile itself
(SAST-style linting) and the built image (SCA-style CVE scanning). This
step is **fully ecosystem-agnostic already**, nothing here changes based on
what language the application is written in:

```bash
docker build -t <your-image>:local .
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
python ci/container_scan.py --scan-type sast
python ci/container_scan.py --scan-type sca --image <your-image>:local
```

The `--scan-type sast` run scans the `Dockerfile` with Hadolint and
OpenGrep. The `--scan-type sca` run scans the built image with Trivy and
OSV-Scanner. Both use the same `container_scan.py` CLI, just with a
different `--scan-type`. Full flag reference:

```bash
python ci/container_scan.py --help
```

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

## Step 5: read the results

Each tool writes its own SARIF file. All three orchestrator scripts print a
summary block at the end of their run, for example:

```text
========== SCA PIPELINE SUMMARY ==========
[trivy]: PASSED
[osv-scanner]: WARNING (findings between 5.0 and 8.0)
============================================
```

If you don't have the GitHub Security tab available (a local run, or a
SARIF file downloaded from workflow artifacts), open it in a SARIF viewer
such as [Microsoft's SARIF Web Component](https://microsoft.github.io/sarif-web-component/)
instead of reading the raw JSON.

To understand what PASSED, WARNING, FAILED, and ERROR mean for each tool,
see [gate-status-cvss.md](../reference/gate-status-cvss.md) and
[gate-status-rule-severity.md](../reference/gate-status-rule-severity.md).

## Wiring this into GitHub Actions

Everything above works as a local, ecosystem-agnostic template for the
workflow YAML too. The three shipped workflow files
(`container-scan.yml`, `sca.yml`, `sast.yml`) already follow this exact
step order; only the ecosystem-specific cache step and the resolve step
inside `sca.yml` change per project, both taken directly from
[reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md). Don't
duplicate the `sast.yml` or `container-scan.yml` files per ecosystem, they
don't need to change at all.

## Next steps

* If a finding is a false positive, see
  [suppress-a-finding.md](../how-to/suppress-a-finding.md).
* If you hit a Maven registry rate limit while resolving dependencies, see
  [troubleshoot-maven-rate-limit.md](../how-to/troubleshoot-maven-rate-limit.md).
* If SBOM generation itself fails, see
  [troubleshoot-sbom-generation-errors.md](../how-to/troubleshoot-sbom-generation-errors.md).
* If your ecosystem isn't in the matrix yet, see
  [the matrix's gap note](../reference/ecosystem-matrix.md#closing-the-gap-wiring-a-new-ecosystem-into-setup-toolssh)
  and [add-new-scanner.md](../how-to/add-new-scanner.md) for the general
  pattern of extending `setup-tools.sh`.
