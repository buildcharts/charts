###################################
# Target: build
###################################
FROM base AS build

WORKDIR /src
COPY . .

ARG BUILDCHARTS_SRC
ENV BUILDCHARTS_SRC=$BUILDCHARTS_SRC

ARG VERSION
ENV VERSION=$VERSION

ARG CONFIGURATION=Release
ENV CONFIGURATION=$CONFIGURATION

ARG COMMIT
ENV COMMIT=$COMMIT

RUN dotnet clean \
    --configuration $CONFIGURATION

RUN dotnet restore \
    --locked-mode \
    --use-lock-file

RUN dotnet build /src/$BUILDCHARTS_SRC \
    --no-restore \
    --configuration $CONFIGURATION \
    -p:Version=$VERSION \
    -p:ContinuousIntegrationBuild=true \
    -p:SourceRevisionId=$COMMIT
