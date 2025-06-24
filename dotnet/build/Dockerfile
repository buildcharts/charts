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

# Optional: Allow git operations to operate on the repository. Recent versions of git require the .git folder to be owned by the same user.
# - This can be skipped when no git-operations in dockerfiles.
# RUN git -C /src config core.autocrlf false \
#     && git -C /src checkout-index -f --all

# Optional: Normalize LF line endings inside the containers, for SourceLink/determinism.
# - Prefer .gitattributes: '* text=auto eol=lf' to enforce LF.
# - If incorrect git (when running inside docker) may automatically add LF line endings which breaks determinism (PDB files links to non-existing sources in git).
# - Validate with `sourcelink test <filename>.pdb` (for PDB files inside the nuget package).
RUN set -eu; \
    if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then \
        echo "Skipping EOL cleanup (.git not present)"; \
        exit 0; \
    fi; \
    git config core.autocrlf false; \
    git config core.eol lf; \
    git diff --name-only > /tmp/git-all-changed; \
    git diff --name-only --ignore-space-at-eol > /tmp/git-real-changed; \
    files="$(grep -Fxv -f /tmp/git-real-changed /tmp/git-all-changed || true)"; \
    if [ -n "$files" ]; then \
        count="$(printf '%s\n' "$files" | wc -l)"; \
        echo "Found $count file(s) changed only due to normalized line endings:"; \
        printf '%s\n' "$files"; \
        printf '%s\n' "$files" | xargs -r git checkout --; \
        echo "Reverted EOL-only changes in those file(s)."; \
    else \
        echo "No files changed only due to normalized line endings."; \
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