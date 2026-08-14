# The fixture every "this must pass" scenario builds.
#
# A real OS layer rather than `scratch`, because two steps depend on one: Trivy
# needs a package database to scan, and the SBOM step fails the build outright
# when syft finds no operating-system component.
#
# The tag floats deliberately. A digest pin would accumulate fixable
# HIGH/CRITICAL findings over time and turn every unrelated pull request red;
# `alpine:latest` is patched upstream faster than this repository changes.
FROM alpine:latest

RUN echo "clean fixture" > /fixture.txt
