# How to Troubleshoot Package-Registry Rate Limiting

Applies to any ecosystem where a scanner resolves artifact metadata live
against a public package registry (Maven Central, npmjs.org, PyPI, the Go
module proxy) instead of reading only what's already local. Under
repeated CI runs from the same IP range (a shared GitHub Actions runner
pool), registries can rate-limit that traffic.

## How this shows up

**Maven Central**, observed directly while testing Trivy against
artifacts not yet present in the local `~/.m2` cache: repeated requests
from a CI-runner IP were interpreted as abuse, and Maven Central began
returning HTTP 429 (Too Many Requests). The scan step failed outright,
not just returned incomplete results.

The same class of failure applies to any ecosystem where the scanning
tool (not just the build tool) makes live registry calls: npm's registry,
PyPI, and the Go module proxy all apply similar rate limits under
sustained automated traffic.

## Why scanning the SBOM avoids most of this

This pipeline's SCA design, scan the generated SBOM rather than resolving
packages live during the scan, sidesteps this class of failure for the
scan step itself: `trivy sbom` and `osv-scanner scan source --lockfile`
both read a local file, they don't need network access to the registry
to evaluate what's already in the SBOM. See
[why-sboms.md](../explanation/why-sboms.md) for the full reasoning behind
scanning the SBOM instead of the raw dependency cache.

Rate limiting can still occur **before** the SBOM exists, during the
dependency resolve/install step itself (`mvn dependency:resolve`,
`npm ci`, `pip install`, `go mod download`), since that step does talk to
the registry directly. This is why dependency resolution runs as its own
explicit CI step ahead of SBOM generation, see
[integrate-a-new-ecosystem.md](integrate-a-new-ecosystem.md), rather than
letting a scanner resolve artifacts on demand mid-scan.

## Mitigations, by ecosystem

| Ecosystem | Mitigation |
|---|---|
| Maven | Ensure the dependency cache (`~/.m2/repository`) is restored from `actions/cache` before `mvn dependency:resolve` runs, so only new/changed dependencies hit the network. If Trivy itself is resolving artifacts live (rather than scanning the SBOM), pass `--offline-scan` so it only uses what's already cached locally. |
| npm | Ensure `~/.npm` is restored from cache before `npm ci`. `npm ci` only re-downloads what changed since the lockfile hash last matched. |
| Python (pip) | Ensure `~/.cache/pip` is restored from cache before `pip install`. Consider a private package index/mirror for high-frequency CI if rate limiting persists. |
| Go | Ensure `~/go/pkg/mod` is restored from cache before `go mod download`. `GOPROXY` can be pointed at an internal module proxy to reduce direct calls to the public proxy. |

In every case, the underlying fix is the same: make the dependency cache
step effective (correct path, correct lockfile-derived key, restored
*before* the install/resolve step), so the resolve step itself makes as
few live registry calls as possible. See the
[ecosystem matrix](integrate-a-new-ecosystem.md#ecosystem-matrix) for the
exact cache path and key per ecosystem.

## If the rate limit is hit anyway

Treat it as a transient CI infrastructure failure, not a code or
dependency problem:

1. Re-run the job. Rate limit windows are typically short (minutes, not
   hours).
2. Confirm the cache step actually restored (check the workflow log for a
   cache hit vs. a cold cache) before assuming the mitigation above is
   working as intended.
3. If it recurs frequently on the same repository, consider a scheduled
   "warm the cache" job outside of PR-triggered runs, so PR builds are
   more likely to hit a warm cache rather than resolving cold.
