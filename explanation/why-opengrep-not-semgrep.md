# Why OpenGrep, Not Semgrep

OpenGrep, not Semgrep, is the SAST tool used across the SAST pipeline and
the Dockerfile SAST half of Container Scanning.

## The relationship between the two tools

Semgrep put some of its critical features behind a paid platform tier. The
open source community forked Semgrep and created OpenGrep, with those
previously paid features available for free. Both tools use the same CLI
commands and rule format, OpenGrep is not a different tool to learn, it's
the same interface with more functionality available at no cost. In short:
OpenGrep is the Semgrep free tier, without the paid-tier feature gate.

## What OpenGrep rules actually are

OpenGrep rules are YAML files that define specific patterns to find
vulnerabilities in code structure, not just plain text matching. The tool
parses code into an AST (abstract syntax tree) to understand logic and
detect dangerous patterns, like insecure function calls, structurally
rather than textually.

## Choosing rulesets

For a CI pipeline, stacking specific packs (for example `p/python`,
`p/secrets`) is preferred over `auto`, which pulls in everything and can
be noisy. This is a live tradeoff: keeping to more general/default
rulesets risks missing things, while adding more specific rulesets
(Docker-, TypeScript-, or React-specific, for example) tends to increase
noise and false positives. Deciding which ruleset to include for a given
project is an ongoing part of running this pipeline, not a one-time
setting. For the exact `SEMGREP_CONFIG_RULESETS` values used by each
reference implementation, and guidance on picking rulesets for a new
repository, see
[reference/pipeline-sast.md](../reference/pipeline-sast.md#choosing-semgrep_config_rulesets-for-a-new-repository)
rather than duplicating that table here.

## What else was considered

See [why-these-tools.md](why-these-tools.md) for the full comparison
against Snyk and SpotBugs/FindSecBugs, both of which were tested and not
selected as the primary SAST tool, for reasons unrelated to the
OpenGrep-vs-Semgrep choice specifically (cost/offline limitations for
Snyk, noise and legacy status for SpotBugs/FindSecBugs).

## Related

- [reference/pipeline-sast.md](../reference/pipeline-sast.md)
- [reference/gate-status-rule-severity.md](../reference/gate-status-rule-severity.md)
