# Proves `build-args` reaches the build, rather than proving the input exists.
#
# The RUN below is the assertion: an arg that never arrives leaves the variable
# empty, the test fails, and the build fails with it. Two args, because the
# input is newline-separated and a single one would not catch a splitter that
# only ever keeps the first line.
FROM alpine:latest

ARG EXPECTED
ARG SECOND_ARG

RUN test "${EXPECTED}" = "passed-through" \
  && test "${SECOND_ARG}" = "also-passed" \
  && echo "build-args arrived: ${EXPECTED} / ${SECOND_ARG}"
