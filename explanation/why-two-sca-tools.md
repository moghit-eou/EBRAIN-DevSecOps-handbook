# Why Run Two SCA Tools, Not Just One

Trivy and OSV-Scanner are both run against the same SBOM in the SCA
pipeline, rather than picking one. This was a deliberate decision based on
observed differences between the two, not redundancy for its own sake.

TODO : clean all repitiaon  , mention that gaps between database , sync of tools , some tools put cve score  unkonwn , anyway -> main reason they don't catch the same thing

## They don't always catch the same vulnerabilities

During local testing, one tool sometimes reported a finding with no
severity score at all (marked "unknown"), where the other tool supplied an
actual score for the same or a related finding. Some CVEs were also new
enough that database coverage between the two differed. Relying on either
tool alone meant accepting gaps the other tool could fill.


## They have meaningfully different CLI behavior around failing a build

- **Trivy** makes it straightforward to fail a build based on severity, it
  accepts a direct `--severity` flag.
- **OSV-Scanner** does not expose an equivalent simple flag for failing
  based on a severity threshold.

This difference is *why* the pipeline uses a custom SARIF parser
(`parse_sarif.py`) instead of relying on either tool's native fail
behavior; see
[why-cvss-based-gating.md](why-cvss-based-gating.md). Rather than have
Trivy gate on its own flag and OSV-Scanner gate on something else
entirely, both tools' outputs are normalized to SARIF and evaluated
through the same `security-severity`-based logic, which produces a
consistent PASSED/WARNING/FAILED decision regardless of which tool found
the issue.

## Trade-offs observed during evaluation

- OSV-Scanner is backed by Google's OSV database and updates its CVE data
  quickly, but its CLI lacks the granular flags Trivy offers for directly
  controlling build-failure behavior.
- Trivy scans both filesystem/SBOM targets and container images with one
  consistent CLI, and supports the `--ignorefile` suppression mechanism
  used across this pipeline.

Neither tool was found to be a strict superset of the other in coverage or
usability, which is the core justification for running both rather than
choosing one.

