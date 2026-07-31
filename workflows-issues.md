# Open issues before the next release

Tracking list for `build-and-push.yaml`. The workflow on `main` is the OCI-layout
flow (build → scan on disk → push by digest → sign → attest → verify → tag), released
as `v2.0.0-rc` for testing in the caller repositories. The earlier `v2.0.0` and `v2`
tags were deleted; there is deliberately no stable `v2` until the push path has run
for real. `v1` still points at v1.1.0.

Ordered by what could break a build, not by severity of consequence. Resolved entries stay
in place with their evidence rather than being deleted.

## 1. `trivy-action`'s `input:` with a directory — RESOLVED

**Status:** resolved 2026-07-31, no workflow change needed.

The scan step passes `input: ${{ runner.temp }}/layout` — a *directory* holding an OCI
layout, not the tar file the parameter's description (`reference of tar file to scan`)
suggests. Settled by reading the action at its pinned SHA and replaying its entrypoint
locally with the environment it builds.

The path the value takes is not obvious. `input:` becomes `INPUT_INPUT`, which
`set_env_var_if_provided "TRIVY_INPUT"` writes into a `trivy_envs.txt` that the entrypoint
sources. The entrypoint then runs:

```bash
cmd=("$TRIVY_CMD" "$scanType" "$scanRef")   # scanType=image, scanRef="." by default
```

So the actual command is `trivy image .` with `TRIVY_INPUT` set in the environment — the
positional argument is a bare dot, and the scan target arrives out of band. It works
because `TRIVY_INPUT` takes precedence over the positional argument.

Verified against trivy **v0.70.0**, the version `trivy-action` pins by default (not the
newer 0.72.0 that happened to be on the workstation):

| Case | Result |
| --- | --- |
| Layout dir, `severity=HIGH,CRITICAL`, `exit-code=1` | scans the image manifest, ignores the attestation manifest, reports the 2 fixable HIGH in alpine 3.19, **exit 1** |
| Same layout, `severity=CRITICAL` (none present) | reports clean, **exit 0** |

So the gate fails builds when it should and passes them when it should, on a directory.

Still local-only evidence: homelab's three callers all set `scan: false`, so no CI run has
executed this step. Its CI runs did confirm the layout export and digest read
(`exporting to oci image format` → `Built sha256:8cbbf7db…`). Covering the scan in CI
would take one caller set to `scan: true` with `scan-severity: CRITICAL`, which these
images pass — they have fixable HIGH findings but no CRITICAL ones.

One constraint this exposes: `image-ref` must stay unset. It is the only thing that
changes `scanRef` away from `"."`, and mixing the two would leave two competing targets in
one command.

## 2. The push path has never executed

