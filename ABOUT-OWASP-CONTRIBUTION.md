# About This Handbook and the OWASP Contribution

## Relationship to the OWASP DevSecOps Guideline

This handbook is a companion to, not a replacement for, the
[OWASP DevSecOps Guideline](https://github.com/OWASP/DevSecOpsGuideline).
The guideline provides the general model, this handbook documents one
concrete, working implementation of that model against a real Java/Spring
Boot and Angular codebase, including the tool choices, rejected
alternatives, and trade-offs that a general guideline cannot cover.

During early research it became clear that two OWASP repositories exist
for this content, and they are not in sync:

| Repository | Status |
|---|---|
| `OWASP/DevSecOpsGuideline` | The active repository; not yet reflected on the public OWASP guideline website at the time of writing. |
| `OWASP/www-project-devsecops-guideline` | The repository currently in sync with the public website; effectively the older version. |

### Upstream pull request

A pull request improving the OWASP DevSecOps Guideline has already been
opened, based directly on lessons learned while building these pipelines:

- **Title:** docs: add SARIF normalization and exit code handling to
  Section 2-3-5 (Security Gates)
- **PR:** [OWASP/DevSecOpsGuideline#107](https://github.com/OWASP/DevSecOpsGuideline/pull/107)
- **Scope:** two improvements: combining multiple scanners for container
  scanning, and normalizing gate checks across scanners so that tools
  reporting CVSS and tools reporting rule severity are handled
  consistently.

