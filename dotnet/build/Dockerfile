###################################
# Target: build
###################################
FROM base AS build

ENV DOTNET_CLI_TELEMETRY_OPTOUT=true \
    DOTNET_SKIP_FIRST_TIME_EXPERIENCE=true

WORKDIR /src

ARG BUILDCHARTS_SRC
ARG VERSION
ARG CONFIGURATION=Release
ARG COMMIT

# Install the self-contained Azure Artifacts Credential Provider to enable support for VSS_NUGET_EXTERNAL_FEED_ENDPOINTS
RUN curl -L https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh | sh

# Copy project files and restore as distinct layer
COPY --link *.sln **/*.csproj **/nuget.config **/NuGet.Config . ./

#RUN dotnet clean

# Restore with secrets
RUN --mount=type=secret,id=VSS_NUGET_ACCESSTOKEN \
    --mount=type=secret,id=VSS_NUGET_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    bash -lc '\
      export VSS_NUGET_ACCESSTOKEN="$(cat /run/secrets/VSS_NUGET_ACCESSTOKEN)" && \
      export VSS_NUGET_EXTERNAL_FEED_ENDPOINTS="$(cat /run/secrets/VSS_NUGET_EXTERNAL_FEED_ENDPOINTS)" && \
      #dotnet nuget locals all --clear && \
      dotnet restore --locked-mode --use-lock-file \
    '

# Copy code
COPY --link . .

# Execute build
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet build "/src/$BUILDCHARTS_SRC" \
    --no-restore \
    --configuration "$CONFIGURATION" \
    -p:Version="$VERSION" \
    -p:ContinuousIntegrationBuild=true \
    -p:SourceRevisionId="$COMMIT"