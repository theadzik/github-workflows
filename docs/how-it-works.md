# How build-and-push.yaml works

This page explains the design of
[`build-and-push.yaml`](../.github/workflows/build-and-push.yaml). The workflow
file has short comments only. The reasons are here.

The language is deliberately plain. Sentences are short. Each one says one
thing.

## The order of the steps

```text
build → OCI layout on disk → Trivy scan
      → push by digest (no tag) → cosign sign
      → attest SBOM → attest provenance → cosign verify → publish tags
```

The order is the whole design. Read it as a sequence of gates. Each gate must
pass before the next one runs.

## One artifact all the way through

The build writes an OCI layout to disk. It does not push to the registry.

Everything after that points at the same layout. Trivy scans it. `oras` pushes
it. syft reads it. Nothing is built a second time.

The digest comes from the layout's `index.json`. That file is the authority. The
build action also reports a digest, but `index.json` is what the registry will
store.

So the bytes that were scanned are the bytes that get signed. They are also the
bytes that get deployed.

A build with `push: false` uses the same steps. It simply never pushes the
layout it made.

## Nothing unscanned reaches the registry

Trivy scans the layout while it is still on disk. A finding fails the job. At
that moment nothing has been pushed, so nothing needs to be removed.

Two inputs control this:

- `push` decides if the image is published.
- `scan` decides if a **non-publishing** run is scanned.

The step condition is `push || scan`. Read it as one rule:

> You can scan without publishing. You cannot publish without scanning.

`scan: false` therefore does nothing on a run that publishes. This is on
purpose. A caller must not be able to turn the gate off and push at the same
time.

### The severity floor

HIGH and CRITICAL always fail the build. This is a floor, not a default.

`extra-scan-severity` adds more severities. It accepts `UNKNOWN`, `LOW` and
`MEDIUM`, in any letter case, with or without spaces.

It cannot remove HIGH or CRITICAL. If you name either one, the run fails with a
message. This is better than ignoring the value. A caller who writes
`extra-scan-severity: CRITICAL` is usually trying to make the gate smaller. They
need to be told that they cannot.

The `Resolve scan severities` step joins the floor and the extra values into one
list. Trivy accepts only one list, so the join must happen somewhere. It happens
in a script because a workflow expression cannot reject a bad value.

That step runs first, and it runs always. A bad value then costs a few seconds.
If the step ran later, it would cost a full build.

## The caller cannot reconfigure the scan

The working directory during the scan is the **caller's** repository. Trivy
looks for configuration files in the working directory. So a caller could change
the scan that guards their own publish.

None of this needs an input. None of it appears in the log as a file name.

Trivy discovers three files. The workflow writes its own copy of each one in
`RUNNER_TEMP` and names it. Discovery then lands outside the workspace.

| File Trivy looks for | What a committed one does | Tested against |
| --- | --- | --- |
| `trivy.yaml` | `scan.skip-dirs` hides the directory that holds the vulnerable package. A blocked build then passes. | `tests/fixtures/vulnerable` |
| `.trivyignore` | Trivy's default `ignorefile`. It hides every finding it lists. | The same fixture. All four of its CVEs scanned clean. |
| `trivy-secret.yaml` | Trivy's default `secret.config`. `disable-rules: [private-key]` stops the secret scanner. The log still shows the scanner as enabled. | An image that carries an RSA private key. |

`.trivyignore.yaml` is **not** discovered this way. It works only when something
names it. It needs no pin. This was checked, not assumed.

A caller **can** still declare an acceptance. The `trivyignores` input names a
file, and that input beats the pinned default. The split is intentional:

- An acceptance the workflow was told about is a decision on record.
- An acceptance it was not told about is never found.

Two details matter if you edit this step:

- The pinned secret config must be `disable-rules: []`. Trivy reads that file on
  its own. A file with no YAML node in it stops the run with
  `secrets config decode error: EOF`. An empty list means "disable nothing", and
  every built-in rule stays active.
- The two paths are appended with `printf`, not written inside the heredoc. The
  heredoc is unexpanded on purpose, so nothing inside it can be substituted.

### Environment variables

A caller **cannot** set environment variables for this workflow. GitHub does not
copy `env` from a caller workflow into a called one. This is true for both the
workflow level and the job level.

`runs-on` is fixed to `ubuntu-latest` in this file, so the runner is a fresh
GitHub-hosted machine. There is no environment to inherit.

This means `TRIVY_CONFIG` and `TRIVY_IGNOREFILE` cannot be injected. That
protection comes from GitHub, not from this file. It holds only while `runs-on`
stays fixed. Do not make it an input.

## An end-of-life base image fails the build

A distribution that has stopped issuing security updates is a special case. An
empty report then means Trivy has nothing to *report*. It does not mean there is
nothing to find. An EOL Alpine scans clean and exits 0.

`exit-on-eol: 1` turns that into a failure. The base image must be one that can
still be patched.

## Secrets are a gate, not a report

