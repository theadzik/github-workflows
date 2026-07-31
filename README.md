# github-workflows

Reusable GitHub Actions workflows shared across [theadzik](https://github.com/theadzik)'s
repositories. This repository is public so that public callers can use it; a private
reusable workflow is only callable from private repositories owned by the same user.

## `build-and-push.yaml`

Builds a container image once, pushes it by digest, then scans, signs and attests that
exact digest — and only publishes tags if all of it passed.

**One artifact all the way through.** The build runs once. Everything after it —
the Trivy scan, the cosign signature, the SBOM and provenance attestations — references
the digest that build produced. Nothing is rebuilt or re-exported in between, so the
bytes that were scanned are provably the bytes that were signed and deployed.

**The tag is the gate, not the push.** ArgoCD Image Updater discovers images by listing
tags, so an image pushed by digest with no tags is inert: nothing can select it, and the
Kyverno `ImageValidatingPolicy` would reject it anyway while it has no signature or
attestations. Publishing tags last means a failed scan leaves an orphan digest rather
than a deployable image. `imagetools create` re-pushes the same manifest under each tag
rather than wrapping it in a new index, so the digest is unchanged and the referrers
attached to it stay findable.

**Signed, not just attested.** `actions/attest` produces signed statements *about* the
image; `cosign sign` produces a signature *on* it, which is what Kyverno's
`verifyImageSignatures` looks for. Both use the same keyless workflow identity. The
signature is verified with `cosign verify` before any tag is published, so an identity
the cluster would refuse fails the build instead of the rollout.

### Usage

```yaml
jobs:
  build-and-push:
    uses: theadzik/github-workflows/.github/workflows/build-and-push.yaml@<commit-sha> # v1
    permissions:
      contents: read
      id-token: write
      attestations: write
      artifact-metadata: write
    with:
      image-name: my-image
      context-path: .
      tags: |
        type=raw,value=main
        type=sha
      push: ${{ github.event_name != 'pull_request' }}
    secrets: inherit
```

`secrets: inherit` passes the caller's `DOCKERHUB_TOKEN`. The registry username comes from
the caller repository's `DOCKERHUB_USERNAME` variable unless `registry-username` is set.
The `permissions` block is required in the caller: a called workflow cannot grant itself
more than the calling job has.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `image-name` | *required* | Image repository name, appended to `<registry>/<username>/`. |
| `context-path` | `.` | Docker build context. |
| `git-ref` | *(triggering ref)* | Ref to check out. |
| `fetch-depth` | `1` | Checkout depth. Use `0` when the build reads `.git`. |
| `tags` | `type=raw,value=latest` | `docker/metadata-action` tag spec, one per line. |
| `build-args` | *(none)* | Build args, one `KEY=value` per line. |
| `platforms` | `linux/amd64` | Single platform only — the pre-push scan loads the image locally. |
| `push` | `true` | `false` builds and scans only, for pull requests. |
| `scan` | `true` | Fail on fixable findings before anything is published. |
| `scan-severity` | `HIGH,CRITICAL` | Trivy severities that fail the build. |
| `registry` | `docker.io` | Registry to push to. |
| `registry-username` | *(`vars.DOCKERHUB_USERNAME`)* | Registry namespace and login user. |
| `extra-registry` | `dhi.io` | Second registry to log in to with the same credentials. Empty to skip. |
| `timeout-minutes` | `30` | Job timeout. |

### Outputs

| Output | Description |
| --- | --- |
| `digest` | Digest of the pushed image. Empty when `push` is `false`. |
| `image-ref` | Image reference without a tag. |

### Secrets

| Secret | Description |
| --- | --- |
| `DOCKERHUB_TOKEN` | Registry token, used for `registry` and `extra-registry`. |

## Versioning

Callers should pin to a commit SHA with the tag in a trailing comment. Dependabot's
`github-actions` ecosystem updates job-level `uses:` references, so pinned callers get a
pull request when a new tag is released here.
