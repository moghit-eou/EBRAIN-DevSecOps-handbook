# How to Troubleshoot SBOM Generation Errors

SBOM generation is handled by `setup-tools.sh --sbom-ecosystem <maven|npm|none>`,
which runs one of:

```bash
# maven
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q

# npm
npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json
```

## Maven: SBOM is empty or missing expected dependencies

The CycloneDX Maven plugin builds the SBOM from Maven's **resolved**
dependency tree. If `mvn dependency:resolve` hasn't run first (or failed
silently), there's nothing resolved for the plugin to read. Confirm the
resolve step succeeded and populated `~/.m2/repository` before generating
the SBOM, this is why `sca.yml` runs `mvn dependency:resolve -q` as its own
step, before `setup-tools.sh --sbom-ecosystem maven`, see
[reference/pipeline-sca.md](../reference/pipeline-sca.md).

If the resulting SBOM contains a version you didn't expect, that's
expected behavior, not a bug: the SBOM reflects whatever Maven's own
conflict resolution decided to use (nearest-definition, first-come-wins,
BOM order), not necessarily the version declared by a specific transitive
dependency. See
[explanation/why-sboms.md](../explanation/why-sboms.md) for the full
mechanics and a worked example.

## npm: `ELSPROBLEMS` / peer dependency errors during SBOM generation

`cyclonedx-npm` walks the actual installed `node_modules/` tree, which
only exists after `npm ci` (or `npm install`) has run against
`package-lock.json`. Errors here usually mean the install itself is
broken, not the SBOM tool.

Typical symptom:

```
npm error code ELSPROBLEMS
npm error invalid: @angular/common@X.Y.Z /path/to/node_modules/@angular/common
```

This means `package.json` and `package-lock.json` (or the installed tree)
disagree about what should be there. `npm ci` is intentionally strict
about this and fails rather than silently reconciling it, treat the
failure as a signal the lockfile needs to be regenerated:

```bash
npm install
git add package-lock.json
git commit -m "chore: sync package-lock.json"
git push
```

Do not swap `npm ci` for `npm install` inside the pipeline to work around
this. `npm install` can quietly upgrade dependencies to whatever's newest
on the registry at run time, which would make what's scanned in CI
different from what's actually shipped, and would let the lockfile drift
without anyone noticing. See
[explanation/why-sboms.md](../explanation/why-sboms.md) for why the SBOM
needs to reflect exactly what's locked and shipped, not what's merely
declared as a version range in `package.json`.

If you need a temporary lockfile just to unblock local SBOM generation
without a full install, `npm install --package-lock-only --legacy-peer-deps`
regenerates only the lockfile without touching `node_modules`. Don't rely
on this in CI, it's a local debugging step, not a substitute for `npm ci`.

## Gradle: `cyclonedxBom` task not found

This means the CycloneDX Gradle plugin isn't applied to the project. Either
add `id("org.cyclonedx.bom") version "<pinned-version>"` to
`build.gradle`/`build.gradle.kts`, or, if you can't modify the target
project, use the init-script invocation instead:
`./gradlew --init-script cyclonedx.init.gradle cyclonedxBom`. See
[ecosystem-matrix.md](../reference/ecosystem-matrix.md) for both forms.

## Gradle: SBOM is missing dependencies you expect to see

Run `./gradlew dependencies` (or `--write-locks` if the project uses
Gradle's dependency locking) before `cyclonedxBom`, the same way
`mvn dependency:resolve -q` runs before the Maven SBOM step. An
unresolved dependency graph produces an incomplete SBOM the same way it
would for Maven, see [why-sboms.md](../explanation/why-sboms.md) for the
underlying mechanism, this hasn't been independently re-verified against
Gradle's resolution strategy the way it was for Maven, treat it as the
same class of problem until proven otherwise.

## Python: `cyclonedx-py` reports no components, or the wrong versions

This almost always means `cyclonedx-py` was pointed at the wrong source of
truth for what's actually installed:

- `cyclonedx-py requirements -o target/bom.json` reads `requirements.txt`
  directly, if that file isn't pinned/hashed (no exact versions, no
  lockfile), the SBOM reflects whatever ranges are declared, not what's
  actually resolved and installed, the same false-positive risk described
  in [why-sboms.md](../explanation/why-sboms.md) for Maven and npm.
- `cyclonedx-py environment -o target/bom.json` scans the currently active
  virtual environment. If this comes back empty or stale, confirm the venv
  is actually activated (`. .venv/bin/activate`) and that `pip install`
  completed successfully before running the SBOM step, not before.
- For Poetry projects, use `cyclonedx-py poetry -o target/bom.json` against
  a project with a committed `poetry.lock`, not `requirements`, mixing the
  two resolution paths on the same project can produce disagreeing SBOMs.

## Both ecosystems: SBOM step succeeds but scanners still report old data

If Trivy or OSV-Scanner appear to be scanning a stale dependency set,
confirm you're pointing them at the freshly generated `target/bom.json`
(`SBOM_PATH` in `sca_scan.py`) rather than a cached copy from a previous
run. The npm cache step in `sca.yml` caches `~/.npm` (the tarball
download cache), keyed on the hash of `package-lock.json`, not the SBOM
itself, so a stale SBOM points to a stale generation step, not a stale
cache. This applies to every ecosystem's cache, not just npm's, check the
matching cache key in [ecosystem-matrix.md](../reference/ecosystem-matrix.md)
if this happens on Gradle or Python.

## Raw JavaScript: "there's no SBOM step to troubleshoot"

That's expected, not an error. A project with no `package.json` and no
lockfile has no manifest to generate an SBOM from in the first place, see
[the raw-JS section of ecosystem-matrix.md](../reference/ecosystem-matrix.md#raw-javascript-no-packagejson-no-lockfile)
for the recommended `trivy fs` substitute and its real coverage limits.
There's no fix here, this is a structural gap in what's scannable, not a
misconfiguration.
