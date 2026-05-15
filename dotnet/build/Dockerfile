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

COPY <<EOF /BuildCharts.Directory.Build.props
<Project>
  <PropertyGroup Condition="'$(BUILDCHARTS_ENABLE_OBJ_CACHE)' != 'false'">
    <BaseIntermediateOutputPath>/src/.buildcache/obj/$(BUILDCHARTS_TYPE)/$(MSBuildProjectName)/</BaseIntermediateOutputPath>
  </PropertyGroup>
  <PropertyGroup>
    <DefaultItemExcludes>$(DefaultItemExcludes);**/bin/**;**/obj/**</DefaultItemExcludes>
  </PropertyGroup>
</Project>
EOF

# Copy project files and restore as distinct layer.
COPY --link --parents *.sln *.slnx **/*.csproj **/*.dcproj **/[Nn]u[Gg]et.[Cc]onfig **/Directory.Build.props **/Directory.Build.targets **/Directory.Packages.props **/global.json ./

# Optional: Clean is only needed when bin/obj folders are NOT included in .dockerignore, which is best practice, so we skip it.
#RUN dotnet clean 

ARG BUILDCHARTS_SRC
ARG BUILDCHARTS_TYPE=build
ARG BUILDCHARTS_CACHE_SCOPE=default
ARG BUILDCHARTS_ENABLE_OBJ_CACHE=true
ARG CONFIGURATION=Release
ARG TARGETARCH
ENV BUILDCHARTS_TYPE=$BUILDCHARTS_TYPE \
    BUILDCHARTS_ENABLE_OBJ_CACHE=$BUILDCHARTS_ENABLE_OBJ_CACHE \
    CustomBeforeDirectoryBuildProps=/BuildCharts.Directory.Build.props

# Restore with secrets.
RUN --mount=type=secret,id=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS,env=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
    --mount=type=cache,id=msbuild-obj-${BUILDCHARTS_CACHE_SCOPE}-${BUILDCHARTS_TYPE}-${CONFIGURATION}-${TARGETARCH},target=/src/.buildcache/obj,sharing=locked \
      dotnet restore "/src/$BUILDCHARTS_SRC" --locked-mode --use-lock-file

# Copy code.
COPY --link . .

ARG COMMIT

# Execute build.
RUN --mount=type=secret,id=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS,env=ARTIFACTS_CREDENTIALPROVIDER_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
    --mount=type=cache,id=msbuild-obj-${BUILDCHARTS_CACHE_SCOPE}-${BUILDCHARTS_TYPE}-${CONFIGURATION}-${TARGETARCH},target=/src/.buildcache/obj,sharing=locked \
    dotnet build "/src/$BUILDCHARTS_SRC" \
        --no-restore \
        --configuration "$CONFIGURATION" \
        /p:ContinuousIntegrationBuild=true \
        /p:SourceRevisionId="$COMMIT"