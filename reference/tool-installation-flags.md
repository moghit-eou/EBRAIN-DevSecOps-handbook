# Reference: `setup-tools.sh` Flags and Installed Tools

```bash
bash ci/setup-tools.sh --install-tool <tool1,tool2,...|all> [--sbom-ecosystem maven|npm|none]
```

## Platform support

Tested on **Linux x86_64 / Ubuntu only**, matching the `ubuntu-latest`
GitHub Actions runner image. Not portable as-is to macOS or native
Windows.

| Platform | Status |
|---|---|
| Linux x86_64 (Ubuntu) | Supported, this is what's tested |
| Windows via WSL2 | Works, WSL2 is a real Linux userspace |
| Windows native | Not supported |
| macOS | No working path yet |

TODO: dockerizing the toolchain and publishing it to a registry (GHCR), so
it would run via `docker run` on any OS instead of the install script, is
an idea under discussion with the docs mentor. It is not a decided
direction. Do not treat it as a current option.

## Flags

| Flag | Values | Purpose |
|---|---|---|
| `--install-tool` | comma-separated list, or `all` | Which scanner binaries to install |
| `--sbom-ecosystem` | `maven`, `npm`, `none` | Which SBOM generator to run after tool installation |

## Installable tools

| Tool | Installed from | Verification | Used by |
|---|---|---|---|
| `trivy` | Official release tarball | SHA256-pinned | Container Scanning (SCA half), SCA |
| `osv-scanner` | GitHub release binary | SHA256-pinned | Container Scanning (SCA half), SCA |
| `opengrep` | GitHub release binary | SHA256-pinned | Container Scanning (SAST half), SAST |
| `hadolint` | GitHub release binary | SHA256-pinned | Container Scanning (SAST half) |
| `semgrep-rules` | Cloned from `semgrep/semgrep-rules` at a pinned commit | Git ref pin | Container Scanning (SAST half), SAST |

`should_install "<name>"` returns true if `--install-tool all` was passed,
or if `<name>` appears in the comma-separated `--install-tool` list.

## Version and checksum pinning

Every downloadable tool is pinned to an exact version and SHA256 checksum
at the top of the script:

```bash
# renovate: datasource=github-release-attachments depName=aquasecurity/trivy
TRIVY_VERSION="${TRIVY_VERSION:-v0.71.1}"
TRIVY_SHA256="${TRIVY_SHA256:-3cbae37cd440cd8676e5ce9207fe460b5641c7579a17e9d00f8894928c41a88d}"
```

The `# renovate:` comment lets Renovate bump the version and its matching
checksum together in one automated update. If you override a `*_VERSION`
via environment variable, you must override the matching `*_SHA256` too,
or `download_and_verify` will refuse to install it:

```bash
download_and_verify() {
  local url="$1" dest="$2" sha256="$3"
  curl -fsSL --retry 3 "${url}" -o "${dest}"
  echo "${sha256}  ${dest}" | sha256sum -c -
}
```

## Script safety flags

```bash
set -e          # stop the pipeline if any command fails
set -o pipefail # prevents silent pipeline successes if the curl download drops
set -u          # treat unset variables as an error
trap 'echo "[setup-tools] ERROR: command failed (exit $?) at line $LINENO: $BASH_COMMAND" >&2' ERR
```

On any error, the script stops and prints the failing line number and
command, rather than continuing silently or failing with an unrelated
downstream error.

## SBOM generation (`--sbom-ecosystem`)

| Value | Command run |
|---|---|
| `maven` | `mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom -q` |
| `npm` | `npx --yes "@cyclonedx/cyclonedx-npm@${CYCLONEDX_NPM_VERSION}" --output-file target/bom.json` |
| `none` | No SBOM generated |

`container-scan.yml` scans the built image directly and needs no SBOM, so
it always uses `--sbom-ecosystem none` (or omits the flag).