###################################
# Target: test
###################################
FROM build AS test

ARG BUILDCHARTS_SRC
ARG CONFIGURATION=Release

# execute unit tests
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet test "/src/$BUILDCHARTS_SRC" \
    --no-build \
    --no-restore \
    --configuration "$CONFIGURATION" \
    --logger "trx;LogFileName=$(basename "$BUILDCHARTS_SRC" .csproj)_$(date +%Y%m%d-%H%M%S).trx" \
    --results-directory /tmp || \
    echo "⚠️ dotnet test failed, continuing so results can be verified later" 

# copy to output
RUN mkdir -p "/output/" && \
    cp /tmp/*.trx "/output/" 2>/dev/null || echo "⚠️ No .trx files found to copy"