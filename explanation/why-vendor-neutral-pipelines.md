# Why the Pipelines Moved to a Vendor-Neutral, Ecosystem-Matrix Model

## The problem, as originally scoped

> The current documentation is highly redundant. There are separate,
> repetitive guides for setting up CI/CD security pipelines (SAST, SCA,
> SBOM generation via GitHub Actions) for Maven and NPM. The goal of the
> project is a vendor-neutral pipeline that can work on GitHub Actions or
> somewhere else.

This is preserved here verbatim as the working problem statement (source:
project notes), not paraphrased, because it's the thing every decision on
this page is answering.

## What "redundant" actually meant in practice

Before this restructure, `platform-backend` (Maven) and `platform-ui`
(npm) each had their own copy of nearly identical `ci/` scripts and their
own copy of the setup tutorial. Diffing them showed the orchestrators
(`sca_scan.py`, `sast_scan.py`, `container_scan.py`, `parse_sarif.py`), the
gate logic, and roughly 90% of `setup-tools.sh` were byte-for-byte
identical between the two repositories. The only genuine differences were:

- The SBOM generation command (`mvn ... makeAggregateBom` vs `cyclonedx-npm`).
- The dependency cache path and key (`~/.m2/repository` vs `~/.npm`).
- The `SEMGREP_CONFIG_RULESETS` value for OpenGrep.

Everything else, the gate models, the exit-code handling, the SARIF upload
categories, the Container Scanning pipeline in its entirety, was already
ecosystem-agnostic. The documentation didn't reflect that: it repeated the
identical 90% twice instead of documenting it once and parameterizing the
10% that actually varies.

## The fix: one guide, one matrix

Rather than writing a third setup guide for the next ecosystem (Gradle, or
Python, or anything else), the differences were pulled into a single
lookup table, [reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md),
and the tutorial and pipeline reference pages now point at that table
instead of hardcoding one ecosystem's commands. Adding support for a new
ecosystem going forward means adding one row to the matrix and, once the
`setup-tools.sh` gap closes (see the matrix's own note on this), one `case`
branch, not a new tutorial file, not a new case study, not a rewritten
`pipeline-sca.md`.

## "Vendor-neutral" specifically means GitHub-Actions-optional

The second half of the original goal, working "on GitHub Actions or
somewhere else", is handled by a decision that was already true of the
architecture before this restructure, but is worth stating explicitly now
that it's a stated goal rather than an incidental property: every
orchestrator (`sca_scan.py`, `sast_scan.py`, `container_scan.py`) is a
plain Python CLI that reads environment variables and writes SARIF files.
None of them call the GitHub Actions API, use GitHub-specific context
(`${{ github.* }}`), or depend on being invoked from a `.yml` workflow file.
The only genuinely GitHub-specific pieces are:

- The `upload-sarif` step, which uploads to the GitHub Security tab.
- The `actions/cache` steps, which use GitHub Actions' cache backend.

Running these pipelines on a different CI system means replacing those two
step types with the equivalent for that system (for example, GitLab's
`artifacts:reports:sast`/`dependency_scanning` or a generic cache action)
and calling the same orchestrator scripts with the same environment
variables. The orchestration logic itself does not need to change.

## What this did not solve

- **SBOM generation for Gradle and Python is documented, not yet
  automated.** See the matrix's own status table, both are one manual step
  away from being CI-ready, but `--sbom-ecosystem` doesn't have a `gradle`
  or `python` case in the shipped script yet.
- **Raw JavaScript with no package manager has no SBOM path at all**, this
  isn't a gap to close the same way, there's no manifest to build an SBOM
  from. The matrix documents the `trivy fs` substitute and its real
  limitations instead of implying SBOM parity that doesn't exist.
- **A CI-system-agnostic reference implementation for something other than
  GitHub Actions doesn't exist yet.** The claim above (that it's
  architecturally possible) hasn't been demonstrated against a second CI
  system.

## Related

- [reference/ecosystem-matrix.md](../reference/ecosystem-matrix.md), the consolidated matrix itself.
- [tutorials/01-integrate-your-repository.md](../tutorials/01-integrate-your-repository.md), the single tutorial this replaced two ecosystem-specific ones with.
- [why-sboms.md](why-sboms.md), why the SBOM-first approach the matrix generalizes matters in the first place.
- [ABOUT-OWASP-CONTRIBUTION.md](../ABOUT-OWASP-CONTRIBUTION.md), the broader relationship between this handbook and the OWASP guideline it documents an implementation of.
