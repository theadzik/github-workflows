# github-workflows

Reusable GitHub Actions workflows shared across [theadzik](https://github.com/theadzik)'s
repositories. This repository is public so that public callers can use it; a private
reusable workflow is only callable from private repositories owned by the same user.

## `build-and-push.yaml`

Builds a container image once into an OCI layout, scans it there, and only then pushes
it — by digest, then signs and attests that digest, and publishes tags last.

```text
build → OCI layout on disk → Trivy → push by digest (no tag)
      → cosign sign → attest SBOM → attest provenance → cosign verify → publish tags
```

**Nothing unscanned reaches the registry.** The build exports to `type=oci` on disk and
Trivy scans the layout in place. A finding fails the job before a single byte is pushed.
The same exporter serves pull requests, which simply never push what they built.

**One artifact all the way through.** The digest comes from the layout's `index.json` —
the digest that will exist in the registry, byte for byte. The scan, the push, the
signature and both attestations all reference it. Nothing is rebuilt or re-exported in
between.

**The tag is the gate.** ArgoCD Image Updater discovers images by listing tags, so an
image pushed at its digest with no tag is inert: nothing can select it. `regctl` does
that push — `docker buildx imagetools create` refuses to run without `--tag` and skopeo
rejects a digest destination, so it is the one tool here that reproduces what buildkit's
`push-by-digest` exporter used to do. Tags go up last, via `imagetools create`, which
re-pushes the same manifest under each tag rather than wrapping it in a new index, so
the digest is unchanged and the referrers attached to it stay findable.

**Signed, not just attested.** `actions/attest` produces signed statements *about* the
image; `cosign sign` produces a signature *on* it, which is what Kyverno's
`verifyImageSignatures` looks for. Both use the same keyless workflow identity. The
signature is verified with `cosign verify` before any tag is published, so an identity
the cluster would refuse fails the build instead of the rollout.

### Usage

Publishing to **ghcr.io** — no registry secret needed, the job token does it:

```yaml
jobs:
  build-and-push:
    uses: theadzik/github-workflows/.github/workflows/build-and-push.yaml@<commit-sha> # v2
    permissions:
      contents: read
      id-token: write
      attestations: write
      artifact-metadata: write
      packages: write          # ghcr.io only
    with:
      registry: ghcr.io
      image-name: my-image
      tags: |
        type=raw,value=main
        type=sha
      push: ${{ github.event_name != 'pull_request' }}
    secrets: inherit
```

Publishing to **docker.io** is the same call without `packages: write` and without the
`registry` input; it authenticates with the caller's `DOCKERHUB_TOKEN`, which
`secrets: inherit` passes.

**The `permissions` block belongs to the caller, and it is not optional.** This workflow
deliberately declares none of its own, because a called workflow that declares permissions
*replaces* what it inherits rather than adding to it — and one that asks for more than the
caller granted fails the run at startup. Both were verified rather than assumed. Omitting
the block here is what lets a Docker Hub caller and a GHCR caller share one workflow.

| | `docker.io` | `ghcr.io` |
| --- | --- | --- |
| Namespace | `vars.DOCKERHUB_USERNAME` | the GitHub owner |
| Credential | `secrets.DOCKERHUB_TOKEN` | the job's `GITHUB_TOKEN` |
| Extra permission | none | `packages: write` |
| Pull rate limits | 200/hour per account | none for public images |

`dhi.io` (or whatever `extra-registry` names) is logged in with the Docker account
regardless of where the build publishes, since that is where base images come from. That
login is skipped when no `DOCKERHUB_TOKEN` is available.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `image-name` | *required* | Image repository name, appended to `<registry>/<username>/`. |
| `context-path` | `.` | Docker build context. |
| `git-ref` | *(triggering ref)* | Ref to check out. |
| `fetch-depth` | `1` | Checkout depth. Use `0` when the build reads `.git`. |
| `tags` | `type=raw,value=latest` | `docker/metadata-action` tag spec, one per line. |
| `build-args` | *(none)* | Build args, one `KEY=value` per line. |
| `platforms` | `linux/amd64` | Multi-platform is allowed, but Trivy scans only one platform of the index and the rest are signed unscanned. The job warns when you opt in. |
| `push` | `true` | `false` builds and scans only, for pull requests. |
| `scan` | `true` | Fail on fixable findings before anything is pushed. |
| `scan-severity` | `HIGH,CRITICAL` | Trivy severities that fail the build. |
| `registry` | `docker.io` | `docker.io` or `ghcr.io`. `ghcr.io` uses the job token and needs `packages: write`. |
| `registry-username` | *(owner for ghcr.io, `vars.DOCKERHUB_USERNAME` otherwise)* | Registry namespace and login user. |
| `extra-registry` | `dhi.io` | Registry for base image pulls, always logged in with the Docker account. Empty to skip. |
| `timeout-minutes` | `30` | Job timeout. |

### Outputs

| Output | Description |
| --- | --- |
| `digest` | Digest of the image that was built, scanned and — when `push` is `true` — signed and tagged. |
| `image-ref` | Image reference without a tag. |

### Secrets

| Secret | Description |
| --- | --- |
| `DOCKERHUB_TOKEN` | Optional. Registry password for `docker.io`, and always the credential for `extra-registry`. Not needed when publishing to `ghcr.io` with no `extra-registry`. |

### Pinned tools

`regctl` is installed with `regclient/actions/regctl-installer`, pinned by SHA and asked
for a specific `release`. That installer verifies the downloaded binary with
`cosign verify-blob` against regclient's release identity — **but only if cosign is
already on `PATH`**, otherwise it logs that metadata is unavailable and installs an
unverified binary. That is why `Install Cosign` runs first; reordering the two steps
silently drops the check.

`cosign` is pinned via the installer's `cosign-release` input rather than left to its
default. cosign 3.x writes the signature as an OCI referrer carrying the predicate
`https://sigstore.dev/cosign/sign/v1`; 2.x wrote a classic simple-signing
`sha256-<digest>.sig` tag. Which of the two an admission controller accepts is worth
confirming against the cluster rather than inheriting from an installer default that can
change. Dependabot does not track this input either.

## Versioning

Callers should pin to a commit SHA with the tag in a trailing comment. Dependabot's
`github-actions` ecosystem updates job-level `uses:` references, so pinned callers get a
pull request when a new tag is released here.

| Tag | Shape |
| --- | --- |
| `v2` | Builds to an OCI layout, scans it on disk, then pushes by digest with `regctl`. Nothing unscanned reaches the registry. Multi-platform allowed with single-platform scan coverage. |
| `v1` | Builds straight into the registry (`push-by-digest`), then scans the pushed digest. Multi-platform rejected. |

Inputs and outputs are identical between the two, so moving from `v1` to `v2` is a pin
change only.
