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
it. The SBOM comes from it. Nothing is built a second time.

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

## The source is scanned as well as the image

A multi-stage build throws most of its dependencies away. The blog builds with
`pnpm install` on a node image, then copies only the compiled site into an nginx
image. `node_modules` never reaches the runtime stage.

Trivy scanning that image reports `Number of language-specific files num=0`, and
it is right: there are no JavaScript packages in the artifact. The build-time
dependencies are still real, though. A compromised one can change what the build
emits, even though it does not ship.

So the workflow scans both, and the two see different things:

| | Reads | Finds |
| --- | --- | --- |
| `trivy fs` on the build context | lock files | build-time dependencies |
| `trivy image` on the layout | installed packages | what actually ships |

Neither is a subset of the other. `tests/fixtures/vulnerable` carries a
`node_modules` entry with no lock file, and the source scan reports nothing for
it. `tests/fixtures/source-deps` carries a lock file and no installed package,
and the image scan reports nothing for it. Both were measured.

The source scan runs on `context-path`, because that is exactly the input a
`docker build` is allowed to read. It uses the same severity floor, the same
`trivyignores`, and the same pinned configuration as the image scan.

It also runs **before** the build. A finding in the source costs seconds rather
than a full build, which is the same reasoning as `Resolve scan severities`.

Findings are reported by whatever id the advisory carries. For npm that is often
a GHSA rather than a CVE, so a `trivyignores` entry has to use the id the scan
printed.

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

The SBOM is CycloneDX, written by Trivy from the same layout it scanned.

Trivy writes it, not syft. Both were compared on the same layout, and their
package coverage is identical:

| | Trivy | syft |
| --- | --- | --- |
| `library` components | 17 | 17 |
| `operating-system` components | 1 (`alpine`) | 1 (`alpine`) |
| `file` components | 0 | 80 |
| Packages the other missed | none | none |
| CVEs found re-scanning the SBOM | 4 | 4 |

syft's extra 80 components are file entries such as `/etc/passwd` and
`/bin/busybox`. They are a file inventory, not packages, and they change nothing
about which vulnerabilities a reader can match. Everything else is the same
document, at the same CycloneDX version.

Using Trivy removes a tool, an install step and a version pin. It also means the
SBOM and the gate come from one binary and one pass over one layout.

SPDX is not used. syft's SPDX writer does not emit an operating-system package.
Trivy's SPDX reader needs that package to work out the distribution. Without it
Trivy reports nothing, which looks exactly like a clean scan.

This was measured. Against `ghcr.io/theadzik` images, `trivy sbom` found zero OS
packages where `trivy image` found twelve.

BuildKit's own `sbom` option is not set, for the same reason. BuildKit writes
SPDX only.

The document carries the vulnerabilities Trivy found, because the scanners are
enabled in the shared configuration. That is valid CycloneDX and it puts the
findings on record next to the inventory.

The step then checks its own output. If the SBOM has no `operating-system`
component, the build fails. Attesting such a document would be worse than
attesting nothing.

### Why the gate scans the image, not the SBOM

Trivy can scan an SBOM with `trivy sbom`. Doing that instead of scanning the
image would lose the secret gate, and gain nothing else.

Measured on one image, `alpine:3.19` carrying `lodash@4.17.11` and an RSA
private key:

| | `trivy image` | `trivy sbom` |
| --- | --- | --- |
| Vulnerabilities | 11 | 11 |
| Missed by the other | none | none |
| OS and language packages | both | both |
| EOL detected, `exit-on-eol` fires | yes | yes |
| Secrets found | 1 (`private-key`) | **0** |
| Time, warm database | 112 ms | 109 ms |

Secret scanning is not merely absent. `trivy sbom --scanners secret` is refused
by the CLI, which accepts only `vuln` and `license`. An SBOM records what is
installed, not what the files contain, so a private key baked into a layer
cannot appear in it.

There is no speed argument either. Writing the SBOM requires the same full pass
over the image, so scanning the SBOM afterwards adds work rather than replacing
it.

The gate would also become indirect. Scanning the image tests the artifact.
Scanning the SBOM tests a description of the artifact, and anything the
description missed is invisible to the gate.

