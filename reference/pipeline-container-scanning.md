# Reference: Container Scanning Pipeline

| Property | Value |
|---|---|
| Workflow file | `container-scan.yml` |
| Orchestrator | `ci/container_scan.py` |
| Triggers | `pull_request`, `workflow_dispatch`, weekly schedule |
| Scans | The built Docker image, and the `Dockerfile` itself |
| SCA tools | Trivy, OSV-Scanner |
| SAST tools | OpenGrep |
| Linting tools| Hadolint |
| Gate model, image scan | CVSS-score gate (see [gate-status-cvss.md](gate-status-cvss.md)) |
| Gate model, Dockerfile scan | Rule-severity gate (see [gate-status-rule-severity.md](gate-status-rule-severity.md)) |
| Ecosystem-specific? | No, this pipeline scans the built image and the Dockerfile; it is identical for every language |

## Execution order

```mermaid
sequenceDiagram
    participant CI as container-scan.yml
    participant Docker as Docker build
    participant Orc as container_scan.py
    participant Sec as GitHub Security tab

    CI->>Docker: docker build -t IMAGE_NAME .
    CI->>Orc: --scan-type sast (Hadolint + OpenGrep on Dockerfile)
    Orc-->>CI: exit code
    CI->>Orc: --scan-type sca (Trivy + OSV-Scanner on built image), if: always()
    Orc-->>CI: exit code
    CI->>Sec: upload-sarif (4 individual categories)
    CI->>Orc: --merge-sarif (combine all 4 into one artifact)
    CI->>Sec: upload merged artifact (retention 30 days)
```

The SCA-type run executes with `if: always()`, so it still runs even if
the Dockerfile SAST step failed. This means a bad Dockerfile lint does not
prevent the image itself from being scanned for CVEs in the same job.

## CLI

`container_scan.py` is a single CLI shared by both scan types:

```
$ python3 ci/container_scan.py --help
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

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `IMAGE_NAME` | `<service>:testing` | Image reference passed to Trivy and OSV-Scanner |
| `TRIVY_IGNOREFILE` | `ci/suppress_trivy.yaml` | Trivy suppression file |
| `OSV_IGNOREFILE` | `ci/suppress_osv_scanner.toml` | OSV-Scanner suppression file |
| `TRIVY_SCA_SARIF_OUTPUT` | `sca-trivy-container.sarif` | Trivy image scan output |
| `OSV_SCA_SARIF_OUTPUT` | `sca-osv-container.sarif` | OSV-Scanner image scan output |
| `SEMGREP_CONFIG_RULESETS` | `semgrep-rules/dockerfile` | OpenGrep ruleset(s) for the Dockerfile |
| `OPENGREP_SAST_SARIF_OUTPUT` | `sast-opengrep-dockerfile.sarif` | OpenGrep Dockerfile scan output |
| `HADOLINT_SAST_SARIF_OUTPUT` | `sast-hadolint-dockerfile.sarif` | Hadolint output |
| `CONTAINER_SCAN_MERGED_SARIF_OUTPUT` | `container-scan-<service>-merged.sarif` | Combined artifact of all four SARIF files |

## Running it locally

```bash
docker build -t app:local .
bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules
python ci/container_scan.py --scan-type sast
python ci/container_scan.py --scan-type sca --image app:local
```

## Workflow steps

Like SAST, `container-scan.yml` has no dependency-cache or dependency-
install step, it operates on a built image and a `Dockerfile`, neither of
which involves a package manager, so the file is identical for every
ecosystem:

| # | Step | Notes |
|---|---|---|
| 1 | `actions/checkout` | Standard checkout |
| 2 | `actions/setup-python` | The orchestrator (`container_scan.py`) is Python |
| 3 | `docker build -t $IMAGE_NAME .` | Builds the image once, reused by both scan types below |
| 4 | `setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules` | Installs all four scanner binaries in one step |
| 5 | `container_scan.py --scan-type sast` | Hadolint + OpenGrep against the `Dockerfile` |
| 6 | `container_scan.py --scan-type sca` (`if: always()`) | Trivy + OSV-Scanner against the built image, runs even if step 5 failed |
| 7 | `upload-sarif` (x4) | One category per tool output, see [SARIF upload categories](#sarif-upload-categories) |
| 8 | `container_scan.py --merge-sarif ...` (`if: always()`) | Combines all four SARIF files into one artifact |
| 9 | `upload-artifact` (`if: always()`) | Publishes the merged artifact, 30-day retention |

## Caching

There's no dependency store to cache here for the same reason there's no
install step: this pipeline scans the built image and the `Dockerfile`,
not a resolved dependency tree. Two optional, purely performance-oriented
caches are still worth knowing about, since a slow `docker build` on every
PR is the actual bottleneck in this pipeline, not the scanning itself:

**Docker layer caching**, using `docker/build-push-action` with GitHub
Actions cache backing, if the plain `docker build -t $IMAGE_NAME .` step
becomes slow on a large image:

```yaml
- name: Build Docker image
  uses: docker/build-push-action@v6
  with:
    context: .
    tags: ${{ env.IMAGE_NAME }}
    load: true            # required so `trivy image`/`osv-scanner` can see it locally
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Trivy's own vulnerability database cache**, so every run doesn't
re-download the full CVE database:

