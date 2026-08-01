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
signature is verified before any tag is published, so an identity the cluster would refuse
fails the build instead of the rollout.

That verification names one signer and nothing else:

```text
^https://github\.com/theadzik/github-workflows/\.github/workflows/build-and-push\.yaml@.+$
```

Keyless signing puts the *called* workflow's `job_workflow_ref` in the certificate, so the
identity is this file regardless of which repository triggered the build. Only the ref
after `@` is open, because callers pin different commits. Renaming or moving this workflow
changes the identity and this pattern with it.

### Usage

The caller owns every registry detail — where to publish, who to authenticate as, and with
which credential. This workflow has no knowledge of any particular registry.

```yaml
jobs:
  build-and-push:
    uses: theadzik/github-workflows/.github/workflows/build-and-push.yaml@<commit-sha> # v2
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
      # Logged into for base image pulls only; nothing is published there.
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

`image-name` is the whole path under the registry, namespace included, so a registry that
namespaces differently from its login user needs no special handling here.

**The `permissions` block belongs to the caller, and it is not optional.** A called
workflow cannot hold more than its caller, and one asking for more fails the run at
startup — both verified rather than assumed. `packages: write` is what a registry
authenticating with the job token needs; it is inert for any other registry, so the block
above is the same everywhere.

Secrets are mapped explicitly rather than inherited, since the names here describe a role
rather than a provider.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `image-name` | *required* | Repository path under the registry, namespace included. |
| `context-path` | `.` | Docker build context. |
| `git-ref` | *(triggering ref)* | Ref to check out. |
| `fetch-depth` | `1` | Checkout depth. Use `0` when the build reads `.git`. |
| `tags` | `type=raw,value=latest` | `docker/metadata-action` tag spec, one per line. |
| `build-args` | *(none)* | Build args, one `KEY=value` per line. |
| `platforms` | `linux/amd64` | Multi-platform is allowed, but Trivy scans only one platform of the index and the rest are signed unscanned. The job warns when you opt in. |
| `push` | `true` | `false` builds and scans only, for pull requests. |
| `scan` | `true` | Fail on fixable findings before anything is pushed. |
| `scan-severity` | `HIGH,CRITICAL` | Trivy severities that fail the build. |
| `registry` | *required* | Registry host to publish to. |
| `registry-username` | *required* | User to authenticate to the registry as. |
| `extra-registry` | *(none)* | Further registry to log in to, for base image pulls. Nothing is published there. |
| `extra-registry-username` | *(none)* | User to authenticate to `extra-registry` as. |
| `timeout-minutes` | `30` | Job timeout. |

### Outputs

| Output | Description |
| --- | --- |
| `digest` | Digest of the image that was built, scanned and — when `push` is `true` — signed and tagged. |
| `image-ref` | Image reference without a tag. |

### Secrets

| Secret | Description |
| --- | --- |
| `REGISTRY_PASSWORD` | *required* — password or token for `registry-username`. |
| `EXTRA_REGISTRY_PASSWORD` | Password or token for `extra-registry-username`. |

### Pinned tools

`oras` pushes the built layout to a digest, which nothing already on the runner can do:
`docker buildx imagetools create` refuses to run without `--tag`, and skopeo rejects a
digest destination outright. It is installed with `oras-project/setup-oras`, pinned by
SHA and asked for a specific `version`. That action carries the SHA256 of each official
release and fails on a mismatch, so pinning the action by commit pins the binary as well.

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
| `v2` | Builds to an OCI layout, scans it on disk, then pushes by digest with `oras`. Nothing unscanned reaches the registry. Registry, credentials and namespace all come from the caller. |
| `v1` | Builds straight into the registry (`push-by-digest`), then scans the pushed digest. Docker Hub assumed throughout. |

`v1` to `v2` is not a pin change. Callers must add `packages: write`, pass `registry` and
`registry-username`, map `REGISTRY_PASSWORD` instead of inheriting `DOCKERHUB_TOKEN`, and
move the namespace from the username into `image-name`.
