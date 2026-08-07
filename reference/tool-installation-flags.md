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

## Platform requirements

`setup-tools.sh` targets Linux x86_64 and requires `bash`, `sudo`, `curl`,
`tar`, and `sha256sum`. This matches GitHub's `ubuntu-latest` CI runners,
which is the only environment this has been tested against.

It does not run as-is on macOS or native Windows:

- Every download URL points to a Linux-specific release asset (e.g.
  `trivy_..._Linux-64bit.tar.gz`, `hadolint-linux-x86_64`). There is no
  branching for other OSes or CPU architectures (including ARM, e.g. Apple
  Silicon).
- `sha256sum` ships by default on Linux, not on macOS.
- The script uses bash-specific syntax (`set -o pipefail`), so it will not
  run correctly under a plain POSIX `sh`.

**Windows**: use WSL2. It runs a real Linux userspace, so the script works
unmodified inside a WSL2 Ubuntu shell.

**macOS**: not currently supported natively. Either run the script inside
a Linux container that mirrors the CI image, or treat this as an open item
if native macOS support becomes a requirement.