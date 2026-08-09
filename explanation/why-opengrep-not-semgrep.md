# Why OpenGrep, Not Semgrep

OpenGrep, not Semgrep, is the SAST tool used across the SAST pipeline and
the Dockerfile SAST half of Container Scanning.

Todo: talk about rules and how companies rely on custom rulset in pipeline-sast.md not here 

## The relationship between the two tools

Semgrep put some of its critical features behind a paid platform tier. The
open source community forked Semgrep and created OpenGrep, with those
previously paid features available for free. Both tools use the same CLI
commands and rule format, OpenGrep is not a different tool to learn, it's
the same interface with more functionality available at no cost. In short:
OpenGrep is the Semgrep free tier, without the paid-tier feature gate.

## Rules and rulesets

OpenGrep rules are YAML files that define specific patterns to find
vulnerabilities in code structure, not just plain text matching. The tool
parses code into an AST (abstract syntax tree) to understand logic and
detect dangerous patterns, like insecure function calls, structurally
rather than textually.

Different configs target different concerns:

- `p/java` targets Java-specific security issues.
- `p/owasp-top-ten` focuses on the OWASP Top 10 risks across languages.
- `auto` pulls in everything, which can be noisy.

For a CI pipeline, stacking specific packs (for example `p/python`,
`p/secrets`) is preferred over `auto`, to keep signal-to-noise
manageable. This is a live tradeoff: keeping to more general/default
rulesets risks missing things, while adding more specific rulesets
(Docker-, TypeScript-, or React-specific, for example) tends to increase
noise and false positives. Deciding which ruleset to include for a given
project is an ongoing part of running this pipeline, not a one-time
setting.

## What else was considered

See [why-these-tools.md](why-these-tools.md) for the full comparison
against Snyk and SpotBugs/FindSecBugs, both of which were tested and not
selected as the primary SAST tool, for reasons unrelated to the
OpenGrep-vs-Semgrep choice specifically (cost/offline limitations for
Snyk, noise and legacy status for SpotBugs/FindSecBugs).

## Related

- [reference/pipeline-sast.md](../reference/pipeline-sast.md)
- [reference/gate-status-rule-severity.md](../reference/gate-status-rule-severity.md)