The scan configuration lists both `vuln` and `secret` scanners. Both are Trivy
defaults today. They are written down so that a future Trivy release cannot drop
either one without anyone noticing.

A private key in a layer fails the build. `ignore-unfixed` does not hide it.

## The SBOM

The SBOM is CycloneDX, written by syft from the same layout.

SPDX is not used. syft's SPDX writer does not emit an operating-system package.
Trivy's SPDX reader needs that package to work out the distribution. Without it
Trivy reports nothing, which looks exactly like a clean scan.

This was measured. Against `ghcr.io/theadzik` images, `trivy sbom` found zero OS
packages where `trivy image` found twelve.

BuildKit's own `sbom` option is not set, for the same reason. BuildKit writes
SPDX only.

The step then checks its own output. If the SBOM has no `operating-system`
component, the build fails. Attesting such a document would be worse than
attesting nothing.

syft needs to be told the platform. It defaults to `linux/amd64` and fails on a
layout that does not contain it. The step passes the first platform in the list.
Trivy does not need this; it reads the layout's own platform.

## Signing and attestation

`actions/attest` produces signed statements *about* the image. `cosign sign`
produces a signature *on* the image.

Kyverno's `verifyImageSignatures` looks for the signature. An attestation is not
a substitute for it.

Both use the same keyless identity: this workflow's OIDC token.

The workflow then verifies its own signature, before any tag exists. An identity
the cluster would refuse fails the build instead of the rollout.

The verification names one signer:

```text
^https://github\.com/theadzik/github-workflows/\.github/workflows/build-and-push\.yaml@.+$
```

Keyless signing puts the **called** workflow's `job_workflow_ref` in the
certificate. The identity is therefore this file, whichever repository triggered
the build. Only the part after `@` is open, because callers pin different
commits.

The dots are escaped because this is a regular expression. An unescaped dot
would also match a look-alike host or path.

**If you rename or move this workflow, change this pattern too.** Every build
fails at this step until you do.

## Tags go up last

ArgoCD Image Updater finds images by listing tags. An image pushed at its digest
with no tag is therefore inert. Nothing can select it.

So the push writes no tag. The signature and both attestations are added first.
Only then do the tags go up.

`oras` performs the push, because nothing else on the runner can push to a
digest. `docker buildx imagetools create` refuses to run without `--tag`. skopeo
rejects a digest destination.

After the push, the workflow reads the descriptor back from the registry. It
does not trust the push output. A registry that stored something else says so
here, not at admission time.

The tags are written with `docker buildx imagetools create`. That command
re-pushes the same manifest under each tag. It does not wrap it in a new index.
The digest is unchanged, so the referrers attached to it stay findable.

## Multi-platform builds

Multi-platform is allowed. Coverage is not complete.

Trivy scans one platform of the index. syft describes one platform. The other
platforms are signed without a scan and without an SBOM.

This is accepted, not blocked. A build that opts in prints a warning, so the gap
appears in the log of every affected run.

## Pinned tools

`cosign` is pinned with the installer's `cosign-release` input. Version 3.x
writes the signature as an OCI referrer with the predicate
`https://sigstore.dev/cosign/sign/v1`. Version 2.x wrote a
`sha256-<digest>.sig` tag instead. Which one an admission controller accepts
should not change because an installer default changed. Dependabot does not
track this input.

`syft` version is pinned above the action's default of v1.42.3. The
default cannot read the layout this workflow builds. `provenance: mode=max`
makes the layout root an image *index*, and v1.42.3 rejects it with
`unexpected media type ... image.index.v1+json`. Check any new version against a
layout with attestations before you raise it.

`oras` is installed by `oras-project/setup-oras`, pinned by SHA, with an
explicit `version`. That action carries the SHA256 of each official release and
fails on a mismatch. Pinning the action by commit therefore pins the binary too.

## Smaller decisions

**`persist-credentials: false` on the checkout.** Otherwise checkout writes an
`x-access-token` header into `.git/config`. A `COPY .` would then bake that token
into a build layer. `cache-to: type=gha,mode=max` would export it to the Actions
cache, where it outlives the job. Nothing here pushes over git, so the
credential is not needed.

**No `image-ref` on the scan step.** The action runs `trivy image .` and passes
the real target in `TRIVY_INPUT`. Setting `image-ref` replaces the dot. Trivy
then has two targets and uses the wrong one.

**No `hide-progress` on the scan step.** That option is `--quiet`. It removes the
whole log, not only the progress bar. The log is where `Detected OS` and the EOL
error appear. The progress bar is turned off in the configuration file instead.

**`show-suppressed: true`.** The log then names which acceptances actually fired,
with the reason and the file each came from. Knowing which file was in force is
not the same as knowing what it hid.

**The caller's `permissions` block.** A called workflow can never hold more
permission than its caller. Asking for more fails the run at startup. Both were
verified, not assumed. `packages: write` is what a registry using the job token
needs. It does nothing for any other registry, so one block works everywhere.
