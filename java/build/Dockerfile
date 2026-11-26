# syntax=docker/dockerfile:1-labs

###################################
# Target: build
###################################
FROM base AS build

WORKDIR /src

# Copy project files and restore as distinct layer.
COPY --link --parents --exclude=**/settings.xml **/pom.xml **/.mvn ./

ARG BUILDCHARTS_SRC
ARG MAVEN_DEPENDENCY_PLUGIN_VERSION=3.11.0

# Keep cross-process locks in the shared cache mount.
ENV MAVEN_ARGS="--batch-mode --no-transfer-progress -Dmaven.repo.local=/root/.m2/repository -Daether.enhancedLocalRepository.split=true -Daether.syncContext.named.factory=file-lock -Daether.syncContext.named.nameMapper=file-gav -Daether.syncContext.named.basedir.locksDir=cached/.locks"

# Use a fixed revision so the cache warmup can be reused across build versions.
RUN --mount=type=secret,id=settings.xml,target=/root/.m2/settings.xml \
    --mount=type=cache,id=maven,target=/root/.m2/repository/cached,sharing=shared \
    mvn --file "/src/$BUILDCHARTS_SRC" -Drevision=0.0.0 \
        "org.apache.maven.plugins:maven-dependency-plugin:$MAVEN_DEPENDENCY_PLUGIN_VERSION:go-offline"

# Copy code.
COPY --link --exclude=**/.mvn/settings.xml . .

ARG VERSION
ARG COMMIT=
ARG AZURE_PIPELINES_SOURCE_ROOT=

# Execute build.
RUN --mount=type=secret,id=settings.xml,target=/root/.m2/settings.xml \
    --mount=type=cache,id=maven,target=/root/.m2/repository/cached,sharing=shared <<'SH'
set -eu
compile() {
    mvn --file "/src/$BUILDCHARTS_SRC" "$@" \
        -Drevision="$VERSION" \
        -Dcommit="$COMMIT" \
        clean test-compile
}

if [ -z "$AZURE_PIPELINES_SOURCE_ROOT" ]; then
    compile
    exit 0
fi

# Forward Maven diagnostics and preserve the Maven exit code through the output filter.
status_file="$(mktemp)"
trap 'rm -f "$status_file"' EXIT
{
    status=0
    compile -Dstyle.color=never 2>&1 || status=$?
    printf '%s\n' "$status" > "$status_file"
} | awk '
    function escape(value) {
        gsub(/%/, "%AZP25", value)
        gsub(/\r/, "%0D", value)
        gsub(/\n/, "%0A", value)
        gsub(/;/, "%3B", value)
        gsub(/\]/, "%5D", value)
        return value
    }

    BEGIN {
        root = ENVIRON["AZURE_PIPELINES_SOURCE_ROOT"]
        sub(/\/+$/, "", root)
    }

    {
        print
        diagnostic = $0
        if (sub(/^\[(ERROR|WARNING)\][[:blank:]]+\/src\//, "", diagnostic) &&
            match(diagnostic, /\.java:\[[0-9]+,[[:blank:]]*[0-9]+\]/)) {
            type = $1 == "[ERROR]" ? "error" : "warning"
            path = root "/" substr(diagnostic, 1, RSTART + length(".java") - 1)
            position = substr(diagnostic, RSTART + length(".java:["), RLENGTH - length(".java:[]"))
            split(position, location, /,[[:blank:]]*/)
            message = substr(diagnostic, RSTART + RLENGTH)
            sub(/^[[:blank:]]+/, "", message)
            issue = sprintf("##" "vso[task.logissue type=%s;sourcepath=%s;linenumber=%s;columnnumber=%s;]%s",
                type, escape(path), location[1], location[2], escape(message))
            if (!seen[issue]++) {
                print issue
            }
        }
        fflush()
    }
'
exit "$(cat "$status_file")"
SH
