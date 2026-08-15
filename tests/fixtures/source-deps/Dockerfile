# The fixture for the source scan. The lock file beside this Dockerfile is what
# `trivy fs` reads; nothing vulnerable is installed into the image, so a finding
# here can only have come from the source scan and not from the image scan.
FROM alpine:latest

COPY package-lock.json /fixture/package-lock.json
