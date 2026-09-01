# The fixture for every non-amd64 build: `linux/arm64` alone, and the
# multi-platform index.
#
# The RUN is the assertion. Without the QEMU step it fails with "exec format
# error" on a cross build, so a green run proves emulation works. It also checks
# that the layer it lands in really is the architecture buildx was asked for.
FROM dhi.io/alpine-base:3.24-dev

ARG TARGETARCH

COPY marker.txt /fixture/marker.txt

RUN machine="$(uname -m)"; \
    case "${TARGETARCH}:${machine}" in \
      amd64:x86_64 | arm64:aarch64) echo "${TARGETARCH} ${machine}" > /fixture/arch.txt ;; \
      *) echo "wrong arch: TARGETARCH=${TARGETARCH} uname -m=${machine}"; exit 1 ;; \
    esac
