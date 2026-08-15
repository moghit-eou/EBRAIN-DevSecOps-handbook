# Case Study: `platform-ui`

`platform-ui` (Medical Informatics Platform org on GitHub) is the second
reference implementation this handbook is built against, an Angular, npm
service. Where the [`platform-backend` case study](platform-backend.md)
covers the Maven-side false-positive investigation, this one covers the
npm-side finding that shaped the case for running multiple SCA tools
rather than one: a malware false positive caused by a PURL casing
collision.

## The malware false positive

While testing SCA tooling locally against `platform-ui`, Dependency-Track
flagged a malware finding (`MAL-2022-4051`) against `platform-ui`'s
legitimate dependency `jQuery-QueryBuilder`.

**The cause was a PURL collision.** `cyclonedx-npm` lowercases all package
names when generating the SBOM, producing
`pkg:npm/jquery-querybuilder@3.0.0`. A real, unrelated malicious package
is registered on npm under that same lowercase name,
`jquery-querybuilder`. Because npm's own registry is case-insensitive,
both the legitimate `jQuery-QueryBuilder` and the malicious
`jquery-querybuilder` collapse into the same PURL once lowercased, and
Dependency-Track (which relies on the PURL as its lookup key against OSV)
couldn't tell them apart.

Notably, this same malware flag was **not** caught by `npm audit`, only by
Dependency-Track via OSV, concrete evidence for why relying on a single
tool and a single database leaves gaps, see
[explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md).

## Why this matters beyond the one false positive

The PURL-casing mechanism is a structural property of how CycloneDX PURLs
are generated for npm packages, not a one-off bug in this specific
dependency. Any npm package whose real name differs from another
package's name only by case is exposed to the same collision. This is
worth flagging during triage of any future npm-ecosystem malware or
typosquat finding, verify the actual package identity (registry page,
maintainer, publish history) before assuming a PURL match is correct,
rather than trusting the PURL string alone.

## What this shaped in the final pipeline

- SCA scans the generated SBOM for `platform-ui` the same way it does for
  `platform-backend`, via `cyclonedx-npm` against the installed
  `node_modules/` tree (which requires `npm ci`, not `npm install`, see
  [explanation/why-sboms.md](../explanation/why-sboms.md)).
- Both Trivy and OSV-Scanner run against the SBOM, rather than a single
  tool, precisely because a single tool and a single database (as with
  `npm audit` here) can miss findings the other catches, see
  [explanation/why-two-sca-tools.md](../explanation/why-two-sca-tools.md).
- The `~/.npm` cache is restored between CI runs, keyed on
  `hashFiles('**/package-lock.json')`, never on `package.json`, see
  [reference/pipeline-sca.md](../reference/pipeline-sca.md#caching).

See [reference/pipeline-sca.md](../reference/pipeline-sca.md) for the
resulting workflow, and
[case-studies/platform-backend.md](platform-backend.md) for the companion
Maven-side investigation that shaped the SBOM-first design more broadly.