Trivy reads the first platform in the index, which is why the SBOM covers one
platform. See below.

There are two SBOM attestations, both CycloneDX, both on the image digest:

- the image SBOM, whose `metadata.component` is the image reference,
- the source SBOM, whose `metadata.component` is the build context path.

`cosign verify-attestation --type cyclonedx` accepts both and returns both.
Measured, because a second attestation of the same predicate type could have
broken the verification step.

One option cannot live in the shared configuration file: `table-mode`. Trivy
rejects it unless the format is `table`, and this file is also used to write the
SBOM. It is passed on the scan command line instead.

## Signing and attestation

`actions/attest` produces signed statements *about* the image. `cosign sign`
produces a signature *on* the image.

Kyverno's `verifyImageSignatures` looks for the signature. An attestation is not
a substitute for it.

Both use the same keyless identity: this workflow's OIDC token.

### Every child is signed, not only the index

`cosign sign` runs with `--recursive`. On a multi-platform build that signs the
index **and** each child manifest.

This matters because the OCI specs do not say which one an artifact should hang
off. `subject` is defined identically for a manifest and for an index, and the
distribution spec's referrers API is an exact-digest lookup: a referrer attached
to a child is not returned when querying the index, and the reverse is also true.
So the choice is mechanical, not normative, and signing both ends removes it.

Measured against a local registry with cosign v3.1.2, one `--recursive` sign of a
two-platform index produced five signatures: the index, both platform manifests,
and both of buildkit's `unknown/unknown` attestation manifests.

The workflow then verifies its own signature, before any tag exists. An identity
the cluster would refuse fails the build instead of the rollout.

Verification covers the same five digests, for the reason the rest of this page
keeps repeating: a child that failed to sign would otherwise be found by the
cluster rather than by the build.

### The attestations are verified too

`cosign verify-attestation` reads both attestations back from the registry
before any tag is published. A malformed or missing one fails the build instead
of the rollout.

The predicate types have to match exactly, and the shorthands are not
interchangeable:

| Document | `actions/attest` writes | cosign shorthand |
| --- | --- | --- |
| CycloneDX | `https://cyclonedx.org/bom` | `cyclonedx` |
| SLSA provenance | `https://slsa.dev/provenance/v1` | `slsaprovenance1` |
| SPDX | `https://spdx.dev/Document/v<version>` | none that matches |

The SPDX row is the warning. `actions/attest` appends the document's own version
to the SPDX type, so cosign's `spdxjson` shorthand — which means
`https://spdx.dev/Document`, with no version — never matches. A raw URI works
where a shorthand does not. CycloneDX has no such problem: the action hardcodes
the type with no version, so `cyclonedx` matches.

Two properties of the SBOM writer are load-bearing here. `actions/attest`
recognises a CycloneDX document only if it carries `bomFormat`, `serialNumber`
and `specVersion`, and it rejects anything else outright. Trivy emits all three.
Checked, because switching the SBOM writer could have broken the attestation
step rather than the SBOM.

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

### Every platform is scanned

`trivy image --input` scans **one** platform of a multi-platform index. It picks
the **first** platform entry in the index, and nothing else changes that choice.

Building `linux/arm64,linux/amd64` instead of `linux/amd64,linux/arm64` makes
Trivy scan arm64. That is the whole selection rule, measured against a layout
built both ways.

So a single scan of a two-platform index checks one image and signs two. A layout
with a clean amd64 image and an arm64 image carrying `lodash@4.17.11` scanned
clean and exited 0. The same arm64 image alone reported four CVEs, one CRITICAL.

Rewriting the index is therefore not a workaround. It is the only input the
choice responds to.

### Why not `trivy image --platform`

Trivy does have a `--platform` flag. It does not work here, and it fails in the
worst possible way: it is accepted, it prints no warning, and it changes nothing.

Read the architecture Trivy reports back in
`.Metadata.ImageConfig.architecture`, rather than guessing from the findings:

| Target | Asked for | Actually scanned |
| --- | --- | --- |
| `--input <oci-layout>` | `linux/amd64` | amd64 |
| `--input <oci-layout>` | `linux/arm64` | **amd64** |
| registry image | `linux/amd64` | amd64 |
| registry image | `linux/arm64` | arm64 |

