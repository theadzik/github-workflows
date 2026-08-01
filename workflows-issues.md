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

**The cosign authentication problem.** cosign reads the image manifest before signing it,
and it did so anonymously — twice. First with the ambient `docker login` (the same
`~/.docker/config.json` that `regctl` and the docker CLI used successfully in that job),
then again in `2026.7.31-rc2` with `--registry-username` / `--registry-password` passed
explicitly. Both ended at Docker Hub's unauthenticated pull rate limit.

The account was not over its limit: `regctl manifest head` — an authenticated manifest
read — succeeded seconds earlier in the same job.

What was ruled out, each by experiment rather than reasoning:

| Hypothesis | Test | Result |
| --- | --- | --- |
| `secrets: inherit` does not reach a workflow in another repo | `docker/login-action` output, step env | `Login Succeeded!`, `REGISTRY_PASSWORD: ***` — secret arrives |
| The flags are ignored by cosign | auth-required local registry, credentials only via flags | 404 with flags vs `UNAUTHORIZED` without — flags work |
| Broken in the pinned v3.0.6, fixed later | same test on v3.0.6 and v3.1.2 | both authenticate |
| Only works for Basic-auth registries, not Docker Hub's bearer flow | same test against ghcr.io | `DENIED` without flags, 404 with — bearer flow works |
| Docker Hub answers with 429 before issuing a challenge | unauthenticated GET to the manifest | `401` + `WWW-Authenticate: Bearer` — challenge is issued |
| The flags never reach the failing call | read `sign.go` at v3.0.6 | `regOpts.ClientOpts(ctx)` is passed to the `ociremote.SignedEntity` that raises `accessing image` |

So the plumbing is correct everywhere it can be observed, and the runner behaviour remains
unexplained.

**What makes it silent.** For a public repository, Docker Hub's token endpoint answers a
pull-scope request with HTTP 200 and an *anonymous* token even when credentials are absent
or rejected. cosign sees a valid token and no error; the problem only appears later as a
rate limit on a request that was never authenticated.

**The fix.** Mint the bearer token in the step with `curl -f -u` and hand it to cosign via
`--registry-token`, for both `sign` and `verify`. cosign then sends
`Authorization: Bearer` directly and negotiates nothing. Validated against a bearer-flow
registry: minted token → authenticated, no token → `DENIED`. It also converts the silent
degradation into a loud 401 on the credentials themselves. `--registry-password` is basic
auth, which is right for a Docker Hub PAT but leaves the token exchange to cosign;
`--registry-token` is the bearer token that exchange would have produced.

