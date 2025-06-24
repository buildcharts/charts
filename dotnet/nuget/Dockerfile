###################################
# Target: nuget
###################################
FROM build AS nuget

ARG BUILDCHARTS_SRC
ARG VERSION
ARG CONFIGURATION=Release
ARG PACK_OUTPUT=/output

# pack the code into a nuget package
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet pack /src/"$BUILDCHARTS_SRC" \
    --no-build \
    --configuration "$CONFIGURATION" \
    --output "$PACK_OUTPUT" \
    /p:PackageVersion="$VERSION"