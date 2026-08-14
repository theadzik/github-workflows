# The fixture for every non-amd64 build: `linux/arm64` alone, and the
# multi-platform index.
#
# Deliberately free of any RUN. build-and-push.yaml sets up buildx but not QEMU,
# so a cross-platform build there can copy layers but cannot execute them. A
# fixture with a RUN would fail on the missing emulator and say nothing about
# the workflow.
FROM alpine:latest

COPY marker.txt /fixture/marker.txt
