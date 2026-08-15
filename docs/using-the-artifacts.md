# Using what the build produces

Every published image carries a signature, two SBOMs and a provenance statement.
This page is about spending them. It answers "what can I do with this", not "why
is it built this way" — that is [how-it-works.md](how-it-works.md).

Every command here was run against a real image in `ghcr.io/theadzik`.

## What is attached to an image

The artifacts are OCI referrers on the image digest. They are not tags, and they
do not change the digest.

| Artifact | `predicateType` | Made by |
| --- | --- | --- |
| Signature | `https://sigstore.dev/cosign/sign/v1` | `cosign sign --recursive` |
| Image SBOM | `https://cyclonedx.org/bom` | Trivy, over the built layout |
| Source SBOM | `https://cyclonedx.org/bom` | Trivy, over the build context |
| Provenance | `https://slsa.dev/provenance/v1` | `actions/attest` |

An image built before v4 has no source SBOM, so it has three referrers rather
than four. List what a given image actually has:

```shell
oras discover ghcr.io/theadzik/zmuda-pro-blog:main
```

Both SBOMs share a predicate type. Tell them apart by
`metadata.component.type`: `container` for the image, `application` for the
source tree.

## Before you start

You need `cosign`, `oras` and `trivy`, and a registry login:

```shell
oras login ghcr.io -u <user> -p "$(gh auth token)"
```

Two values appear in every verification. The issuer is always GitHub's. The
identity is this workflow's own path, which is what makes a signature meaningful:

```shell
ISSUER=https://token.actions.githubusercontent.com
IDENTITY='^https://github\.com/theadzik/github-workflows/\.github/workflows/build-and-push\.yaml@.+$'
```

Verifying without pinning that identity proves only that *somebody* signed the
image.

## Scenario 1 — trust an image before you run it

The cluster does this at admission. Do it by hand when you are about to pull
something onto a workstation, or when you want to know whether an image really
came from your own pipeline.

```shell
cosign verify \
  --certificate-oidc-issuer "$ISSUER" \
  --certificate-identity-regexp "$IDENTITY" \
  ghcr.io/theadzik/zmuda-pro-blog:main
```

Exit 0 means this workflow built it. A lookalike image pushed by anyone else
fails here, because the certificate identity will not match.

`gh` does the same thing with less typing, if the image was built by a GitHub
workflow in your account:

```shell
gh attestation verify oci://ghcr.io/theadzik/zmuda-pro-blog:main --owner theadzik
```

## Scenario 2 — a new CVE lands, and you need to know if you are affected

This is the scenario SBOMs exist for. You do **not** need to pull the image,
rebuild it, or still have the source.

Pull the attested SBOM out of the registry and scan it against today's database:

```shell
IMG=ghcr.io/theadzik/zmuda-pro-blog

for digest in $(oras discover --format json "$IMG:main" \
                | jq -r '(.referrers // .manifests // [])[].digest'); do
  blob=$(oras manifest fetch "$IMG@$digest" | jq -r '.layers[0].digest')
  oras blob fetch --output - "$IMG@$blob" \
    | jq -r '.dsseEnvelope.payload' | base64 -d > statement.json
  if [ "$(jq -r .predicateType statement.json)" = "https://cyclonedx.org/bom" ]; then
    jq '.predicate' statement.json > sbom.json
  fi
done

trivy sbom --severity HIGH,CRITICAL --ignore-unfixed sbom.json
```

The image passed its scan on the day it was built. This tells you whether it
still passes today, which is a different question and the one that matters during
an incident.

Do the same across every image you run, and you have answered "are we affected"
for the whole estate without touching a single running container.

**Two tools that look right and are not.**

`cosign download attestation` fails with `no attestations found`. It looks for
the older `.att` tag layout; `actions/attest` writes OCI referrers. Use `oras`.

`cosign verify-attestation` is a gate, not an extractor. Keyless, it prints
nothing to stdout — it just exits 0 or 1. Use it to check, and `oras` to read.

## Scenario 3 — where did this image come from?

Something is running in the cluster and you want its origin: which repository,
which commit, which workflow run. The provenance carries all three.

```shell
oras blob fetch --output - "$IMG@$blob" \
  | jq -r '.dsseEnvelope.payload' | base64 -d \
  | jq '{
      workflow: .predicate.buildDefinition.externalParameters.workflow,
      builder:  .predicate.runDetails.builder.id
    }'
```

```json
{
  "workflow": {
    "path": ".github/workflows/manual-build.yaml",
    "ref": "refs/tags/2026.8.1"
  },
  "builder": "https://github.com/theadzik/github-workflows/.github/workflows/build-and-push.yaml@069e91f4..."
}
```

Useful for three questions:

- Which commit is this image? The `revision` and `ref` are recorded, so you do
  not have to trust a tag.
- Is this ours at all? An image nobody can produce provenance for did not come
  from this pipeline.
- Which workflow version built it? The builder id names the exact commit of
  `build-and-push.yaml`, so you can tell whether an image predates a gate you
  have since added.

That last one is how you find images built before a security fix landed.

## Scenario 4 — what did the *build* use?

The image SBOM lists what ships. On a multi-stage build that is a small list: the
blog's runtime image is nginx plus static files, and holds no JavaScript packages
at all.

The source SBOM answers the other half. It lists the dependencies the build
consumed, which is where a supply-chain compromise would sit.

Pick the one you want by `metadata.component.type`:

```shell
jq 'select(.metadata.component.type == "application")' sbom.json   # build-time
jq 'select(.metadata.component.type == "container")'   sbom.json   # shipped
```

Use the source SBOM when a build tool is compromised — a bundler, a plugin, a
transitive dependency of either. Those never appear in the image, so the image
SBOM cannot answer that question.

## Scenario 5 — enforce it in the cluster

Kyverno verifies the signature today. The policy names the same issuer and
identity used above, so an image signed by any other workflow is denied:

```yaml
attestors:
  - name: cosign
    cosign:
      keyless:
        identities:
          - issuer: "https://token.actions.githubusercontent.com"
            subjectRegExp: "^https://github\\.com/theadzik/github-workflows/\\.github/workflows/build-and-push\\.yaml@.+$"
```

Two things worth knowing before you extend it.

The regexp accepts **any** ref after `@`. That includes old tags of this
workflow, which had weaker gates. Pinning it to release tags is what stops an
image built by `@v1` from being admitted.

Attestations are not checked yet. `verifyImageSignatures` proves who built the
image; it says nothing about whether an SBOM or provenance exists. Requiring
those needs a separate attestation check, and any image built before the
predicate type you require will be denied until it is rebuilt.

## Scenario 6 — a multi-platform image

From v3 the index and every child manifest are signed, so verification works
whichever platform a node resolves to:

```shell
oras discover --format json "$IMG:main" --platform linux/arm64
cosign verify --certificate-oidc-issuer "$ISSUER" \
  --certificate-identity-regexp "$IDENTITY" "$IMG@<child-digest>"
```

The SBOM still describes the first platform only. If you need a per-platform
inventory, that is a gap rather than something to query — see
[how-it-works.md](how-it-works.md).

## Scenario 7 — hand an auditor the evidence

Everything above is the evidence pack, and none of it is a claim you make about
yourself. Each artifact is signed by a workflow identity nobody else can assume:

- the SBOM says what is in the artifact,
- the source SBOM says what built it,
- the provenance says which commit and which run produced it,
- the signature ties all of it to one identity.

Export them with the `oras` loop in Scenario 2. They verify offline against the
sigstore transparency log, so an auditor does not need access to your
repositories.
