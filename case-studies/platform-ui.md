# Case Study: `platform-ui`

`platform-ui` (Medical Informatics Platform org on GitHub) is the npm/
Angular reference implementation this handbook's SCA pipeline is validated
against, alongside the Maven-based `platform-backend`. This case study was
split out from `platform-backend.md` (previously both were combined in one
file, tracked as an open item, now resolved) and covers the npm-side
finding that shaped the case for running multiple SCA tools: a real
malware-detection false positive caused by a PURL collision.

## The finding: a malware flag on a legitimate dependency

Dependency-Track flagged a malware finding (`MAL-2022-4051`) against
`platform-ui`'s legitimate dependency `jQuery-QueryBuilder`.

## The mechanism: lowercase PURL collision

`cyclonedx-npm` lowercases all package names when generating the SBOM,
producing the PURL `pkg:npm/jquery-querybuilder@3.0.0` for the legitimate
package. A separate, real, unrelated malicious package is registered on npm
under that same lowercase name, `jquery-querybuilder`. Because npm's own
registry is case-insensitive, both the legitimate `jQuery-QueryBuilder` and
the malicious `jquery-querybuilder` collapse into the same PURL once
lowercased. Dependency-Track, which relies on the PURL as its lookup key
against OSV, couldn't distinguish them.

## Why this matters beyond the one false positive

This same malware flag was **not** caught by `npm audit`, only by
Dependency-Track via OSV. That's concrete, first-hand evidence for why
relying on a single tool and a single vulnerability database leaves gaps,
directly informing the decision to run both Trivy and OSV-Scanner rather
than either alone, see
[explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md).

It's also a caution specific to any ecosystem where `cyclonedx-npm`'s
lowercasing behavior applies: a case-sensitive package name with a
case-insensitive npm registry is a structural collision risk, not a one-off
bug in this specific package. Anyone extending the pipeline to a new
JavaScript-adjacent tool that also generates or consumes lowercased PURLs
should expect the same class of false positive to be possible again.

## What this shaped in the final pipeline

- Both Trivy and OSV-Scanner run against every generated SBOM, not just one
  tool chosen as "the" SCA scanner, see
  [pipeline-sca.md](../reference/pipeline-sca.md).
- Suppression entries for confirmed false positives (like this one, once
  triaged) go through `ci/suppress_osv_scanner.toml` /
  `ci/suppress_trivy.yaml` with a `statement` and `expires` date, see
  [how-to/suppress-a-finding.md](../how-to/suppress-a-finding.md), rather
  than being silently ignored.
- The npm row of
  [reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md) calls
  out this exact collision risk as a caveat for anyone integrating a new
  npm-based repository.

## Related

- [case-studies/platform-backend.md](platform-backend.md), the Maven-side companion investigation (the SBOM-vs-cache false-positive story).
- [explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md).
- [explanation/why-sboms.md](../explanation/why-sboms.md).
