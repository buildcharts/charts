###################################
# Target: pre-publish
###################################
FROM build AS pre-publish

ARG BUILDCHARTS_SRC
ARG CONFIGURATION=Release
ARG DEBUG_SYMBOLS=false
ARG DEBUG_TYPE=none
ARG ENABLE_COMPRESSION_IN_SINGLE_FILE=true
ARG PUBLISH_AOT=false
ARG PUBLISH_IIS_ASSETS=false
ARG PUBLISH_NAME
ARG PUBLISH_SINGLE_FILE=true
ARG PUBLISH_TRIMMED=true
ARG RUNTIME=win-x64
ARG SELF_CONTAINED=true
ARG TRIM_MODE=partial
ARG VERSION

# Restore with secrets.
RUN --mount=type=secret,id=VSS_NUGET_ACCESSTOKEN,env=VSS_NUGET_ACCESSTOKEN \
    --mount=type=secret,id=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS,env=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
      dotnet restore "/src/$BUILDCHARTS_SRC" --runtime "$RUNTIME"

# Execute publish.
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
    dotnet publish "/src/$BUILDCHARTS_SRC" \
        --no-restore \
        --configuration "$CONFIGURATION" \
        --output /app/publish \
        --runtime "$RUNTIME" \
        --self-contained "$SELF_CONTAINED" \
        /p:DebugSymbols="$DEBUG_SYMBOLS" \
        /p:DebugType="$DEBUG_TYPE" \
        /p:EnableCompressionInSingleFile="$ENABLE_COMPRESSION_IN_SINGLE_FILE" \
        /p:PublishAot="$PUBLISH_AOT" \
        /p:PublishIISAssets="$PUBLISH_IIS_ASSETS" \
        /p:PublishSingleFile="$PUBLISH_SINGLE_FILE" \
        /p:PublishTrimmed="$PUBLISH_TRIMMED" \
        /p:TrimMode="$TRIM_MODE" \
        /p:Version="$VERSION"

# Remove IIS hosting assets if the SDK still emits them.
RUN rm -f /app/publish/aspnetcorev2_inprocess.dll

###################################
# Target: pre-publish-zip
###################################
FROM alpine:3.20 AS pre-publish-zip

ARG BUILDCHARTS_SRC
ARG PUBLISH_NAME
ARG RUNTIME=win-x64
ARG VERSION

# Install zip.
RUN apk add --no-cache zip

# Copy published output.
COPY --from=pre-publish /app/publish /app/publish

# Create zip.
RUN mkdir -p /output && \
    PUBLISH_NAME="${PUBLISH_NAME:-}" && \
    if [ -z "$PUBLISH_NAME" ]; then \
        PUBLISH_NAME="$(git -C /src config --get remote.origin.url 2>/dev/null || true)"; \
        PUBLISH_NAME="${PUBLISH_NAME##*/}"; \
        PUBLISH_NAME="${PUBLISH_NAME##*:}"; \
        PUBLISH_NAME="${PUBLISH_NAME%.git}"; \
    fi && \
    PUBLISH_NAME="${PUBLISH_NAME:-$(basename "$BUILDCHARTS_SRC" .csproj)}" && \
    cd /app/publish && \
    zip -r "/output/${PUBLISH_NAME}_${VERSION}_${RUNTIME}.zip" .

###################################
# Target: publish
###################################
FROM scratch AS publish
COPY --from=pre-publish-zip /output /