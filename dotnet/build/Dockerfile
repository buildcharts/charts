# syntax=docker/dockerfile:1-labs

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

# Install the self-contained Azure Artifacts Credential Provider to enable support for VSS_NUGET_EXTERNAL_FEED_ENDPOINTS.
RUN curl -L https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh | sh

# Copy project files and restore as distinct layer.
COPY --link --parents *.sln **/*.csproj **/NuGet.Config **/Directory.Build.props **/Directory.Build.targets **/Directory.Packages.props global.json ./

# Optional: Clean is only needed when bin/obj folders are NOT included in .dockerignore, which is best practice, so we skip it.
#RUN dotnet clean 

# Restore with secrets.
RUN --mount=type=secret,id=VSS_NUGET_ACCESSTOKEN,env=VSS_NUGET_ACCESSTOKEN \
    --mount=type=secret,id=VSS_NUGET_EXTERNAL_FEED_ENDPOINTS,env=VSS_NUGET_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
      dotnet restore "/src/$BUILDCHARTS_SRC" --locked-mode --use-lock-file

# Copy code.
COPY --link . .

# # Optional: Normalize line endings for SourceLink/determinism.
# # - Prefer .gitattributes: '* text=auto eol=lf' to enforce LF.
# # - If incorrect git (when running inside docker) may automatically add CRLF line endings which breaks determinism (PDB files links to non-existing sources in git).
# # - Validate with `sourcelink test <filename>.pdb` (for PDB files inside the nuget package).
# RUN git -C /src config core.autocrlf false \
#     && git -C /src checkout-index -f --all

# Normalize line endings via git without losing local changes:
# 1) stage current working tree (including uncommitted edits from host)
# 2) re-checkout from index with LF-only settings
RUN if git -C /src rev-parse --is-inside-work-tree >/dev/null 2>&1 && \
      git -C /src diff --quiet && git -C /src diff --cached --quiet; then \
      git -C /src config core.autocrlf false; \
      git -C /src config core.eol lf; \
      git -C /src checkout-index -f --all; \
    else \
      echo "Skipping line-ending normalization; working tree is dirty"; \
    fi
    


# Execute build.
RUN --mount=type=secret,id=VSS_NUGET_ACCESSTOKEN,env=VSS_NUGET_ACCESSTOKEN \
    --mount=type=secret,id=VSS_NUGET_EXTERNAL_FEED_ENDPOINTS,env=VSS_NUGET_EXTERNAL_FEED_ENDPOINTS \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    dotnet build "/src/$BUILDCHARTS_SRC" \
        --no-restore \
        --configuration "$CONFIGURATION" \
        /p:ContinuousIntegrationBuild=true \
        /p:SourceRevisionId="$COMMIT" \
        /p:Version="$VERSION"