**Closed 2026-07-31** by
[run 30649345089](https://github.com/theadzik/blog/actions/runs/30649345089), which ran the
whole chain against Docker Hub for the first time: build → layout → scan → push by digest
→ sign → attest SBOM → attest provenance → verify → publish tag. The digest now carries
three referrers, all `application/vnd.dev.sigstore.bundle.v0.3+json`:

| Referrer | Predicate |
| --- | --- |
| `09f333a…` | `https://spdx.dev/Document/v2.3` |
| `10ba8ab…` | `https://sigstore.dev/cosign/sign/v1` |
| `aa39e59…` | `https://slsa.dev/provenance/v1` |

What is *not* closed is why it worked. See issue 7.

Pull requests call the workflow with `push: false`, so nothing beyond the build and scan
runs there. Before the alpha build above, the links had only been rehearsed locally
against a scratch registry: regctl pushing a layout at its digest with no tag written,
cosign signing that digest and verifying it, `imagetools create` retagging without
changing the digest. Keyless OIDC and Docker Hub were never in the picture.

## 3. cosign 3.x signature format versus Kyverno 1.18.2

**Status:** open, but narrowed to one question.

cosign is pinned to `v3.0.6`, which has no `--new-bundle-format` flag. Where it puts the
signature turned out to depend on the signing config: locally, with a rekor-less
`--signing-config`, it wrote a `sha256-<digest>` tag; on the runner with the default
config it wrote an **OCI referrer** carrying the predicate
`https://sigstore.dev/cosign/sign/v1`. Neither is the classic `sha256-<digest>.sig` that
2.x produced.

The signing identity is confirmed correct — the certificate SAN on the published signature
reads:

```text
URI:https://github.com/theadzik/github-workflows/.github/workflows/build-and-push.yaml@68e8afe…
```

which the policy's `subjectRegExp`
(`^https://github.com/theadzik/.+/.github/workflows/build-and-push.yaml@.+$`) matches.

So the only open question is whether `verifyImageSignatures` reads referrer-style Sigstore
bundles. Kyverno 1.18.2 is sigstore-go based and already verifies the bundle-format
attestations `actions/attest` produces, so it very likely does — but "very likely" is not
a basis for a policy that gates admission.

**Now testable:** `theadzik/zmuda-pro-blog:tellme` carries a real signature with a matching
identity. Admitting a pod from it, or letting the background scan reach it, answers this —
provided the 429s in issue 4 do not mask the result.

If it turns out to want the classic layout, the fix is one line: pin a 2.x release in the
installer's `cosign-release` input.

## 4. Kyverno is not verifying anything today

**Status:** open — and *not* independent of this repository after all. Same root cause as
issue 7: the Docker Hub account has no pull budget left, so Kyverno's manifest fetches 429
exactly as cosign's did.

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

blog tracks the release candidates on `main` and is the only repository exercising the
push path; [homelab#331](https://github.com/theadzik/homelab/pull/331) is still open and
its three callers run with `push: false`, so they cover the build and layout only.

Repin to `v2.0.0` once issues 3 and 7 are closed.

## 7. Docker Hub pull quota is permanently exhausted

**Status:** root cause found 2026-08-01. Was filed as "cosign authenticates
non-deterministically"; that framing was wrong.

`cosign sign` failed with Docker Hub's unauthenticated pull rate limit at 06:50, 07:07 and
16:41, succeeded at 16:59, then failed again at 17:23 — with a byte-identical invocation
throughout. Chasing that as an authentication fault cost most of a day and produced four
release candidates. It was not an authentication fault.

The probe in blog's `cosign-auth-probe.yaml` tests each credential mechanism against an
already-published digest, reading the account's pull quota either side of every attempt:

```text
mechanism        result   account pull quota
ambient-login    exit=1   quota     0 -> 0     saw: TOOMANYREQUESTS
written-config   exit=1   quota     0 -> 0     saw: TOOMANYREQUESTS
user-password    exit=1   quota     0 -> 0     saw: TOOMANYREQUESTS
registry-token   exit=1   quota     0 -> 0     saw: TOOMANYREQUESTS
```

Zero remaining, on a reading taken with a token minted from the account's own credentials.
Re-run eleven hours later on a one-hour window: still zero. The account is not spiking over
its limit, it is pinned at it.

**What consumes it.** The budget is 200 pulls per hour. ArgoCD Image Updater reconciles 20
images every ~2 minutes; Kyverno re-verifies every `theadzik/*` pod, pulling manifests and
referrer bundles, and retries when those 429; CI builds add a handful each. Kyverno's
retries make it self-reinforcing.

**Why it was so hard to see.** Docker Hub reports the failure as *"You have reached your
unauthenticated pull rate limit"* regardless, which reads as an authentication problem. Two
further facts kept the wrong theory alive: an authenticated `regctl manifest head` had
succeeded seconds earlier in the same job, and every credential mechanism demonstrably
works when tested locally.

**The fix.** Migrate images to ghcr.io, which has no pull rate limits for public images.
Implemented in the build workflow first; cluster secrets and ArgoCD follow. Reducing the
pollers would buy margin against the same ceiling rather than removing it.

## 8. Reusable workflow permissions, for the record

**Status:** settled 2026-08-01 by experiment, since the documentation does not state it.

| Called workflow | Caller grants | Result |
| --- | --- | --- |
| omits `permissions` | `packages: write` | inherits it — the job token shows `Packages: write` |
| declares `packages: write` | only `contents: read` | `startup_failure` |

So a called workflow that declares permissions *replaces* what it would inherit, and
cannot ask for more than the caller holds. `build-and-push.yaml` therefore declares none:
that is what lets one workflow serve a Docker Hub caller and a GHCR caller, since only the
latter needs `packages: write`.
