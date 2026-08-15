# How to Integrate the Security Pipeline Into a New Repository

This is the single, ecosystem-agnostic guide for wiring Container Scanning,
SCA, and SAST into any repository, regardless of build tool or language.
Maven and npm are two supported ecosystems, not the only two. Everything
here is written so it applies identically to a Gradle service, a raw
(package-manager-free) JavaScript app, or a Python project.

If you're setting up a **Maven or npm** repo and just want the exact
commands, jump straight to the [ecosystem matrix](#ecosystem-matrix) below.
If you're adding support for an ecosystem that isn't in the matrix yet,
read [Adding a new ecosystem](#adding-a-new-ecosystem-not-in-the-matrix)
first.

## What actually changes per ecosystem

Only two things in the whole pipeline are ecosystem-specific. Everything
else (`sca_scan.py`, `sast_scan.py`, `container_scan.py`, `parse_sarif.py`,
the gate models, the suppression files) is already language-agnostic and
requires **no changes**:

| What | Ecosystem-specific? | Where it lives |
|---|---|---|
| Resolving/installing dependencies before SBOM generation | Yes | One step in `sca.yml` (`mvn dependency:resolve`, `npm ci`, etc.) |
| Generating the CycloneDX SBOM | Yes | The `--sbom-ecosystem` branch in `setup-tools.sh` |
| Caching the dependency store between runs | Yes | The `actions/cache` step in `sca.yml` |
| Scanning the SBOM (Trivy, OSV-Scanner) | **No** | `sca_scan.py` reads `SBOM_PATH`; it has no idea what produced the file |
| SAST of source code (OpenGrep) | Partially | Same tool everywhere; only the ruleset/exclude list differs, see [pipeline-sast.md](../reference/pipeline-sast.md) |
| Container Scanning | **No** | Scans the built image and the `Dockerfile`; independent of language |

In other words, integrating a new ecosystem means changing **one cache
step, one install/resolve command, and one SBOM-generation command**. Once
`target/bom.json` (or wherever `SBOM_PATH` points) exists in CycloneDX
format, the rest of the pipeline is identical for every language.

## Ecosystem matrix

Each block below is copy-paste-ready: the dependency step, the SBOM
command, the cache configuration, and the `setup-tools.sh` case branch for
that ecosystem. Expand the one you need.

<details>
<summary><strong>Maven</strong></summary>

**Dependency resolve step** (`sca.yml`, before `setup-tools.sh`):
```yaml
- name: Cache Maven packages
  uses: actions/cache@v6
  with:
    path: ~/.m2/repository   # /.m2 alone is too broad; risks cache corruption
    key: ${{ runner.os }}-m2-v1-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-m2-v1-

- name: Resolve Maven dependencies
  run: mvn dependency:resolve -q
```

**SBOM tool:** `cyclonedx-maven-plugin` (no separate install step, invoked
via `mvn`).

**SBOM command** (`setup-tools.sh` branch):
```bash
maven)
  echo "Generating SBOM for Maven project"
  mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q
  ;;
```

**Default SBOM output:** `target/bom.json`

**Local run:**
```bash
mvn dependency:resolve -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem maven
python ci/sca_scan.py
```

`mvn dependency:resolve` must run first: the CycloneDX plugin builds the
SBOM from Maven's **resolved** dependency tree, not from `pom.xml`
declarations directly. See [why-sboms.md](../explanation/why-sboms.md) for
why this distinction matters.

</details>

<details>
<summary><strong>Gradle</strong></summary>

**Dependency resolve step:**
```yaml
- name: Cache Gradle packages
  uses: actions/cache@v6
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-v1-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: ${{ runner.os }}-gradle-v1-

- name: Resolve Gradle dependencies
  run: ./gradlew dependencies --write-locks -q
```

**SBOM tool:** [`cyclonedx-gradle-plugin`](https://github.com/CycloneDX/cyclonedx-gradle-plugin).
Add it to `build.gradle` (or `build.gradle.kts`):
```groovy
plugins {
    id 'org.cyclonedx.bom' version '2.3.1'
}
cyclonedxBom {
    outputName = "bom"
    outputFormat = "json"
    destination = file("build/reports")
}
```

**SBOM command** (`setup-tools.sh` branch):
```bash
gradle)
  echo "Generating SBOM for Gradle project"
  ./gradlew cyclonedxBom -q
  cp build/reports/bom.json target/bom.json   # normalize to the same SBOM_PATH as other ecosystems
  ;;
```

**Default SBOM output:** `build/reports/bom.json` (copy or symlink it to
`target/bom.json` so `SBOM_PATH` doesn't need to vary per ecosystem in
`sca_scan.py`).

**Local run:**
```bash
./gradlew dependencies --write-locks -q
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem gradle
python ci/sca_scan.py
```

Like Maven, the SBOM must be generated from Gradle's own dependency
resolution (`dependencies`/`cyclonedxBom`), not by reading `build.gradle`
declarations directly, for the same reasons documented in
[why-sboms.md](../explanation/why-sboms.md): a build tool's conflict
resolution can silently override a declared version, and only the
resolved graph reflects what's actually shipped.

</details>

<details>
<summary><strong>NPM</strong></summary>

**Dependency install step:**
```yaml
- name: Cache npm packages
  uses: actions/cache@v6
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-v1-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-npm-v1-

- name: Install dependencies
  run: npm ci
```

**SBOM tool:** `@cyclonedx/cyclonedx-npm` (via `npx`, no separate install
step).

**SBOM command** (`setup-tools.sh` branch):
```bash
npm)
  echo "Generating SBOM for NPM project"
  npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json
  ;;
```

**Default SBOM output:** `target/bom.json`

**Local run:**
```bash
npm ci
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem npm
python ci/sca_scan.py
```

Use `npm ci`, never `npm install`, ahead of SBOM generation: `npm ci`
installs strictly from `package-lock.json`, wipes `node_modules` first,
fails immediately on a lockfile/manifest mismatch, and never rewrites the
lockfile. `cyclonedx-npm` walks the installed `node_modules/` tree, so an
install done with `npm install` can make the SBOM reflect a different
dependency set than what's actually locked. See
[troubleshoot-sbom-generation-errors.md](troubleshoot-sbom-generation-errors.md#npm-elsproblems--peer-dependency-errors-during-sbom-generation).

</details>

<details>
<summary><strong>Raw JavaScript (no package manager, no lockfile)</strong></summary>

Applies to static/vendored front-end JS with no `package.json`, or a
project that ships third-party scripts directly rather than through a
package manager. Neither `npm ci` nor `cyclonedx-npm` has anything to
walk in this case, there's no manifest and no `node_modules/`.

**Dependency step:** none, there's nothing to install.

**SBOM tool:** [`cdxgen`](https://github.com/CycloneDX/cdxgen), a
filesystem-based CycloneDX generator that fingerprints vendored/bundled
JS files directly (by parsing script contents and known library
signatures) instead of reading a manifest.

**SBOM command** (`setup-tools.sh` branch):
```bash
raw-js)
  echo "Generating SBOM for raw JavaScript (no package manager) project"
  npx --yes @cyclonedx/cdxgen -t js -o target/bom.json .
  ;;
```

**Default SBOM output:** `target/bom.json`

**Local run:**
```bash
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem raw-js
python ci/sca_scan.py
```

**Caching:** none applicable, there's no dependency store to cache; `cdxgen`
reads only the files already checked out.

Coverage from `cdxgen`'s filesystem fingerprinting is inherently weaker
than a lockfile-based SBOM, it can only detect what it recognizes by
signature. Treat this path as a fallback for genuinely unmanaged JS, not a
substitute for adopting npm if that's an option.

</details>

<details>
<summary><strong>Python</strong></summary>

**Dependency install step:**
```yaml
- name: Cache pip packages
  uses: actions/cache@v6
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-v1-${{ hashFiles('**/requirements*.txt', '**/poetry.lock', '**/Pipfile.lock') }}
    restore-keys: ${{ runner.os }}-pip-v1-

- name: Install dependencies
  run: pip install -r requirements.txt   # or: poetry install --no-root
```

**SBOM tool:** [`cyclonedx-py`](https://github.com/CycloneDX/cyclonedx-python)
(`pip install cyclonedx-bom`).

**SBOM command** (`setup-tools.sh` branch):
```bash
python)
  echo "Generating SBOM for Python project"
  pip install --quiet "cyclonedx-bom==${CYCLONEDX_PY_VERSION}"
  cyclonedx-py requirements requirements.txt -o target/bom.json
  # Poetry projects: cyclonedx-py poetry -o target/bom.json
  # Pipenv projects: cyclonedx-py pipenv   -o target/bom.json
  ;;
```

**Default SBOM output:** `target/bom.json`

**Local run:**
```bash
pip install -r requirements.txt
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem python
python ci/sca_scan.py
```

`cyclonedx-py` has a dedicated subcommand per dependency manager
(`requirements`, `poetry`, `pipenv`, `environment`). Pick the one matching
how the repository declares dependencies, mixing `requirements.txt` scanning
with a Poetry-managed repo produces an incomplete SBOM, same
resolved-vs-declared failure mode as reading raw Maven POMs, see
[why-sboms.md](../explanation/why-sboms.md).

</details>

<details>
<summary><strong>Go</strong></summary>

**Dependency step:**
```yaml
- name: Cache Go modules
  uses: actions/cache@v6
  with:
    path: |
      ~/go/pkg/mod
      ~/.cache/go-build
    key: ${{ runner.os }}-go-v1-${{ hashFiles('**/go.sum') }}
    restore-keys: ${{ runner.os }}-go-v1-

- name: Download Go modules
  run: go mod download
```

**SBOM tool:** [`cyclonedx-gomod`](https://github.com/CycloneDX/cyclonedx-gomod).

**SBOM command** (`setup-tools.sh` branch):
```bash
go)
  echo "Generating SBOM for Go module"
  cyclonedx-gomod mod -json -output target/bom.json
  ;;
```

**Default SBOM output:** `target/bom.json`

**Local run:**
```bash
go mod download
bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem go
python ci/sca_scan.py
```

`cyclonedx-gomod` reads `go.sum`, the resolved and checksummed module set,
not `go.mod`'s version constraints directly, so it has the same
resolved-vs-declared correctness property as the Maven and npm paths.

</details>

## Extending `setup-tools.sh`

All six branches above plug into the same `case` statement already in
`setup-tools.sh`:

```bash
case "$SBOM_ECOSYSTEM" in
  maven)   mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q ;;
  gradle)  ./gradlew cyclonedxBom -q && cp build/reports/bom.json target/bom.json ;;
  npm)     npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json ;;
  raw-js)  npx --yes @cyclonedx/cdxgen -t js -o target/bom.json . ;;
  python)  cyclonedx-py requirements requirements.txt -o target/bom.json ;;
  go)      cyclonedx-gomod mod -json -output target/bom.json ;;
  none)    echo "No SBOM generation needed" ;;
  *)       echo "Unknown SBOM_ECOSYSTEM: $SBOM_ECOSYSTEM" >&2; exit 1 ;;
esac
```

Adding an ecosystem not listed here means adding one more branch that ends
with a CycloneDX file at `SBOM_PATH`. Nothing downstream (`sca_scan.py`,
the gate model, suppression) needs to know or care which branch produced
it.

## The generic `sca.yml` template

```yaml
name: Software Composition Analysis (SCA)

on:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'

jobs:
  sca:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    env:
      SBOM_PATH: target/bom.json
      TRIVY_IGNOREFILE: ci/suppress_trivy.yaml
      OSV_IGNOREFILE: ci/suppress_osv_scanner.toml
      TRIVY_SARIF_OUTPUT: trivy-<service>.sarif
      OSV_SARIF_OUTPUT: osv-scanner-<service>.sarif
      SCA_MERGED_SARIF_OUTPUT: SCA-<service>-merged.sarif

    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v6
        with:
          python-version: '3.14.4'

      # --- replace this block with the matching ecosystem block above ---
      - name: Cache dependencies
        uses: actions/cache@v6
        with:
          path: <ecosystem cache path>
          key: ${{ runner.os }}-<ecosystem>-v1-${{ hashFiles('<lockfile glob>') }}
          restore-keys: ${{ runner.os }}-<ecosystem>-v1-

      - name: Install / resolve dependencies
        run: <ecosystem install command>
      # --------------------------------------------------------------------

      - name: Setup tools
        run: bash ci/setup-tools.sh --install-tool trivy,osv-scanner --sbom-ecosystem <ecosystem>

      - name: Run SCA tools
        run: python ci/sca_scan.py

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.TRIVY_SARIF_OUTPUT }}
          category: trivy-app

      - uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: ${{ env.OSV_SARIF_OUTPUT }}
          category: osv-scanner-app

      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: sca-scan-sarif-report
          path: ${{ env.SCA_MERGED_SARIF_OUTPUT }}
          retention-days: 30
```

`sast.yml` and `container-scan.yml` need no ecosystem-specific block at
all beyond the `SEMGREP_CONFIG_RULESETS` / `OPENGREP_EXCLUDE` values, see
[pipeline-sast.md](../reference/pipeline-sast.md#ruleset-configuration-by-repository).

## Adding a new ecosystem not in the matrix

1. Confirm a CycloneDX generator exists for it. Check the
   [CycloneDX tool center](https://cyclonedx.org/tool-center/) before
   writing custom SBOM generation, most ecosystems already have one.
2. Confirm it reads the ecosystem's **resolved** dependency state (a
   lockfile, a resolved module graph), not just manifest declarations. A
   generator that reads declarations only reproduces the false-positive
   failure mode documented in [why-sboms.md](../explanation/why-sboms.md).
3. Add a cache step keyed on that ecosystem's lockfile hash.
4. Add one `case` branch to `setup-tools.sh` that ends with a CycloneDX
   file at `SBOM_PATH`.
5. Nothing else changes. `sca_scan.py`, the gate model, and suppression
   are already ecosystem-agnostic.

This is the same three-file pattern used for
[adding a new scanner](add-new-scanner.md), applied to ecosystems instead
of tools.
