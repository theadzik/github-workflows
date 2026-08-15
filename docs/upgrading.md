# Upgrading

Version tags mark the caller contract. A major tag means a caller has to change
something, not only move a pin.

## v3 to v4

v4 scans the build context as well as the image.

Move the pin, then read the one thing that can fail a build that used to pass.

```yaml
uses: theadzik/github-workflows/.github/workflows/build-and-push.yaml@<commit-sha> # v4.0.0
```

No inputs change. No cluster change is needed.

### 1. Your build context is now scanned

A multi-stage build usually throws its dependencies away. If you build with
`pnpm install` and then copy only the compiled output into a runtime image, the
image holds no JavaScript packages at all, and the image scan correctly reports
none.

Those dependencies still shape what the build produces. v4 therefore runs
`trivy fs` over `context-path` before the build, using the same HIGH/CRITICAL
floor, the same `extra-scan-severity`, and the same `trivyignores`.

The shared configuration enables the `secret` scanner as well as `vuln`, so the
context is checked for committed credentials at the same time. A private key in
the build context now fails the build, just as one inside a layer already did.

**A repository with a fixable HIGH or CRITICAL in a lock file, or a secret in
its build context, will fail on its next build.** Those findings were always
there; nothing was scanning for them.

Two ways forward, and they are the usual two:

- Upgrade the dependency.
- Record the acceptance in the file named by `trivyignores`.

Use the id the scan prints. For npm advisories that is often a GHSA rather than
a CVE:

```text
GHSA-5c6j-r48x-rmvq
```

An audit tool's own ignore list does **not** apply here. A `pnpm-workspace.yaml`
carrying `auditConfig.ignoreGhsas` is read by `pnpm audit` and by nothing else.
The workflow only reads the file you name in `trivyignores`.

### 2. There are now two SBOM attestations

Both are CycloneDX, both attached to the image digest. Tell them apart by
`metadata.component`: the image reference for the image SBOM, the context path
for the source one.

`cosign verify-attestation --type cyclonedx` returns both and still exits 0, so
any existing verification keeps working.

### 3. The scan runs before the build

A source finding now fails the run in seconds instead of after a full build. The
Trivy configuration and install moved earlier in the job for the same reason.

## v2 to v3

v3 makes the scan impossible to weaken from the caller side. Three things that
were caller decisions in v2 are now fixed by the workflow.

Move the pin, then work through the checklist below.

```yaml
uses: theadzik/github-workflows/.github/workflows/build-and-push.yaml@<commit-sha> # v3.0.0
```

No cluster change is needed. The Kyverno policy matches the workflow path and
any ref after `@`, so a v3 signature is accepted with no policy edit.

### 1. Replace `scan-severity` with `extra-scan-severity`

`scan-severity` is **removed**. A caller that still passes it fails at startup
with an invalid-input error, before anything builds.

The replacement is not a rename. The meaning is inverted:

| | v2 `scan-severity` | v3 `extra-scan-severity` |
| --- | --- | --- |
| What it is | The whole list of severities that fail the build | Severities added to a fixed HIGH,CRITICAL floor |
| Default | `HIGH,CRITICAL` | *(none)* |
| Accepts | Any severity | `UNKNOWN`, `LOW`, `MEDIUM` only |
| Can it narrow the gate? | Yes | No |

Translate what you had:

```yaml
# v2                          # v3
scan-severity: "HIGH,CRITICAL"   → delete the line; this is the default now
scan-severity: "CRITICAL"        → delete the line; HIGH is enforced as well
scan-severity: "MEDIUM,HIGH,CRITICAL" → extra-scan-severity: "MEDIUM"
```

Naming `HIGH` or `CRITICAL` in `extra-scan-severity` fails the run. This is
deliberate. A caller who writes them is usually trying to make the gate smaller,
and needs to be told that they cannot.

**A build that passed under v2 can fail under v3.** If you narrowed the list to
`CRITICAL`, fixable HIGH findings were invisible before and now fail the build.
Fix them, or record them with `trivyignores`.

### 2. `scan: false` no longer applies to a publishing run

The gate is now `push || scan`. Read it as one rule:

> You can scan without publishing. You cannot publish without scanning.

`scan` is still a valid input. It still works on a run with `push: false`. It now
does nothing on a run that publishes.

Callers that pass `scan: true` alongside the default `push: true` can delete the
line. It reads as though it does something, and it does not.

If you used `scan: false` with `push: true` anywhere, that build was publishing
unscanned images. Under v3 it is scanned, and it may fail. That is the point of
the change.

### 3. Trivy configuration files in your repository stop being read

v2 ran Trivy in your checked-out repository with no configuration of its own.
Trivy discovers three files in the working directory, so your repository could
change the scan that guards your own publish.

v3 writes all three itself and names them. Files in your repository are no longer
found:

