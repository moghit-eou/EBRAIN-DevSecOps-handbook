# Tool installation flags

```bash
bash ci/setup-tools.sh --install-tool <tool1,tool2,...|all> [--sbom-ecosystem maven|npm|none]
```

## `--install-tool`

Comma-separated list, or `all`. Optional, defaults to `none` (installs
nothing) if omitted.

| Tool | Installed from | Used by |
|---|---|---|
| `trivy` | Pinned release tarball, SHA256-verified | Container Scanning (sca), SCA |
| `osv-scanner` | Pinned GitHub release binary, SHA256-verified | Container Scanning (sca), SCA |
| `opengrep` | Pinned GitHub release binary, SHA256-verified | Container Scanning (sast), SAST |
| `hadolint` | Pinned GitHub release binary, SHA256-verified | Container Scanning (sast) |
| `semgrep-rules` | Cloned from `semgrep/semgrep-rules` at a pinned commit | Container Scanning (sast), SAST |

## `--sbom-ecosystem`

Optional, defaults to `none`. Generates the dependency SBOM used by the SCA
pipeline.

| Value | What runs |
|---|---|
| `maven` | `mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom`, writes `target/bom.json` |
| `npm` | `npx @cyclonedx/cyclonedx-npm`, writes `target/bom.json` |
| `none` | Skipped. Used by pipelines that scan an image directly and don't need an SBOM. |

## Version pinning

Every tool version and its release-asset SHA256 are pinned at the top of
`setup-tools.sh`, not passed as flags. Each download is verified against its
pinned checksum before install; the script exits nonzero on a mismatch
rather than installing an unverified binary.

Versions can be overridden via environment variable (e.g. `TRIVY_VERSION`),
but if you override a version you must also override its matching `_SHA256`
variable, or verification fails by design.

TODO: document the `# renovate:` marker convention, for teams that want
Renovate to bump version and checksum together automatically.