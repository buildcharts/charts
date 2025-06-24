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

# copy project files and restore as distinct layer
COPY --link *.csproj **/nuget.config . ./

RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet restore \
    --locked-mode \
    --use-lock-file

COPY --link . .

# app source & build
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet build "/src/$BUILDCHARTS_SRC" \
    --no-restore \
    --configuration "$CONFIGURATION" \
    -p:Version="$VERSION" \
    -p:ContinuousIntegrationBuild=true \
    -p:SourceRevisionId="$COMMIT"