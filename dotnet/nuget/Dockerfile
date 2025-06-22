###################################
# Target: nuget
###################################
FROM build AS nuget

ARG BUILDCHARTS_SRC
ENV BUILDCHARTS_SRC=$BUILDCHARTS_SRC

ARG VERSION
ENV VERSION=$VERSION

ARG PACK_OUTPUT=/output
ENV PACK_OUTPUT=$PACK_OUTPUT

RUN dotnet pack /src/$BUILDCHARTS_SRC \
    --no-build \
    --configuration $CONFIGURATION \
    --output $PACK_OUTPUT \
    /p:PackageVersion=$VERSION
