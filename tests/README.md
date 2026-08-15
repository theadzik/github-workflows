# Test fixtures

Build contexts and ignore files for
[`test-build-and-push.yaml`](../.github/workflows/test-build-and-push.yaml), which calls
`build-and-push.yaml` the way a caller does — `uses: ./` resolves to the copy on the pull
request's own merge ref, so what runs is the change under review.

## Fixtures

| Path | Used for | Why it looks like this |
| --- | --- | --- |
| `fixtures/clean` | Every scenario that must pass | A real OS layer: Trivy needs a package database, and the SBOM step fails the build when syft reports no operating-system component. |
| `fixtures/build-args` | `build-args` | The `RUN` is the assertion — args that never arrive leave the variables empty and fail the build. Two of them, because the input is newline-separated. |
| `fixtures/cross` | `linux/arm64`, and the multi-platform index | No `RUN`: the workflow sets up buildx but not QEMU, so a cross build can copy layers but not execute them. |
| `fixtures/vulnerable` | Every scan scenario | Carries `lodash@4.17.11` in `node_modules` — CVE-2019-10744, CRITICAL, fixed in 4.17.12, so it survives `ignore-unfixed`. |
| `trivyignore/*` | `trivyignores`, both forms | One YAML file with a reason and an expiry; two plain files where only the second suppresses anything. |

Base images float on `alpine:latest` deliberately. A digest pin would accumulate fixable
HIGH/CRITICAL findings over time and turn unrelated pull requests red.

The finding comes from an installed package rather than an old base image for the same
reason in reverse: a pinned old distro ages out of Trivy's support window and quietly
starts reporting nothing — the failure mode that looks exactly like a clean scan. A fixed
vulnerability in a published npm release never stops being reported.

It has to be a `node_modules` entry rather than a lockfile. Scanning an image, Trivy runs
its node-pkg analyzer over installed `package.json` files; the lockfile analyzers only run
against a filesystem, so a `package-lock.json` baked into an image scans as
`language-specific files: 0` — indistinguishable from clean. Nothing here would notice; see
what is not covered, below.

## What each scenario covers

| Scenario | Path through the workflow |
| --- | --- |
| `defaults` | Every optional input defaulted, and the whole publish → sign → attest → verify → tag chain. |
| `build-only` | `push: false`, plus `extra-scan-severity` and `timeout-minutes`. |
| `every-optional-input` | `build-args`, several `tags`, `git-ref`, `fetch-depth: 0`, `extra-registry` login. |
| `arm64` | A single platform that is not the default — the case syft gets wrong when left to pick. |
| `multi-platform` | The coverage warning, an index push, syft describing the first platform. |
| `scan-disabled` | `scan: false`, against the vulnerable fixture. Built, never pushed — `scan: false` is only honoured when `push` is `false`. |
| `trivyignore-yaml` / `trivyignore-plain` | Both accepted forms of `trivyignores`. |
| `published` | The workflow's `digest` and `image-ref` outputs, the signature, the referrers and the tag — checked against what the `defaults` call left in the registry. |

## What is not covered

**Anything that is supposed to fail.** A job that calls a reusable workflow may not set
`continue-on-error` — the workflow schema rejects it — so a scenario meant to fail can only
ever report failure, and there is no green run that proves it. Every guard in
`build-and-push.yaml` is therefore untested: the scan failing the build on a fixable
HIGH/CRITICAL, `Resolve scan severities` rejecting `HIGH` or a value that is not a severity,
the SBOM step refusing a document with no operating-system component, the push step catching
a registry that stored a different digest, and `Publish tags` refusing to run when
`docker/metadata-action` produced nothing.

One consequence is worth naming. The `trivyignore-*` scenarios pass when the scan finds
nothing to report — which is also what a fixture that has quietly stopped being vulnerable
looks like. If `fixtures/vulnerable` ages out, those two scenarios keep passing and stop
meaning anything.

They fail the opposite way too, and that one is new. `HIGH,CRITICAL` is a floor no input can
narrow, so the ignore files have to list *every* fixable HIGH and CRITICAL in the fixture,
not just the CVE the fixture exists for. A new lodash advisory turns both scenarios red until
it is added to `accepted.trivyignore` and `accepted.trivyignore.yaml`, which are meant to
stay in step with each other.

Both directions are one command:

```shell
docker build -t vuln tests/fixtures/vulnerable

# Must exit 1 — the fixture is still vulnerable.
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 vuln

# Must exit 0 — the ignore file still covers all of it. Anything listed here
# and missing from the file above is what turned the scenarios red.
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
  --ignorefile tests/trivyignore/accepted.trivyignore.yaml vuln
```

**The default `context-path`.** Building `.` would mean a `Dockerfile` at the root of a
repository that contains no application, so every scenario passes the input explicitly.

## Who can run it

The called workflow needs `id-token: write` for keyless signing even when it is only
building, and a called workflow cannot hold more permission than its caller. A fork's pull
request is capped at read-only and cannot grant it, so every scenario is skipped there and
the run tests nothing at all — it says so in an annotation rather than reporting a quiet
green. Dispatch the workflow by hand against the branch to test one.

Dependabot's pull requests do run. Its workflow runs are treated like a fork's — read-only
token, no repository secrets — but scopes named explicitly in a `permissions:` block are
still granted, and every scenario names all five. Nothing here reads a repository secret;
`secrets.GITHUB_TOKEN` is the job token and is always present. Adding a scenario that needs
one — a real second registry credential, say — is what would break this, and it would break
it only for dependabot.

## Cleanup

Publishing scenarios push to `ghcr.io/theadzik/github-workflows-test`. The `cleanup` job
deletes what the run created — every version carrying a tag with this run's id, plus the
signature and attestations hanging off each one, which are separate untagged versions and
are the things that would otherwise pile up. It runs at the end of the run rather than at
pull request close: the images are junk the moment `published` has looked at them, and a
manually dispatched run has no close event to hang cleanup on. It runs after a failed
scenario too, since a build that failed after pushing still left tags behind.

Two things it deliberately does not delete:

- **`latest`, and its referrers.** Keeping one version keeps the package alive. A package
  deleted down to nothing comes back on the next push without the Actions access grant
  below, and cleanup would then fail until someone re-granted it by hand.
- **Anything belonging to another run.** Scoping by run id means a concurrent run on
  another branch is never in scope, including when an identical build resolved to a digest
  both runs tagged. Untagged orphans left by a run that died mid-flight are swept only once
  they are more than six hours old.

`GITHUB_TOKEN` can delete package versions only where the package grants this repository
the Admin role under **Package settings → Manage Actions access**. A package created by a
workflow push normally has it already; if cleanup fails, that setting is what to check —
the job says so in its error.
