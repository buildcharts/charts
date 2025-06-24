###################################
# Target: pre-docker
###################################
FROM build AS publish

ARG BUILDCHARTS_SRC
ARG CONFIGURATION=Release
ARG PUBLISH_OUTPUT=/output

# publish application and dependencies for deployment
RUN --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet publish /src/"$BUILDCHARTS_SRC" \
    --no-build \
    --no-restore \
    --configuration "$CONFIGURATION" \
    --output "$PUBLISH_OUTPUT"

# ensure static entrypoint in final image
RUN DLL_NAME=$(basename "$BUILDCHARTS_SRC" .csproj).dll && \
    echo "#!/bin/sh" > /entrypoint.sh && \
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