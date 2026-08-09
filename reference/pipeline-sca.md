# Reference: SCA (Software Composition Analysis) Pipeline

| Property | Value |
|---|---|
| Workflow file | `sca.yml` |
| Orchestrator | `ci/sca_scan.py` |
| Triggers | `pull_request`, `workflow_dispatch`, weekly schedule |
| Scans | Application dependencies, via a generated SBOM, not the source code and not a container image |
| Tools | Trivy, OSV-Scanner |
| Gate model | CVSS-score gate (see [gate-status-cvss.md](gate-status-cvss.md)) |

## Execution order

```mermaid
flowchart TD
    A["Checkout + Setup Python"] --> B{"Ecosystem?"}
    B -->|Maven| C1["Cache ~/.m2/repository"]
    C1 --> C2["mvn dependency:resolve -q"]
    C2 --> C3["setup-tools.sh --sbom-ecosystem maven\n(mvn cyclonedx-maven-plugin -> target/bom.json)"]
    B -->|npm| D1["Cache ~/.npm"]
    D1 --> D2["npm ci"]
    D2 --> D3["setup-tools.sh --sbom-ecosystem npm\n(cyclonedx-npm -> target/bom.json)"]
    C3 --> E["sca_scan.py"]
    D3 --> E
    E --> F["run_trivy(): trivy sbom target/bom.json"]
    E --> G["run_osv_scanner(): osv-scanner scan source --lockfile target/bom.json"]
    F --> H["parse_sarif.evaluate()\nCVSS-score gate"]
    G --> H
    H --> I["Upload Trivy SARIF\ncategory: trivy-app"]
    H --> J["Upload OSV-Scanner SARIF\ncategory: osv-scanner-app"]
    I --> K["Upload merged SARIF artifact"]
    J --> K
```

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `SBOM_PATH` | `target/bom.json` | Path to the generated SBOM that both tools scan |
| `TRIVY_IGNOREFILE` | `ci/suppress_trivy.yaml` | Trivy suppression file |
| `OSV_IGNOREFILE` | `ci/suppress_osv_scanner.toml` | OSV-Scanner suppression file |
| `TRIVY_SARIF_OUTPUT` | `trivy-<service>.sarif` | Trivy SBOM scan output |
| `OSV_SARIF_OUTPUT` | `osv-scanner-<service>.sarif` | OSV-Scanner SBOM scan output |
| `SCA_MERGED_SARIF_OUTPUT` | `SCA-<service>-merged.sarif` | Combined artifact, not used for the gate decision itself |

## Commands run

```bash
# Trivy scans the SBOM directly
trivy sbom "$SBOM_PATH" --format sarif --ignorefile "$TRIVY_IGNOREFILE" --output "$TRIVY_SARIF_OUTPUT"

# OSV-Scanner scans the SBOM as its "lockfile" input
osv-scanner scan source --lockfile "$SBOM_PATH" --config "$OSV_IGNOREFILE" --format sarif --output-file "$OSV_SARIF_OUTPUT"
```

Note the OSV-Scanner orchestration detail: OSV-Scanner returns exit code
`1` whenever it finds any vulnerability at all, regardless of severity.
`run_osv_scanner()` treats that specific exit code as non-fatal and
remaps it to `0`, so the pipeline continues to the SARIF-based gate
evaluation instead of failing immediately on any finding. The actual
pass/fail decision comes from `parse_sarif.evaluate()` afterward, not from
OSV-Scanner's own exit code.

## Running it locally

**Maven:**

```bash
mvn dependency:resolve -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py
```

**npm:**

```bash
npm ci
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py
```

## Caching

| Ecosystem | Cache path | Cache key |
|---|---|---|
| Maven | `~/.m2/repository` | `${{ runner.os }}-m2-v1-${{ hashFiles('**/pom.xml') }}` |
| npm | `~/.npm` | `${{ runner.os }}-npm-v1-${{ hashFiles('**/package-lock.json') }}` |

The npm cache key must be derived from `package-lock.json`, not
`package.json`. Hashing the wrong file could give two different resolved
dependency trees the same cache key, a correctness bug, not just a
performance one, since the cached tarballs would then be reused across
builds that should have installed different versions.

See also: [why-sboms.md](../explanation/why-sboms.md),
[why-two-sca-tools.md](../explanation/why-two-sca-tools.md).