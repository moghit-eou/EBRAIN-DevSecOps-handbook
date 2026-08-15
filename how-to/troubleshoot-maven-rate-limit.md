# How to Troubleshoot the Maven Central Rate Limit (HTTP 429)

> **Note on this page:** this file is referenced throughout the handbook
> (README's table of contents, `case-studies/platform-backend.md`,
> `explanation/why-sboms.md`) but its full original content was not present
> in the source material available at the time of this restructure, only
> the surrounding references to it. What follows is reconstructed from
> those references and from Trivy's own documented behavior, not copied
> from a prior version of this page. Treat it as a starting point to verify
> against the actual incident notes, not a guaranteed match for the
> original.

## The symptom

While testing Trivy against Maven artifacts not yet present in the local
`~/.m2` cache, Maven Central began returning HTTP 429 (Too Many Requests).
This happened because Trivy, when it encounters an artifact it doesn't
already have metadata for, can attempt to resolve that metadata live
against the upstream registry. Maven Central interpreted the resulting
request volume from a single CI-runner IP as abuse and started throttling
it. See [case-studies/platform-backend.md](../case-studies/platform-backend.md)
for the incident this page documents the mitigation for.

## Why this is a CI-stability problem, not just a false-positive problem

A rate-limited request doesn't produce a wrong scan result, it fails the
tool run outright. This is a different failure mode from the false-positive
problem described in [why-sboms.md](../explanation/why-sboms.md), and it's
part of why this pipeline scans a pre-generated SBOM instead of letting a
scanner resolve artifacts on demand: a pre-generated SBOM removes the
live-registry-lookup step from the scan itself entirely.

## The mitigation

1. **Run `mvn dependency:resolve -q` as its own explicit CI step**, before
   SBOM generation and before Trivy runs at all, see
   [pipeline-sca.md](../reference/pipeline-sca.md). This front-loads all
   registry calls into one step that already has retry semantics built
   into Maven itself, rather than having Trivy make ad-hoc calls mid-scan.
2. **Use Trivy's `--offline-scan` flag** when scanning something that
   doesn't need live registry lookups, this stops Trivy from attempting any
   network calls during the scan and forces it to work only from what's
   already resolved locally or present in the SBOM.
3. **Cache `~/.m2/repository` between CI runs** (see
   [ecosystem-matrix.md](../reference/ecosystem-matrix.md)'s Maven section),
   so repeat runs against unchanged dependencies don't re-trigger resolution
   traffic at all.

## If you still hit 429s after applying the mitigation

Confirm the resolve step is actually running *before* the SBOM/scan steps
in your workflow YAML, not just present somewhere in the file. A misordered
step (SBOM generation before `mvn dependency:resolve -q` completes) can
reintroduce on-demand resolution even with the mitigation nominally in
place. See [ecosystem-matrix.md](../reference/ecosystem-matrix.md)'s Maven
drop-down for the exact step order.

## Related

- [case-studies/platform-backend.md](../case-studies/platform-backend.md), the incident this page documents the fix for.
- [ecosystem-matrix.md](../reference/ecosystem-matrix.md), the Maven row, including the resolve-before-SBOM ordering requirement.
- [why-sboms.md](../explanation/why-sboms.md), the related but distinct false-positive problem solved by the same SBOM-first approach.
