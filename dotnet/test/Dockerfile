###################################
# Target: test
###################################
FROM build AS test

ARG BUILDCHARTS_SRC
ENV BUILDCHARTS_SRC=$BUILDCHARTS_SRC

ARG CONFIGURATION=Release
ENV CONFIGURATION=$CONFIGURATION

ARG RESULTS_OUTPUT=/output
ENV RESULTS_OUTPUT=$RESULTS_OUTPUT

RUN dotnet test /src/$BUILDCHARTS_SRC \
    --no-build \
    --no-restore \
    --configuration $CONFIGURATION \
    --logger "trx;LogFileName=$(basename "$BUILDCHARTS_SRC" .csproj)_$(date +%Y%m%d-%H%M%S).trx" \
    --results-directory /tmp || \
    echo "⚠️ dotnet test failed, continuing so results can be verified later" 

RUN mkdir -p "$RESULTS_OUTPUT" && \
    cp /tmp/*.trx "$RESULTS_OUTPUT/" 2>/dev/null || echo "⚠️ No .trx files found to copy"
