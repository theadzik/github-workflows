# github-workflows

Reusable GitHub Actions workflows shared across
[theadzik](https://github.com/theadzik)'s repositories. This repository is public
so that public callers can use it. A private reusable workflow can only be called
from private repositories owned by the same user.

## `build-and-push.yaml`

Builds a container image into an OCI layout on disk. Scans it there. Pushes it by
digest, signs it, attests it, and publishes the tags last.

```text
build → OCI layout on disk → Trivy scan
      → push by digest (no tag) → cosign sign
      → attest SBOM → attest provenance → cosign verify → publish tags
```

Four rules hold, whatever the caller passes:

1. **Nothing unscanned reaches the registry.** The scan runs while the image is
   still on disk. A finding fails the job before anything is pushed. Every
   platform of a multi-platform index is scanned on its own.
2. **The caller cannot reconfigure the scan.** Trivy looks for three
   configuration files in the working directory, which is the caller's own
   repository. All three are pinned to files this workflow writes.
3. **HIGH and CRITICAL are a floor.** A caller can widen the gate. No input makes
   it smaller.
4. **A tag appears only after the signature does.** An image with no tag cannot
   be selected by ArgoCD Image Updater, so it is inert until the last step.

[**docs/how-it-works.md**](docs/how-it-works.md) explains each of these, and the
smaller decisions behind them. Read it before you change the workflow — the file
itself carries short comments only.

### Usage

The caller owns every registry detail: where to publish, who to authenticate as,
and with which credential. This workflow knows nothing about any registry.

```yaml
jobs:
  build-and-push:
    uses: theadzik/github-workflows/.github/workflows/build-and-push.yaml@<commit-sha> # v3
    permissions:
      contents: read
      id-token: write
      attestations: write
      artifact-metadata: write
      packages: write
    with:
      registry: ghcr.io
      image-name: theadzik/my-image
      registry-username: ${{ github.actor }}
      # Logged into for base image pulls only. Nothing is published there.
      extra-registry: dhi.io
      extra-registry-username: ${{ vars.DOCKERHUB_USERNAME }}
      tags: |
        type=raw,value=main
        type=sha
      push: ${{ github.event_name != 'pull_request' }}
    secrets:
      REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
      EXTRA_REGISTRY_PASSWORD: ${{ secrets.DOCKERHUB_TOKEN }}
```

`image-name` is the whole path under the registry, namespace included. A registry
that namespaces differently from its login user therefore needs no special
handling.

**The `permissions` block belongs to the caller, and it is required.** A called
workflow cannot hold more permission than its caller, and one that asks for more
fails the run at startup. `packages: write` is what a registry using the job
token needs. It is inert for any other registry, so the block above works
everywhere.

Secrets are mapped explicitly rather than inherited. The names describe a role,
not a provider.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `registry` | *required* | Registry host to publish to. |
| `image-name` | *required* | Repository path under the registry, namespace included. |
| `registry-username` | *required* | User to authenticate to the registry as. |
| `context-path` | `.` | Docker build context. |
| `git-ref` | *(triggering ref)* | Ref to check out. |
| `fetch-depth` | `1` | Checkout depth. Use `0` when the build reads `.git`. |
| `tags` | `type=raw,value=latest` | `docker/metadata-action` tag spec, one per line. |
| `build-args` | *(none)* | Build args, one `KEY=value` per line. |
| `platforms` | `linux/amd64` | Comma-separated. Every platform is scanned separately. Anything other than `linux/amd64` sets up QEMU, so a cross build can `RUN`. The attested SBOM still describes the first platform only, and the job warns when that applies. |
| `push` | `true` | `false` builds and scans only, for pull requests. |
| `scan` | `true` | Fail on fixable findings. Applies only when `push` is `false`. The gate is `push \|\| scan`, so nothing is ever pushed unscanned. |
| `extra-scan-severity` | *(none)* | Severities to fail on as well as HIGH and CRITICAL: `UNKNOWN`, `LOW`, `MEDIUM`, comma-separated. Naming HIGH or CRITICAL fails the run. |
| `trivyignores` | *(none)* | Path to a Trivy ignore file, relative to the caller's repository. One `.trivyignore.yaml`, or a comma-separated list of plain `.trivyignore` files. Prefer the YAML form: it carries `expired_at` and `statement`. |
| `extra-registry` | *(none)* | Further registry to log in to, for base image pulls. Nothing is published there. |
| `extra-registry-username` | *(none)* | User to authenticate to `extra-registry` as. |
| `timeout-minutes` | `30` | Job timeout. |

### Outputs

| Output | Description |
| --- | --- |
| `digest` | Digest of the image that was built and scanned, and — when `push` is `true` — signed and tagged. |
| `image-ref` | Image reference without a tag. |

### Secrets

| Secret | Description |
| --- | --- |
| `REGISTRY_PASSWORD` | *required* — password or token for `registry-username`. |
| `EXTRA_REGISTRY_PASSWORD` | Password or token for `extra-registry-username`. |

## Testing

[`test-build-and-push.yaml`](.github/workflows/test-build-and-push.yaml) runs on
any pull request that touches `build-and-push.yaml`, the test workflow itself, or
`tests/`. It can also be dispatched by hand against any branch.

It calls `build-and-push.yaml` the way a caller does, once per scenario. `uses: ./`
resolves to the copy on the pull request's merge ref, so what runs is the change
under review.

A fork's pull request cannot grant `id-token: write`, so it skips every scenario
and says so. It does not report a quiet green. Dependabot's runs can grant it, so
they run.

[`tests/README.md`](tests/README.md) lists the fixtures, what each scenario
covers, and the paths a green run cannot reach.

## Versioning

Pin to a commit SHA with the tag in a trailing comment. Dependabot's
`github-actions` ecosystem updates job-level `uses:` references, so pinned callers
get a pull request when a new tag is released here.

| Tag | Shape |
| --- | --- |
| `v3` | Same pipeline as `v2`, but the caller can no longer weaken the scan. HIGH and CRITICAL become a floor, `scan: false` stops applying to a publishing run, and the workflow owns the Trivy configuration instead of reading the caller's. Every platform of a multi-platform index is scanned, and QEMU makes a cross-platform `RUN` work. |
| `v2` | Builds to an OCI layout, scans it on disk, then pushes by digest with `oras`. Nothing unscanned reaches the registry. Registry, credentials and namespace all come from the caller. |
| `v1` | Builds straight into the registry (`push-by-digest`), then scans the pushed digest. Docker Hub assumed throughout. |

A major tag means a caller has to change something, not only move a pin.
[**docs/upgrading.md**](docs/upgrading.md) has the checklist for each step.

**v2 to v3 in short.** `scan-severity` is removed; use `extra-scan-severity`,
which only widens the gate. `scan: false` no longer applies when `push` is true.
A `trivy.yaml`, `.trivyignore` or `trivy-secret.yaml` in the caller's repository
is no longer read. An end-of-life base image now fails the build. Every platform
of a multi-platform build is scanned, where v2 scanned one. A build that passed
under v2 can therefore fail under v3 — which is the point of the change.