| File in your repository | v2 | v3 |
| --- | --- | --- |
| `trivy.yaml` | Reconfigured the scan | Ignored |
| `.trivyignore` | Suppressed the findings it listed | Ignored |
| `trivy-secret.yaml` | Could disable secret rules | Ignored |

`.trivyignore.yaml` was never discovered automatically, in either version. It
works only when something names it.

**Check for these files before you upgrade.** A repository that relied on one for
suppression will fail its next build.

```shell
ls trivy.yaml .trivyignore trivy-secret.yaml 2>/dev/null
```

If you find one, move the acceptances into a file and name it with the
`trivyignores` input. That input still overrides the workflow's default, because
an acceptance the workflow was told about is a decision on record.

Prefer the YAML form. It carries `expired_at`, so an acceptance stops applying on
a date, and `statement`, so the reason sits next to what it excuses.

```yaml
trivyignores: "apps/my-app/.trivyignore.yaml"
```

### 4. An end-of-life base image now fails the build

v3 sets `exit-on-eol`. A distribution that has stopped issuing security updates
now fails the run.

This is a new failure for an image that passed under v2. It is not a false
alarm. An EOL base scans clean and exits 0 because Trivy has nothing to
*report*, not because there is nothing to find.

Move to a base image that still receives patches.

### 5. Multi-platform builds are scanned in full, and can now `RUN`

Two changes land together here.

**Every platform is scanned.** v2 scanned one platform of a multi-platform index
and signed all of them. `trivy image --input` picks one child manifest and has no
flag to choose another. v3 scans each platform separately.

If you build for more than one platform, v3 checks images that v2 never looked
at. A build that passed under v2 can fail under v3 because of a finding on a
platform that was never scanned before. That finding was always there.

**A cross-platform `RUN` works.** v3 sets up QEMU whenever `platforms` is
anything other than `linux/amd64`. Under v2 a `RUN` on a foreign architecture
failed with `exec format error`, so cross builds had to be limited to `COPY`.

No input changes for either. The behaviour follows `platforms`.

One gap remains: the attested SBOM still describes the first platform in the
list. The build prints a warning when that applies.

### 6. The SBOM is written by Trivy, not syft

v2 installed syft to write the CycloneDX SBOM. v3 uses Trivy, which already runs
for the scan.

No input changes, and the SBOM is still CycloneDX with an `operating-system`
component, so anything reading it keeps working. Two differences if you compare
documents byte for byte:

- syft also emitted about 80 `file` components, such as `/etc/passwd`. Trivy does
  not. Package coverage is identical, and re-scanning either document reports the
  same CVEs.
- Trivy embeds the vulnerabilities it found, because the scanners are enabled.

### 7. Every platform of a multi-platform image is signed

v2 signed the index only. v3 adds `cosign sign --recursive`, which signs the
index and every child manifest, and then verifies all of them before publishing
any tag.

No input changes. A multi-platform build now produces more signatures than it
did, and a single-platform build produces two or three instead of one, because
buildkit's provenance manifests are children too.

### 8. The attestations are verified before any tag is published

v3 adds a `Verify attestations` step. It reads the SBOM and provenance
attestations back from the registry with `cosign verify-attestation`, and fails
the build if either is missing or does not verify.

No input changes. A build that produced a broken attestation used to publish
tags anyway; it now stops.

### 9. Things that did not change

- Every other input keeps its name, type and default.
- The outputs are unchanged.
- The secrets are unchanged.
- The required `permissions` block is unchanged.
- Secret scanning already failed the build in v2. v3 writes the scanner list down
  so that a future Trivy release cannot drop it quietly, but the gate is the
  same.

### Checklist

1. Remove `scan-severity`. Add `extra-scan-severity` only if you were widening
   the list, and only with `UNKNOWN`, `LOW` or `MEDIUM`.
2. Remove `scan: true` if `push` is true. Keep `scan: false` only where `push` is
   false.
3. Look for `trivy.yaml`, `.trivyignore` and `trivy-secret.yaml` in the
   repository root. Move any acceptance into a file named by `trivyignores`.
4. Check that the base image is not end-of-life.
5. Move the pin to the v3 commit SHA, with the tag in a trailing comment.
6. Run the build on a pull request first, with `push: false`. The scan runs
   there, so a new finding appears before anything can be published.

## v1 to v2

v2 builds to an OCI layout on disk, scans it there, and pushes by digest with
`oras`. v1 built straight into the registry and scanned the pushed digest, so an
image existed in the registry before it was scanned.

v1 assumed Docker Hub throughout. v2 takes the registry, the credentials and the
namespace from the caller.

Callers must:

1. Add `packages: write` to the `permissions` block.
2. Pass `registry` and `registry-username`.
3. Map `REGISTRY_PASSWORD` instead of inheriting `DOCKERHUB_TOKEN`.
4. Move the namespace out of the username and into `image-name`.
