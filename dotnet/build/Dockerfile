# syntax=docker/dockerfile:1-labs

###################################
# Target: build
###################################
FROM base AS build

ENV DOTNET_CLI_TELEMETRY_OPTOUT=true \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=true

WORKDIR /src

# Install the self-contained Azure Artifacts Credential Provider to enable support for ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS.
RUN curl -L https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh | sh

# Copy project files and restore as distinct layer.
COPY --link --parents *.sln *.slnx **/*.csproj **/*.dcproj **/[Nn]u[Gg]et.[Cc]onfig **/Directory.Build.props **/Directory.Build.targets **/Directory.Packages.props **/global.json **/packages.lock.json ./

# Optional: Clean is only needed when bin/obj folders are NOT included in .dockerignore, which is best practice, so we skip it.
#RUN dotnet clean 

ARG BUILDCHARTS_SRC

# Restore with secrets.
# --use-lock-file creates missing lock files; --locked-mode only fails when an existing lock file is out of sync.
RUN --mount=type=secret,id=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS,env=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
      dotnet restore "/src/$BUILDCHARTS_SRC" --locked-mode --use-lock-file

# Copy code.
COPY --link . .

ARG CONFIGURATION=Release
ARG COMMIT
ARG VERSION

# Execute build.
RUN --mount=type=secret,id=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS,env=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
    dotnet build "/src/$BUILDCHARTS_SRC" \
        --no-restore \
        --configuration "$CONFIGURATION" \
        /p:ContinuousIntegrationBuild=true \
        /p:SourceRevisionId="$COMMIT" \
        /p:Version="$VERSION"
