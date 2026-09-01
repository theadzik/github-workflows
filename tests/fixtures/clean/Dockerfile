# The fixture every "this must pass" scenario builds.
#
# A real OS layer rather than `scratch`, because two steps depend on one: Trivy
# needs a package database to scan, and the SBOM step fails the build outright
# when the document has no operating-system component.
#
# The base is a Docker Hardened Image rather than `alpine:latest`. The float was
# supposed to keep this fixture clean, and it stopped: Docker Hub's
# `alpine:latest` shipped a fixable HIGH (CVE-2026-14456, libcrypto3/libssl3)
# and turned every unrelated pull request red until Alpine published the fix.
# `dhi.io/alpine-base:3.24-dev` is a mutable tag too, rebuilt with patched
# packages, so nothing here is pinned to a distribution that can age out - but
# it is rebuilt ahead of the advisory rather than behind it.
#
# `-dev` rather than the runtime variant: the RUN below needs a shell and a
# writable root, and the runtime image runs as nonroot.
FROM dhi.io/alpine-base:3.24-dev

RUN echo "clean fixture" > /fixture.txt
