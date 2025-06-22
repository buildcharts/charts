###################################
# Target: pre-docker
###################################
FROM build AS publish

ARG BUILDCHARTS_SRC
ENV BUILDCHARTS_SRC=$BUILDCHARTS_SRC

ARG CONFIGURATION=Release
ENV CONFIGURATION=$CONFIGURATION

ARG PUBLISH_OUTPUT=/output
ENV PUBLISH_OUTPUT=$PUBLISH_OUTPUT

RUN dotnet publish /src/$BUILDCHARTS_SRC \
    --no-build \
    --no-restore \
    --configuration $CONFIGURATION \
    --output $PUBLISH_OUTPUT

RUN DLL_NAME=$(basename "$BUILDCHARTS_SRC" .csproj).dll && \
    echo '#!/bin/sh' > /entrypoint.sh && \
    echo "exec dotnet /app/$DLL_NAME \"\$@\"" >> /entrypoint.sh && \
    chmod +x /entrypoint.sh

###################################
# Target: docker
###################################
FROM base AS docker

WORKDIR /app
COPY --link --from=publish /entrypoint.sh /entrypoint.sh
COPY --link --from=publish /output .
ENTRYPOINT ["/entrypoint.sh"]
