# Tests

Fixtures and scenarios for
[`test-build-and-push.yaml`](../.github/workflows/test-build-and-push.yaml).

That workflow calls `build-and-push.yaml` the way a caller does, once per
scenario. `uses: ./` resolves to the copy on the pull request's merge ref, so
what runs is the change under review, not what is already on `main`.

The workflow file carries short comments only. The reasons are here.

## Fixtures

| Path | Used for | Why it looks like this |
| --- | --- | --- |
| `fixtures/clean` | Every scenario that must pass | A real OS layer. Trivy needs a package database, and the SBOM step fails the build when syft reports no operating-system component. |
| `fixtures/build-args` | `build-args` | The `RUN` is the assertion. Args that never arrive leave the variables empty and fail the build. There are two, because the input is newline-separated. |
| `fixtures/cross` | `linux/arm64`, and the multi-platform index | The `RUN` is the assertion. It fails with `exec format error` without QEMU, so a green run proves emulation works, and it checks that `uname -m` matches the requested `TARGETARCH`. |
| `fixtures/vulnerable` | Every scan scenario | Carries `lodash@4.17.11` in `node_modules`. CVE-2019-10744 is CRITICAL and fixed in 4.17.12, so it survives `ignore-unfixed`. |
| `trivyignore/*` | `trivyignores`, both forms | One YAML file with a reason and an expiry. Two plain files, where only the second suppresses anything. |

Base images float on `alpine:latest` on purpose. A digest pin would collect
fixable HIGH and CRITICAL findings over time, and would turn unrelated pull
requests red.

The finding comes from an installed package, not from an old base image. The
reason is the same one in reverse. A pinned old distribution ages out of Trivy's
support window and then quietly reports nothing. That looks exactly like a clean
scan. A fixed vulnerability in a published npm release never stops being
reported.

It has to be a `node_modules` entry, not a lockfile. When Trivy scans an image it
runs its node-pkg analyzer over installed `package.json` files. The lockfile
analyzers run only against a filesystem. A `package-lock.json` baked into an
image therefore scans as `language-specific files: 0`, which cannot be told apart
from clean.

## What each scenario covers

| Scenario | Path through the workflow |
| --- | --- |
| `defaults` | Every optional input defaulted, and the whole publish → sign → attest → verify → tag chain. |
| `build-only` | `push: false`, plus `extra-scan-severity` and `timeout-minutes`. |
| `every-optional-input` | `build-args`, several `tags`, `git-ref`, `fetch-depth: 0`, `extra-registry` login. |
| `arm64` | One platform that is not the default. This is the case syft gets wrong when left to pick. |
| `multi-platform` | The SBOM coverage warning, an index push, QEMU, and the per-platform scan loop running more than once. |
| `scan-disabled` | `scan: false` against the vulnerable fixture. Built, never pushed, because `scan: false` applies only when `push` is `false`. |
| `trivyignore-yaml` / `trivyignore-plain` | Both accepted forms of `trivyignores`. |
| `published` | The `digest` and `image-ref` outputs, the signature, the referrers and the tag. Checked from outside, against what `defaults` left in the registry. |

Two scenarios are shaped by details worth stating.

**`build-only` passes `extra-scan-severity: "medium, low"`.** The value is
lowercase, spaced and two items long on purpose. Together those test the letter
folding, the whitespace stripping and the splitter in `Resolve scan severities`.
A single bare `MEDIUM` would test none of them. The scenario cannot prove that
MEDIUM findings now fail a build, because nothing that fails is testable here. It
proves only that a wider list is accepted and used. The clean fixture has no
fixable finding at any severity today, so this cannot make the scenario red.

**The `trivyignore-*` scenarios pass no severity input.** They used to narrow the
scan to CRITICAL, so that the fixture's one CRITICAL was all Trivy considered.
HIGH is a floor now, so narrowing is no longer possible. See below.

## What is not covered

**Anything that is supposed to fail.** A job that calls a reusable workflow may
not set `continue-on-error`; the schema rejects it. A scenario meant to fail can
therefore only report failure, and no green run proves it.

Every guard in `build-and-push.yaml` is untested for this reason:

- the scan failing the build on a fixable HIGH or CRITICAL,
- `Resolve scan severities` rejecting `HIGH`, or a value that is not a severity,
- the SBOM step refusing a document with no operating-system component,
- the push step catching a registry that stored a different digest,
- `Publish tags` refusing to run when `docker/metadata-action` produced nothing.

**The default `context-path`.** Building `.` would need a `Dockerfile` at the
root of a repository that holds no application. Every scenario passes the input
instead.

## Keeping the vulnerable fixture honest

The `trivyignore-*` scenarios can go wrong in both directions.

**They can pass for the wrong reason.** They pass when the scan finds nothing to
report. That is also what a fixture that stopped being vulnerable looks like. If
`fixtures/vulnerable` ages out, both scenarios keep passing and stop meaning
anything.

**They can fail for a reason that is not a bug.** `HIGH,CRITICAL` is a floor that
no input can narrow. The ignore files must therefore list *every* fixable HIGH
and CRITICAL in the fixture, not only the CVE the fixture exists for. A new
lodash advisory turns both scenarios red until it is added.

`accepted.trivyignore` and `accepted.trivyignore.yaml` must stay in step with
each other.

Check both directions with two commands:

```shell
docker build -t vuln tests/fixtures/vulnerable

# Must exit 1. The fixture is still vulnerable.
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 vuln

# Must exit 0. The ignore file still covers all of it. Anything the first
# command lists and this file does not is what turned the scenarios red.
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
  --ignorefile tests/trivyignore/accepted.trivyignore.yaml vuln
```

## Who can run it

The called workflow needs `id-token: write` for keyless signing, even on a run
that only builds. A called workflow cannot hold more permission than its caller.

A fork's pull request is capped at read-only, so it cannot grant that scope.
Every scenario is skipped there and the run tests nothing. It says so in an
annotation rather than reporting a quiet green. Dispatch the workflow by hand
against the branch to test such a change.

Dependabot's pull requests do run. Its runs are treated like a fork's — read-only
token, no repository secrets — but scopes named in a `permissions:` block are
still granted, and every scenario names all five. Nothing here reads a repository
secret; `secrets.GITHUB_TOKEN` is the job token and is always present.

Adding a scenario that needs a real repository secret would break this, and it
would break it only for dependabot.

## Cleanup

Publishing scenarios push to `ghcr.io/theadzik/github-workflows-test`. The
`cleanup` job deletes what the run created: every version carrying a tag with
this run's id, plus the signature and attestations hanging off each one. Those
are separate untagged versions, and they are what would otherwise pile up.

It runs at the end of the run, not at pull request close. The images are junk as
soon as `published` has looked at them, and a manually dispatched run has no
close event to hang cleanup on. It also runs after a failed scenario, because a
build that failed after pushing still left tags behind.

Two things it deliberately keeps:

- **`latest`, and its referrers.** Keeping one version keeps the package alive. A
  package deleted down to nothing comes back on the next push without the Actions
  access grant below. Cleanup would then fail until someone granted it again by
  hand.
- **Anything belonging to another run.** Scoping by run id keeps a concurrent run
  on another branch out of scope. That holds even when an identical build
  resolved to a digest both runs tagged. Untagged orphans from a run that died
  mid-way are swept only once they are more than six hours old.

`GITHUB_TOKEN` can delete package versions only where the package grants this
repository the Admin role, under **Package settings → Manage Actions access**. A
package created by a workflow push normally has it already. If cleanup fails,
check that setting first — the job says so in its error.
