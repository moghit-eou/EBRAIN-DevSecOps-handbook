# How to Troubleshoot SBOM Generation Errors

SBOM generation is handled by `setup-tools.sh --sbom-ecosystem <value>`.
Exact commands per ecosystem are in
[integrate-a-new-ecosystem.md](integrate-a-new-ecosystem.md#ecosystem-matrix).
This page covers the failure modes actually seen in practice.

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
mechanics and a worked example. The same class of problem applies to
Gradle (resolve before generating) and Go (`go.sum` must be up to date
before `cyclonedx-gomod` runs).

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

## Raw JavaScript: `cdxgen` finds nothing, or under-reports components

`cdxgen`'s filesystem mode fingerprints known library signatures inside
vendored/bundled files, it has no manifest to read. If a vendored script
has been minified, renamed, or bundled into a single file with other
scripts, `cdxgen` may not recognize it. This is an inherent limitation of
manifest-free SBOM generation, not a misconfiguration, treat any SBOM
produced this way as a best-effort inventory, and prefer moving the
project onto a package manager if reliable SCA coverage matters.

## Python: `cyclonedx-py` output doesn't match what's actually installed

`cyclonedx-py` has a separate subcommand per dependency manager
(`requirements`, `poetry`, `pipenv`, `environment`). Running
`cyclonedx-py requirements requirements.txt` against a Poetry-managed
project (which may not even have a `requirements.txt`) either fails or
produces an incomplete SBOM. Match the subcommand to how the project
actually declares dependencies, see
[integrate-a-new-ecosystem.md](integrate-a-new-ecosystem.md#python).

## Both ecosystems: SBOM step succeeds but scanners still report old data

If Trivy or OSV-Scanner appear to be scanning a stale dependency set,
confirm you're pointing them at the freshly generated `target/bom.json`
(`SBOM_PATH` in `sca_scan.py`) rather than a cached copy from a previous
run. The dependency cache step in `sca.yml` (for example, `~/.npm`, keyed
on the hash of `package-lock.json`) caches the package-manager's own
download cache, not the SBOM itself, so a stale SBOM points to a stale
generation step, not a stale dependency cache.
