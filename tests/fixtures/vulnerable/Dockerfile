# The fixture every scan scenario builds: an image Trivy is guaranteed to have
# something to say about.
#
# The finding comes from an installed package rather than from an old base
# image. A pinned old distro would age out of Trivy's support window and quietly
# start reporting nothing - the failure mode that looks exactly like a clean
# scan. A fixed vulnerability in a published npm release never stops being
# reported, and `lodash@4.17.11` carries CVE-2019-10744 (CRITICAL, fixed in
# 4.17.12), which survives `ignore-unfixed: true`.
#
# It has to be a `node_modules` entry rather than a lockfile. Scanning an image,
# Trivy runs its node-pkg analyzer over installed `package.json` files; the
# lockfile analyzers only run against a filesystem, so a `package-lock.json`
# baked into an image is reported as "language-specific files: 0" - which reads
# exactly like a clean scan.
#
# The base contributes nothing of its own. `HIGH,CRITICAL` is a floor no input
# can narrow, so every fixable finding in this image has to be listed in the
# ignore files beside it; a hardened base keeps that list at the lodash
# advisories instead of growing it every time the distribution lags a fix.
FROM dhi.io/alpine-base:3.24-dev

COPY node_modules /app/node_modules