The flag resolves a platform out of a *remote* index. It does nothing for a
layout on disk.

A workflow built on it would therefore loop over platforms, print a clean result
for each, and scan amd64 every time. That is worse than not looping at all,
because the log would say the opposite of what happened.

Using it would mean pushing the index first and scanning it from the registry.
That breaks the rule this whole design exists for: nothing unscanned reaches the
registry.

Do not replace the rewrite loop with `--platform` without re-running that table.

### Why not run Trivy in a container of the target platform

QEMU is set up for cross-platform builds, so Trivy could run inside a
`linux/arm64` container and scan the layout from there. The hope is that Trivy
then picks the matching platform by itself.

It does not. Measured with `tonistiigi/binfmt` emulation registered:

| Trivy runs in | `uname -m` inside | Platform scanned | Time |
| --- | --- | --- | --- |
| `linux/amd64` container | `x86_64` | amd64 | 18s |
| `linux/arm64` container | `aarch64` | **amd64** | 36s |

The container really was arm64. Trivy still read the first entry in the index.
Its choice does not depend on the architecture it runs on.

The attempt also costs twice the time on a fixture with one package, before any
real image is scanned, because the scanner itself is emulated.

The workflow therefore scans each platform on its own. For each child manifest in
the index it rewrites `index.json` to name only that manifest, runs Trivy against
the layout, and moves on. The blobs never move. Only the index changes, and the
original is restored when the loop ends, because the push reads it back.

Findings on any platform fail the build. The loop does not stop at the first one,
so a run reports every affected platform instead of one at a time.

This is why the scan uses the Trivy CLI rather than `aquasecurity/trivy-action`.
A `uses:` step cannot run in a loop. The action's `trivyignores` handling is
replaced by the resolution in `Write the Trivy configuration`, which applies the
same rules and prints the file it settled on.

Three layout shapes reach the loop, and all three are handled:

| Build | Root of `index.json` | Treated as |
| --- | --- | --- |
| `push: false`, no provenance | image manifest | one platform |
| One platform with provenance | index, one child | one platform |
| Several platforms | index, several children | one scan per child |

Children whose platform is `unknown/unknown` are skipped. Those are buildkit's
attestation manifests, not images.

### QEMU

A cross-platform `RUN` needs emulation. Without it the build fails with
`exec format error`.

`docker/setup-qemu-action` runs whenever `platforms` is anything other than
`linux/amd64`. A native-only build skips it, because there it costs time and
does nothing.

### The SBOM still covers one platform

Trivy reads the first platform in the index, so the attested SBOM describes that
platform only. Scanning does cover them all, because the loop rewrites the index;
the SBOM step runs after the loop has restored it.

The platforms really do differ, so this is a gap rather than a formality. An
`alpine` image built for amd64 and arm64 from one Dockerfile, with no
per-architecture logic at all, produced two SBOMs with the same 27 packages at
the same versions — and 26 of the 27 purls different:

```text
amd64: pkg:apk/alpine/libcrypto3@3.5.7-r0?arch=x86_64&distro=3.24.1
arm64: pkg:apk/alpine/libcrypto3@3.5.7-r0?arch=aarch64&distro=3.24.1
```

purl is what a consumer matches on, so an arm64 deployment does not match the
attested amd64 document. A Dockerfile that varies by `TARGETARCH` can differ far
more than that: the multi-platform test fixture carries a vulnerable package on
one architecture and not the other.

An index built for several platforms carries one signed SBOM, and that SBOM does
not describe the other images under it. The build warns when this applies.

This is the remaining gap in multi-platform coverage. The scan no longer has one.

## Pinned tools

`cosign` is pinned with the installer's `cosign-release` input. Version 3.x
writes the signature as an OCI referrer with the predicate
`https://sigstore.dev/cosign/sign/v1`. Version 2.x wrote a
`sha256-<digest>.sig` tag instead. Which one an admission controller accepts
should not change because an installer default changed. Dependabot does not
track this input.

`trivy` is installed by `aquasecurity/setup-trivy` with an explicit `version`,
because the scan is a CLI loop rather than an action. Raising it changes what the
gate catches, so raise it deliberately.

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
