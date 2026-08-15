# Case Study: `platform-backend`

Todo : Things feel redundant , case-studies should be for platform-ui and platform-backend 


`platform-backend` (Medical Informatics Platform org on GitHub) is the
reference implementation this whole handbook is built against, a Maven,
Spring Boot service. This case study walks through the concrete, named
investigation that shaped the SCA pipeline's design, in particular why it
scans an SBOM instead of the raw Maven cache.

## The starting problem

Early local testing pointed OSV Scanner at `~/.m2/platform-backend`,
a clean Maven local cache populated only by `mvn dependency:resolve`. This
gave 46 critical findings, scoped correctly to the project's actual
resolved dependencies. Widening the scope made things worse, not better:

| Scope | Critical findings | Why |
|---|---|---|
| `dependency:resolve` only | 46 | Correct scope, matches what the SBOM later contains |
| `dependency:resolve` + `dependency:resolve-plugins` | 113 | Pulls in Maven build tooling, not runtime dependencies, all false positives |
| Full `~/.m2/repository` cache | 91 | Too broad, includes unrelated libraries never used by this project |

A separate run the following week, comparing Trivy and OSV-Scanner
directly against the Maven cache, found Trivy's `fs` scan of `pom.xml`
came back clean (one Medium finding, for an already-patched but
not-yet-recognized vulnerability), while OSV-Scanner alone reported over
300 CVEs, 34 of them critical, against the same cache. The two tools
diverged this sharply specifically because OSV-Scanner reads POM files
inside the Maven cache directly, and does not distinguish a library's own
declared (and possibly overridden) dependency versions from what the
project actually resolved and uses. Trivy's `fs` scanner does not have
this same failure mode against `pom.xml`, but neither tool's cache-based
scan is a substitute for scanning the SBOM, see
[explanation/why-sboms.md](../explanation/why-sboms.md) for the mechanism
behind why.

## The worked example

The specific case that made the mechanism concrete: `platform-backend`
depends on `micrometer-core 1.16.5`. Micrometer's own POM file, bundled in
the Maven cache, declares its own dependency on `tomcat-embed-core
8.5.100`. But `platform-backend`'s own `pom.xml` pins `tomcat.version` to
`11.0.22`. Maven's own resolution (nearest-definition, first-come-wins,
BOM order, see [explanation/why-sboms.md](../explanation/why-sboms.md) for
the mechanics) picks `11.0.22` as the version actually used. The generated
SBOM reflects that resolution: it contains `11.0.22`, and
`tomcat-embed-core 8.5.100` never appears in it. Scanning the SBOM with
OSV-Scanner against `11.0.22` came back clean, correctly, because that's
what actually ships.

## The Maven Central rate limit

While testing Trivy against artifacts not yet in the local cache, Maven
Central began returning HTTP 429 (Too Many Requests), it interpreted the
volume of requests from a single CI-runner IP as abuse. This is
documented, with Trivy's own recommended mitigation
(`--offline-scan`), in
[how-to/troubleshoot-maven-rate-limit.md](../how-to/troubleshoot-maven-rate-limit.md).
This is also part of why `mvn dependency:resolve` runs as its own explicit
CI step ahead of SBOM generation, see
[reference/pipeline-sca.md](../reference/pipeline-sca.md), rather than
letting Trivy resolve artifacts on demand during the scan itself.

## `platform-ui`'s companion story: the malware false positive

While `platform-backend` surfaced the Maven false-positive problem,
`platform-ui`'s npm-based testing surfaced a different, unrelated false
positive worth recording here since it shaped the case for running
multiple tools (see
[explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md)):
Dependency-Track flagged a malware finding (`MAL-2022-4051`) against
`platform-ui`'s legitimate dependency `jQuery-QueryBuilder`. The cause was
a PURL collision: `cyclonedx-npm` lowercases all package names when
generating the SBOM, producing `pkg:npm/jquery-querybuilder@3.0.0`. A
real, unrelated malicious package is registered on npm under that same
lowercase name, `jquery-querybuilder`. Because npm's own registry is
case-insensitive, both the legitimate `jQuery-QueryBuilder` and the
malicious `jquery-querybuilder` collapse into the same PURL once
lowercased, and Dependency-Track (which relies on the PURL as its lookup
key against OSV) couldn't tell them apart. Notably, this same malware flag
was **not** caught by `npm audit`, only by Dependency-Track via OSV,
concrete evidence for why relying on a single tool and a single database
leaves gaps.

## What this shaped in the final pipeline

- SCA scans the generated SBOM (`target/bom.json`), not the raw
  dependency cache, for both Maven and npm.
- `mvn dependency:resolve -q` runs as an explicit, separate CI step before
  SBOM generation.
- Both Trivy and OSV-Scanner run against the SBOM, gated through the
  shared `parse_sarif.evaluate()` CVSS-score model.
- The `~/.m2/repository` cache (not the whole `~/.m2` directory, which
  risks cache corruption) is cached between CI runs, keyed on
  `hashFiles('**/pom.xml')`.

See [reference/pipeline-sca.md](../reference/pipeline-sca.md) for the
resulting workflow, and
[explanation/why-sboms.md](../explanation/why-sboms.md) for the general
explanation this case study is the evidence for.