**Status:** partially closed 2026-07-31 by
[blog run 30610770330](https://github.com/theadzik/blog/actions/runs/30610770330), a real
`2026.7.31-alpha` build. It reached the registry and then failed at `Sign image`:

```text
Error: signing [docker.io/theadzik/zmuda-pro-blog@sha256:456ea2ff…]: accessing image:
GET https://index.docker.io/v2/theadzik/zmuda-pro-blog/manifests/sha256:456ea2ff…:
TOOMANYREQUESTS: You have reached your unauthenticated pull rate limit.
```

**What that run proved.** `regctl image copy` pushed the layout by digest and the digest
check passed, so the by-digest push works against Docker Hub, not just a scratch registry.
The `regctl-installer` ran `cosign verify-blob` on its own binary, confirming the
cosign-before-regctl ordering does what it was ordered for. And the gate behaved
correctly under failure: the job died before `Publish tags`, and Docker Hub has no
`2026.7.31-alpha` tag — an orphan digest, nothing deployable, exactly the designed
failure mode.

**Root cause of the failure.** cosign reads the image manifest before signing it, and it
did not use the `docker login` credentials sitting in `~/.docker/config.json` — the same
file `regctl` and the docker CLI used successfully in that job. It pulled anonymously and
hit Docker Hub's unauthenticated rate limit. Fixed by passing `--registry-username` and
`--registry-password` explicitly to both `cosign sign` and `cosign verify` rather than
relying on ambient keychain behaviour. Why cosign's keychain misses a config other tools
read is still unexplained; the explicit flags sidestep it rather than answer it.

**Still unproven:** `cosign sign`, `cosign verify`, both attestations and `Publish tags`.
The next tag build after repinning is what closes this.

Pull requests call the workflow with `push: false`, so nothing beyond the build and scan
runs there. Before the alpha build above, the links had only been rehearsed locally
against a scratch registry: regctl pushing a layout at its digest with no tag written,
cosign signing that digest and verifying it, `imagetools create` retagging without
changing the digest. Keyless OIDC and Docker Hub were never in the picture.

**How to settle the rest:** push another alpha tag in the blog repo once it pins an rc
carrying the cosign credential fix, then read the referrers off the resulting digest — the
same check settles issue 3.

## 3. cosign 3.x signature format versus Kyverno 1.18.2

**Status:** open. Depends on 2.

cosign is pinned to `v3.0.6`. Version 3 has no `--new-bundle-format` flag: it writes a
Sigstore bundle to a `sha256-<digest>` tag, where 2.x wrote a classic simple-signing
`sha256-<digest>.sig`. Confirmed locally — after signing, the scratch registry listed
exactly one tag, `sha256-ec0cb064…`, with no `.sig` suffix.

Whether `verifyImageSignatures` in the `ImageValidatingPolicy` accepts that shape is
unconfirmed. Kyverno 1.18.2 is recent and sigstore-go based, and it already verifies the
bundle-format attestations `actions/attest` produces, so it very likely does. "Very
likely" is not a basis for a policy that gates admission.

If it turns out to want the classic layout, the fix is one line: pin a 2.x release in the
installer's `cosign-release` input.

## 4. Kyverno is not verifying anything today

**Status:** open. Independent of this repository.

Every pod running a `theadzik/*` image reports `error`, not `pass`, in its PolicyReport:

```text
zmuda-pro-blog:2026.7.5   error   429 Too Many Requests   index.docker.io/v2/theadzik/zmuda-pro-blog/manifests/…
vw-restore:2026.7.1       error   429 Too Many Requests
custom-argocd:v3.4.4      error   429 Too Many Requests
```

Reports re-evaluated at 05:22 UTC on 2026-07-31, so this is current, not stale. With
`failurePolicy: Ignore`, an evaluation error admits the pod — so no image is currently
verified at admission, signature or attestation. Pods that show `pass` are the ones whose
images do not match the `theadzik/*` glob, meaning there was nothing to check.

`dockerhub-image-pull-secret` exists in the `kyverno` namespace and the policy references
it under `credentials.secrets`, yet the fetches still go out rate-limited. Worth finding
out whether the credential is reaching the verifier at all. The same rate limit hit a
local `cosign verify` from a workstation, so it is not specific to the cluster.

Once it verifies for real, `failurePolicy: Fail` is worth considering — under `Ignore`,
any registry hiccup silently admits an unverified image, which is the opposite of what
the policy is for.

## 5. The SBOM round-trips through the registry

**Status:** open. Cleanup, not a defect.

`Extract buildx-generated SBOM` runs `docker buildx imagetools inspect` against the
pushed digest to pull out the SPDX document, then feeds it to `actions/attest`. The OCI
layout on disk already contains that same SBOM as an attestation-manifest blob, so the
round trip is avoidable: reading it locally would remove a registry dependency from the
attestation path and make the step work before the push rather than after.

Left alone for now because the current form is inherited from v1 and known to work,
and changing it while 1 and 2 are unresolved would confuse the diagnosis.

## 6. Caller PRs pin the release candidate

**Status:** in progress.

[homelab#331](https://github.com/theadzik/homelab/pull/331) and
[blog#111](https://github.com/theadzik/blog/pull/111) pin `v2.0.0-rc`, so the two caller
repositories are the test bed for the layout flow. homelab exercises it on every PR
(three callers, `push: false`); blog does not, because `website-checks` excludes workflow
files from its path filter.

Repin to `v2.0.0` once issues 2 and 3 are closed.
