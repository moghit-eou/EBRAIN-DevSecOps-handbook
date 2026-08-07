# DevSecOps Handbook

## Purpose

This handbook helps a team add automated security scanning to their CI
pipeline and block known CVEs and insecure code before merge, without
designing the pipeline from scratch. It generalizes a working reference
implementation into a pattern any repo can adopt.

It grew out of a GSoC 2026 project applying the OWASP DevSecOps Maturity
Model to the EBRAINS community, using the MIP platform as a proof of
concept. The OWASP DevSecOps Guideline sets the general principles; this
handbook is the concrete, step by step layer underneath it.

## Who this is for

A team that already runs CI on pull requests and wants a security scanning
gate, without inventing their own severity policy, suppression workflow, or
gate logic from first principles.

## Scope

Covered now, following the OWASP DevSecOps model, as three independent
pipelines:

- **Container Scanning**: the built image and the Dockerfile itself, using
  a CVSS gate (image CVEs) and a rule-severity gate (Dockerfile SAST)
- **SCA** (Software Composition Analysis): application dependencies via
  SBOM, CVSS gate
- **SAST** (Static Application Security Testing): application source code,
  rule-severity gate

This handbook does not cover manual security review, penetration testing,
or runtime protection. It covers what can be automated and gated on a pull
request.

## How to use this handbook

Content is organized by what you are trying to do, not by reading order:

- Read a **tutorial** when you want to build something end to end for the
  first time.
- Read a **how-to guide** when you already have the pipelines running and
  need to accomplish one specific task.
- Read **reference** when you need a fact, a table, or a flag, and nothing
  else.
- Read **explanation** when you want to understand why something is built
  the way it is, not how to do it.

## Table of contents

1. Tutorial: [tutorials/01-setup-guide.md](tutorials/01-setup-guide.md), stand up all three pipelines in a new repo
2. How-to guides
   - [Suppress a false positive](how-to/suppress-a-finding.md)
   - [Adjust the severity gate](how-to/adjust-severity-gate.md)
   - [Add a new scanner](how-to/add-new-scanner.md)
3. Reference
   - [Tool installation flags](reference/tool-installation-flags.md)
   - [Exit codes](reference/exit-codes.md)
   - [Gate status, CVSS model](reference/gate-status-cvss.md)
   - [Gate status, rule-severity model](reference/gate-status-rule-severity.md)
4. Explanation
   - [Why three independent pipelines](explanation/why-three-independent-pipelines.md)
   - [Why CVSS based gating](explanation/why-cvss-based-gating.md)
   - [Why two gate models](explanation/why-two-gate-models.md)
5. Case study: [reference-implementation.md](case-studies/reference-implementation.md), one real, fully instantiated example

## Status

Draft. The pattern has been implemented and proven on one repo, see the
case study. Sections are being generalized and filled in one at a time;
TODO markers indicate what is not written yet.

## Related

OWASP DevSecOps Guideline, Vulnerability Scanning:
https://owasp.org/www-project-devsecops-guideline/latest/02-Vulnerability-Scanning
