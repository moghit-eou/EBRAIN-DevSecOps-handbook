# Reference: Ecosystem Matrix (Build, SBOM, Cache)

This page is the single source of truth for adapting the SCA pipeline to a
new language ecosystem. It replaces having separate `pipeline-sca-maven.md` /
`pipeline-sca-npm.md`-style pages, one row here is all that changes between
ecosystems, everything else in [pipeline-sca.md](pipeline-sca.md) (the
orchestrator, the gate, the tools) is identical regardless of what's in this
table.

**How to use this page:** find your ecosystem, copy the four commands, drop
them into the four slots described in
[tutorials/01-integrate-your-repository.md](../tutorials/01-integrate-your-repository.md).
Nothing else in the pipeline needs to change.

## Status

| Ecosystem | SBOM support | Notes |
|---|---|---|
| Maven | **Shipped.** `setup-tools.sh --sbom-ecosystem maven` | Reference implementation: `platform-backend` |
| npm | **Shipped.** `setup-tools.sh --sbom-ecosystem npm` | Reference implementation: `platform-ui` |
| Gradle | **Proposed, not yet wired into `setup-tools.sh`.** Commands below are validated manually, not yet behind a `--sbom-ecosystem gradle` flag. | See [gap below](#closing-the-gap-wiring-a-new-ecosystem-into-setup-toolssh) |
| Python | **Proposed, not yet wired into `setup-tools.sh`.** Same caveat as Gradle. | pip/requirements.txt and Poetry variants both covered |
| Raw JavaScript (no package manager) | **No SBOM path.** See [the raw-JS caveat](#raw-javascript-no-packagejson-no-lockfile) below, use direct filesystem scanning instead. | Not a gap to close the same way, there is no manifest to build an SBOM from |

<details>
<summary><strong>Maven</strong></summary>

| Step | Command |
|---|---|
| Resolve dependencies | `mvn dependency:resolve -q` |
| Generate SBOM | `mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q` |
| SBOM output path | `target/bom.json` |
| Cache path | `~/.m2/repository` (not `~/.m2`, see caveat below) |
| Cache key | `${{ runner.os }}-m2-v1-${{ hashFiles('**/pom.xml') }}` |
| `--sbom-ecosystem` value | `maven` |

**Caveats:**
- Cache exactly `~/.m2/repository`, not the whole `~/.m2` directory. Caching
  `~/.m2` risks cache corruption from Maven's own lock/settings files; see
  [case-studies/platform-backend.md](../case-studies/platform-backend.md).
- Run `mvn dependency:resolve -q` as its **own step**, before SBOM
  generation, not inside it. Otherwise an unresolved artifact can trigger
  Maven Central rate limiting (HTTP 429) mid-SBOM-generation instead of
  during a clearly-labeled resolve step, see
  [troubleshoot-maven-rate-limit.md](../how-to/troubleshoot-maven-rate-limit.md).
- The SBOM reflects Maven's resolved versions (nearest-definition,
  first-come-wins, BOM order), not whatever a transitive dependency's own
  POM declares. See [why-sboms.md](../explanation/why-sboms.md).

</details>

<details>
<summary><strong>Gradle</strong></summary>

| Step | Command |
|---|---|
| Resolve dependencies | `./gradlew dependencies --write-locks` (or just `./gradlew build -x test` if lockfiles aren't in use) |
| Generate SBOM | `./gradlew cyclonedxBom` |
| SBOM output path | `build/reports/bom.json` (default for the CycloneDX Gradle plugin) |
| Cache path | `~/.gradle/caches` and `~/.gradle/wrapper` |
| Cache key | `${{ runner.os }}-gradle-v1-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}` |
| `--sbom-ecosystem` value | `gradle` (**proposed**, not yet implemented, see gap below) |

**Requires:** the [CycloneDX Gradle plugin](https://github.com/CycloneDX/cyclonedx-gradle-plugin)
applied in `build.gradle` / `build.gradle.kts`:

```kotlin
plugins {
    id("org.cyclonedx.bom") version "<pinned-version>"
}
```

If the target repository can't be modified to add the plugin (for example,
scanning a third-party Gradle project), the CycloneDX Gradle plugin also
supports an [init script](https://github.com/CycloneDX/cyclonedx-gradle-plugin#using-an-init-script)
applied at invocation time instead:

```bash
./gradlew --init-script cyclonedx.init.gradle cyclonedxBom
```

**Caveats:**
- Same version-conflict-resolution mechanism as Maven applies conceptually,
  Gradle resolves a single winning version per dependency, and the SBOM
  should reflect that resolution, not raw declared ranges. This has not
  been independently verified against Gradle's own resolution strategy
  (which differs in detail from Maven's) the way it was for
  `platform-backend`, see [why-sboms.md](../explanation/why-sboms.md) for
  the Maven-specific evidence this is extrapolated from.
- Cache both `~/.gradle/caches` (downloaded dependencies) and
  `~/.gradle/wrapper` (the Gradle distribution itself, if using the
  wrapper), otherwise every run re-downloads Gradle itself, not just
  dependencies.

</details>

<details>
<summary><strong>npm</strong></summary>

| Step | Command |
|---|---|
| Resolve dependencies | `npm ci` (not `npm install`, see caveat) |
| Generate SBOM | `npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json` |
| SBOM output path | `target/bom.json` |
| Cache path | `~/.npm` |
| Cache key | `${{ runner.os }}-npm-v1-${{ hashFiles('**/package-lock.json') }}` |
| `--sbom-ecosystem` value | `npm` |

**Caveats:**
- `npm ci` only, never `npm install` in the pipeline. `npm ci` installs
  strictly from `package-lock.json`, wipes `node_modules` first, and fails
  loudly if `package.json`/`package-lock.json` disagree, instead of
  silently rewriting the lockfile and scanning something that doesn't match
  what's actually shipped.
- The cache key must hash `package-lock.json`, not `package.json`. Hashing
  the wrong file can give two different resolved dependency trees the same
  cache key, a correctness bug, not just a performance one.
- `cyclonedx-npm` **lowercases every package name** when it builds PURLs for
  the SBOM. This has produced a real false-positive/collision incident, see
  [case-studies/platform-ui.md](../case-studies/platform-ui.md) (the
  `jQuery-QueryBuilder` malware-flag collision).

</details>

<details>
<summary><strong>Python</strong></summary>

| Step | Command |
|---|---|
| Resolve dependencies (requirements.txt) | `python -m venv .venv && . .venv/bin/activate && pip install -r requirements.txt` |
| Resolve dependencies (Poetry) | `poetry install --no-interaction --no-root` |
| Generate SBOM (requirements.txt) | `cyclonedx-py requirements -o target/bom.json` |
| Generate SBOM (Poetry) | `cyclonedx-py poetry -o target/bom.json` |
| Generate SBOM (installed environment, either case) | `cyclonedx-py environment -o target/bom.json` |
| SBOM output path | `target/bom.json` |
| Cache path (pip) | `~/.cache/pip` |
| Cache key (pip) | `${{ runner.os }}-pip-v1-${{ hashFiles('**/requirements*.txt') }}` |
| Cache path (Poetry) | `~/.cache/pypoetry` |
| Cache key (Poetry) | `${{ runner.os }}-poetry-v1-${{ hashFiles('**/poetry.lock') }}` |
| `--sbom-ecosystem` value | `python` (**proposed**, not yet implemented, see gap below) |

**Requires:** [`cyclonedx-py`](https://github.com/CycloneDX/cyclonedx-python)
installed in the runner (`pip install cyclonedx-bom`), separate from
whatever the target project itself depends on.

**Caveats:**
- `requirements.txt` alone doesn't pin transitive dependencies the way a
  lockfile does. If the project has no lockfile (`poetry.lock`,
  `Pipfile.lock`, or pinned/hashed `requirements.txt` via
  `pip-compile`/`pip freeze`), the resulting SBOM can drift between runs the
  same way an unlocked npm install would. The `environment` mode
  (scan the actually-installed venv) is the closest Python equivalent to
  scanning npm's `node_modules/` post-`npm ci`, and is the recommended
  default for that reason.
- Pick one resolution path (`requirements.txt` or `poetry`) per project;
  don't run both against the same repository, they can disagree on
  resolved versions if the two manifests aren't kept in sync.

</details>

<details>
<summary><strong>Raw JavaScript (no <code>package.json</code>, no lockfile)</strong></summary>

There is **no SBOM path** for this case, an SBOM is generated from a
resolved dependency manifest, and there is no manifest to resolve. This
applies to vendored or CDN-included `.js` files with no package manager
involved at all.

| Step | Command |
|---|---|
| Resolve dependencies | N/A, nothing to install |
| Generate SBOM | N/A, not possible without a manifest |
| Recommended substitute | `trivy fs <source-dir> --scanners vuln --format sarif --output trivy-fs.sarif` run directly against the source tree |
| Cache | N/A |
| `--sbom-ecosystem` value | `none` |

**What this actually catches:** Trivy's filesystem scanner can identify some
known-vulnerable JS libraries by file signature even without a manifest, but
coverage is meaningfully weaker than a manifest-driven scan, it depends on
Trivy recognizing the specific vendored file. For projects that vendor
identifiable libraries (a full, unminified `jquery-3.x.x.js`, for example),
this catches known CVEs in that exact file. For heavily bundled or minified
code with no recognizable file signature, this approach will not find
anything meaningful, treat that as a gap to flag to the project, not a
silent pass. Migrating the project to a package manager (even just for
dependency tracking, without changing how the code ships) is the durable
fix if compliance requirements need real SBOM coverage.

</details>

## Closing the gap: wiring a new ecosystem into `setup-tools.sh`

Gradle and Python are **not yet** wired behind `--sbom-ecosystem` in the
shipped `setup-tools.sh`, only `maven`, `npm`, and `none` are implemented
today:

```bash
# Current, shipped:
case "$SBOM_ECOSYSTEM" in
  maven) mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q ;;
  npm)   npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json ;;
  none)  echo "No SBOM generation needed" ;;
  *)     echo "Unknown SBOM_ECOSYSTEM: $SBOM_ECOSYSTEM" >&2; exit 1 ;;
esac
```

Adding a new case follows the same shape as
[how-to/add-new-scanner.md](../how-to/add-new-scanner.md), pin nothing new
here except the SBOM generator's own version if it isn't already present on
the runner:

```bash
# Proposed addition, not yet merged:
case "$SBOM_ECOSYSTEM" in
  maven)  mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q ;;
  npm)    npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json ;;
  gradle) ./gradlew cyclonedxBom ;;
  python) cyclonedx-py environment -o target/bom.json ;;
  none)   echo "No SBOM generation needed" ;;
  *)      echo "Unknown SBOM_ECOSYSTEM: $SBOM_ECOSYSTEM" >&2; exit 1 ;;
esac
```

Until this lands, treat Gradle and Python as **documented but manual**:
run the commands from this page directly in your workflow YAML rather than
relying on `--sbom-ecosystem` to do it for you.

## Related

- [tutorials/01-integrate-your-repository.md](../tutorials/01-integrate-your-repository.md), the step-by-step walkthrough that uses this matrix.
- [pipeline-sca.md](pipeline-sca.md), what happens to the SBOM once it's generated (identical regardless of ecosystem).
- [why-sboms.md](../explanation/why-sboms.md), why scanning a resolved SBOM instead of a raw dependency cache matters, and why this matters even more for ecosystems not yet battle-tested the way Maven and npm are.
- [tool-installation-flags.md](tool-installation-flags.md), the full `setup-tools.sh` flag reference.