```yaml
- name: Cache Trivy DB
  uses: actions/cache@v6
  with:
    path: ~/.cache/trivy
    key: ${{ runner.os }}-trivy-db-${{ steps.trivy-db-date.outputs.date }}
    restore-keys: ${{ runner.os }}-trivy-db-
```

Neither cache affects gate correctness the way the SCA pipeline's
dependency cache does (see
[pipeline-sca.md](pipeline-sca.md#caching)), a stale or missing cache here
only costs build time, it never changes what Trivy, OSV-Scanner, Hadolint,
or OpenGrep actually find.

## Generic GitHub Actions template

This pipeline needs no ecosystem substitution, the same file works for a
Java, npm, Python, or Go service, or anything else with a `Dockerfile`:

```yaml
name: Container Vulnerability Scanning

on:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'

jobs:
  container-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    env:
      IMAGE_NAME: <service>:testing
      CONTAINER_SCAN_MERGED_SARIF_OUTPUT: container-scan-<service>-merged.sarif
      TRIVY_IGNOREFILE: ci/suppress_trivy.yaml
      OSV_IGNOREFILE: ci/suppress_osv_scanner.toml
      TRIVY_SCA_SARIF_OUTPUT: sca-trivy-container.sarif
      OSV_SCA_SARIF_OUTPUT: sca-osv-container.sarif
      SEMGREP_CONFIG_RULESETS: semgrep-rules/dockerfile
      OPENGREP_SAST_SARIF_OUTPUT: sast-opengrep-dockerfile.sarif
      HADOLINT_SAST_SARIF_OUTPUT: sast-hadolint-dockerfile.sarif

    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v6
        with:
          python-version: '3.14.4'

      - name: Build Docker image
        run: docker build -t ${{ env.IMAGE_NAME }} .

      - name: Setup tools
        run: bash ci/setup-tools.sh --install-tool trivy,osv-scanner,opengrep,hadolint,semgrep-rules

      - name: Run SAST scanning (Dockerfile)
        run: python ci/container_scan.py --scan-type sast

      - name: Run SCA scanning (image)
        if: always()
        run: python ci/container_scan.py --scan-type sca --image ${{ env.IMAGE_NAME }}

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.TRIVY_SCA_SARIF_OUTPUT }}
          category: trivy-container-scanning

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.OSV_SCA_SARIF_OUTPUT }}
          category: osv-scanner-container-scanning

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.OPENGREP_SAST_SARIF_OUTPUT }}
          category: opengrep-sast

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.HADOLINT_SAST_SARIF_OUTPUT }}
          category: hadolint-sast

      - name: Merge all SARIF reports
        if: always()
        run: |
          python ci/container_scan.py \
            --merge-sarif "${{ env.TRIVY_SCA_SARIF_OUTPUT }}" "${{ env.OSV_SCA_SARIF_OUTPUT }}" "${{ env.OPENGREP_SAST_SARIF_OUTPUT }}" "${{ env.HADOLINT_SAST_SARIF_OUTPUT }}" \
            --merge-output "${{ env.CONTAINER_SCAN_MERGED_SARIF_OUTPUT }}"

      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: container-scan-sarif-report
          path: ${{ env.CONTAINER_SCAN_MERGED_SARIF_OUTPUT }}
          retention-days: 30
```

> **Note on Action versions:** pin every `uses:` to a full commit SHA in a
> real workflow rather than the floating `@v7`-style tags shown here for
> readability, see [platform-ui.md](../case-studies/platform-ui.md) and
> [platform-backend.md](../case-studies/platform-backend.md) for real,
> SHA-pinned examples.
>
> **Ecosystem-independent by design.** This is the one pipeline of the
> three that never needs an ecosystem-specific block, whatever language or
> build tool produced the `Dockerfile`, this file doesn't change. If your
> project doesn't containerize (a library with no image, for example), this
> pipeline simply doesn't apply, run SCA and SAST only.

## SARIF upload categories

Each SARIF file uploads under its own category, to avoid GitHub's
Code Scanning upload rejecting duplicate categories from the same job:

| Tool output | Category |
|---|---|
| `TRIVY_SCA_SARIF_OUTPUT` | `trivy-container-scanning` |
| `OSV_SCA_SARIF_OUTPUT` | `osv-scanner-container-scanning` |
| `OPENGREP_SAST_SARIF_OUTPUT` | `opengrep-sast` |
| `HADOLINT_SAST_SARIF_OUTPUT` | `hadolint-sast` |

See also: [why-three-independent-pipelines.md](../explanation/why-three-independent-pipelines.md),
[why-these-tools.md](../explanation/why-these-tools.md).
