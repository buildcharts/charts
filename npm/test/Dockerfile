###################################
# Target: pre-test
###################################
FROM base AS pre-test

# Configure Testcontainers to talk to dind daemon used by the TestcontainersDinD@v1 plugin.
ARG DOCKER_HOST=tcp://host.docker.internal:2375
ARG TESTCONTAINERS_HOST_OVERRIDE=

WORKDIR /src

COPY --link --from=build /usr/local/ /usr/local/
COPY --link --from=build /src/ /src/

ARG BUILDCHARTS_SRC
ARG NPM_SCRIPT=test

# Execute tests and collect JUnit + coverage outputs.
RUN set -eu; \
    export BUILDCHARTS_RESULTS_NAME="$(basename "$BUILDCHARTS_SRC")"; \
    export BUILDCHARTS_RESULTS_DIR="/output/${BUILDCHARTS_RESULTS_NAME}_$(date +%Y%m%d-%H%M%S)"; \
    mkdir -p "$BUILDCHARTS_RESULTS_DIR"; \
    pnpm run "$NPM_SCRIPT" || printf 'Test command exited with code %s; exporting available reports.\n' "$?"

###################################
# Target: test
###################################
FROM scratch AS test
COPY --from=pre-test /output /
