# Tool installation flags

TODO: generalize from setup-tools.sh --install-tool syntax.

| Tool | Used by |
|---|---|
| trivy | Container Scanning (sca), SCA |
| osv-scanner | Container Scanning (sca), SCA |
| opengrep | Container Scanning (sast), SAST |
| hadolint | Container Scanning (sast) |
| semgrep-rules | Container Scanning (sast), SAST |

--sbom-ecosystem generates the SBOM afterward, used by SCA only.
