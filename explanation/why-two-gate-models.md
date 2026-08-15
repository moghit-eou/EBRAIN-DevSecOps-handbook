# Why Two Separate Gate Models Exist

The pipelines use two gate models that are deliberately kept separate and
are not interchangeable: the CVSS-score gate and the rule-severity gate.

## The reason they can't be one model

A CVSS score describes the severity of a **known vulnerability** (a CVE),
something with an entry in a vulnerability database, a CVSS vector, and a
numeric score. Trivy and OSV-Scanner report exactly this kind of finding,
because they match dependencies and image layers against vulnerability
databases.

OpenGrep and Hadolint don't do that. They match source code or Dockerfile
instructions against **rule patterns** (an AST-based pattern for OpenGrep,
a linting rule for Hadolint), there is no CVE, and therefore no CVSS score
to read. A rule either matched at error severity, or it didn't. Forcing
these two fundamentally different kinds of findings through the same
CVSS-threshold logic would mean inventing a fake severity score for
pattern-matched findings that never had one, which would misrepresent what
was actually found.

## The two models, side by side

| | CVSS-score gate | Rule-severity gate |
|---|---|---|
| Used by | Trivy, OSV-Scanner | OpenGrep, Hadolint |
| What's being measured | Known-vulnerability severity (CVSS) | Rule/pattern match severity |
| Decision logic | Highest `security-severity` across all SARIF results, compared to fixed thresholds | Each tool's own severity flag (`--severity=ERROR`, `--failure-threshold error`) decides directly |
| States | PASSED / WARNING / FAILED / ERROR | PASSED / FAILED / ERROR |
| Shared logic | `parse_sarif.evaluate()`, one function for both tools | None shared; each tool's own exit code is authoritative |

Full detail on each: [gate-status-cvss.md](../reference/gate-status-cvss.md),
[gate-status-rule-severity.md](../reference/gate-status-rule-severity.md).

## Where this shows up in a single pipeline

The Container Scanning pipeline runs both models in the same workflow, on
purpose: `container_scan.py --scan-type sca` (Trivy/OSV-Scanner against
the image) uses the CVSS-score gate, while
`container_scan.py --scan-type sast` (Hadolint/OpenGrep against the
Dockerfile) uses the rule-severity gate, even though both run from the
same orchestrator CLI and the same workflow file. Keeping the two gate
functions separate, rather than merging them into one general "is this bad
enough to fail" function, is what makes it possible for one pipeline to
correctly evaluate both kinds of findings without conflating them.

## Relation to the OWASP contribution

Normalizing how these two kinds of gates are handled and documented,
rather than treating them as interchangeable, is one of the two
improvements sent upstream in
[OWASP/DevSecOpsGuideline#107](https://github.com/OWASP/DevSecOpsGuideline/pull/107).
