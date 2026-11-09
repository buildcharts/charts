###################################
# Target: pre-nuget
###################################
FROM build AS pre-nuget

ARG BUILDCHARTS_SRC
ARG VERSION
ARG CONFIGURATION=Release
ARG PATHS_INCLUDE
ARG SOURCELINK_STRICT=1

# Follow this for SourceLink + deterministic NuGet packages:
# - Ensure '.dockerignore' does NOT list '.git' (when not following the minimal git repository metadata guide in docs).
# - Ensure '.dockerignore' does NOT list '.gitignore' (so bin/ and obj/ are ignored correctly).
# - Ensure 'bin/' and 'obj/' are in .gitignore.
# - Add this to your .csproj:
#     <EmbedUntrackedSources>true</EmbedUntrackedSources>
#     <PublishRepositoryUrl>true</PublishRepositoryUrl>
# - Add this to your .csproj to include PDBs in the package (when not using symbol packages):
#     <AllowedOutputExtensionsInPackageBuildOutputFolder>$(AllowedOutputExtensionsInPackageBuildOutputFolder);.pdb</AllowedOutputExtensionsInPackageBuildOutputFolder>

RUN --mount=type=bind,from=git,source=.,target=/src/.git,readonly \
    set -eu; \
    echo "Verifying SourceLink prerequisites" >&2; \
    root="/src"; \
    warnings=0; \
    if [ ! -d "$root/.git" ] && [ ! -f "$root/.git" ]; then \
      warnings=1; \
      echo "warning: .git is missing inside the container (expected at: $root/.git)." >&2; \
      echo "- reason: .git is probably listed in .dockerignore, so it was not copied into the build context." >&2; \
      echo "- impact: SourceLink cannot embed repository metadata and deterministic validation may fail." >&2; \
      echo "- fix:    Remove '.git' from .dockerignore and rebuild the image." >&2; \
    fi; \
    if [ ! -f "$root/.gitignore" ]; then \
      warnings=1; \
      echo "warning: .gitignore is missing inside the container (expected at: $root/.gitignore)." >&2; \
      echo "- reason: .gitignore is probably listed in .dockerignore, so it was not copied into the build context." >&2; \
      echo "- impact: bin/ and obj/ may be treated as untracked sources and can break SourceLink/deterministic validation." >&2; \
      echo "- fix:    Remove '.gitignore' from .dockerignore and rebuild the image." >&2; \
    else \
      has_ignore_rule() { \
        pattern="$1"; \
        awk -v pat="$pattern" ' \
          /^[[:space:]]*#/ {next}                     # skip comments \
          /^[[:space:]]*$/ {next}                     # skip empty lines \
          { gsub(/^[[:space:]]+/, "", $0); }          # trim leading spaces \
          index($0, pat) == 1 { found=1 }             # line starts with pattern \
          END { exit(found ? 0 : 1) }                 \
        ' "$root/.gitignore"; \
      }; \
      if ! has_ignore_rule "bin/"; then \
        warnings=1; \
        echo "warning: 'bin/' is not ignored in .gitignore inside the container." >&2; \
        echo "- impact: build outputs under 'bin/' may be treated as tracked/untracked sources." >&2; \
        echo "- fix:    Add 'bin/' to the root .gitignore and commit the change." >&2; \
      fi; \
      if ! has_ignore_rule "obj/"; then \
        warnings=1; \
        echo "warning: 'obj/' is not ignored in .gitignore inside the container." >&2; \
        echo "- impact: SourceLink and deterministic builds may treat files in 'obj/' as sources (e.g. validation errors like 'contains untracked obj')." >&2; \
        echo "- fix:    Add 'obj/' to the root .gitignore and commit the change." >&2; \
      fi; \
    fi; \
    if [ "$warnings" -eq 0 ]; then \
      echo "SourceLink prerequisites OK" >&2; \
    else \
      if [ "$SOURCELINK_STRICT" = "1" ] || [ "$SOURCELINK_STRICT" = "true" ]; then \
        echo "SourceLink prerequisites completed with warnings" >&2; \
        exit 1; \
      else \
        echo "SourceLink prerequisites completed with warnings (SOURCELINK_STRICT=$SOURCELINK_STRICT; continuing)" >&2; \
      fi; \
    fi

RUN --mount=type=bind,from=git,source=.,target=/src/.git,readonly \
    --mount=type=cache,id=nuget,target=/root/.nuget/packages \
    --mount=type=cache,target=/root/.local/share/NuGet/v3-cache \
    --mount=type=cache,target=/root/.local/share/NuGet/plugins-cache \
    --mount=type=cache,target=/tmp/NuGetScratchroot \
    set -eu; \
    mkdir -p /output; \
    pack() { \
        dotnet pack "/src/$BUILDCHARTS_SRC" \
            --no-build \
            --configuration "$CONFIGURATION" \
            --output "/output" \
            /p:PackageVersion="$VERSION"; \
    }; \
    PATHS_INCLUDE="${PATHS_INCLUDE:-}"; \
    if [ -z "$PATHS_INCLUDE" ]; then \
        echo "PATHS_INCLUDE is empty. Running dotnet pack..." >&2; \
        pack; \
        exit 0; \
    fi; \
    # PATHS_INCLUDE scenarios:
    # 1. Default-branch CI build: detached HEAD at origin/HEAD -> compare HEAD^..HEAD.
    # 2. PR/feature build with origin/HEAD available: compare origin/HEAD...HEAD.
    # 3. PR merge commit without origin/HEAD: compare first parent to HEAD.
    # 4. Local changes are checked via branch diff, worktree diff, staged diff, and untracked files.
    # 5. Shallow checkout always runs pack to avoid false skips.
    SHALLOW_REPOSITORY="$(git -C /src rev-parse --is-shallow-repository 2>/dev/null || echo unknown)"; \
    if [ "$SHALLOW_REPOSITORY" = "true" ]; then \
        echo "warning: Git repository is shallow inside the container." >&2; \
        echo "- impact: PATHS_INCLUDE diff detection may incorrectly skip dotnet pack." >&2; \
        echo "Running dotnet pack to avoid a false skip." >&2; \
        pack; \
        exit 0; \
    fi; \
    COMPARE_REF=""; \
    DEFAULT_BRANCH_REF="$(git -C /src symbolic-ref --quiet --short refs/remotes/origin/HEAD || true)"; \
    HAS_PARENT="$(git -C /src rev-parse --verify --quiet HEAD^ >/dev/null && echo 1 || echo 0)"; \
    HAS_SECOND_PARENT="$(git -C /src rev-parse --verify --quiet HEAD^2 >/dev/null && echo 1 || echo 0)"; \
    IS_DETACHED_HEAD="$(git -C /src symbolic-ref --quiet HEAD >/dev/null && echo 0 || echo 1)"; \
    IS_DEFAULT_HEAD="$(test -n "$DEFAULT_BRANCH_REF" && test "$(git -C /src rev-parse HEAD)" = "$(git -C /src rev-parse "$DEFAULT_BRANCH_REF")" && echo 1 || echo 0)"; \
    if [ "$IS_DETACHED_HEAD" = "1" ] && [ "$IS_DEFAULT_HEAD" = "1" ]; then \
        if [ "$HAS_PARENT" != "1" ]; then \
            echo "No parent commit found for default-branch HEAD. Running dotnet pack..." >&2; \
            pack; \
            exit 0; \
        fi; \
        COMPARE_REF="HEAD^..HEAD"; \
    elif [ -n "$DEFAULT_BRANCH_REF" ]; then \
        COMPARE_REF="$DEFAULT_BRANCH_REF...HEAD"; \
    elif [ "$IS_DETACHED_HEAD" = "1" ] && [ "$HAS_SECOND_PARENT" = "1" ]; then \
        COMPARE_REF="HEAD^1..HEAD"; \
    fi; \
    if [ -z "$COMPARE_REF" ]; then \
        # No remote compare ref found. With fetchDepth=0 + explicit fetch in CI,
        # this path should be rare. Fail open to avoid false skips.
        echo "No origin compare ref found. Running dotnet pack..." >&2; \
        pack; \
        exit 0; \
    fi; \
    echo "Detecting changes in $PATHS_INCLUDE against $COMPARE_REF" >&2; \
    HAS_CHANGES=0; \
    # Check committed branch changes.
    git -C /src -c core.autocrlf=false -c core.safecrlf=false diff --quiet --exit-code "$COMPARE_REF" -- $PATHS_INCLUDE || HAS_CHANGES=1; \
    # Check unstaged changes; ignore CRLF-only differences from Windows-mounted worktrees.
    git -C /src -c core.autocrlf=false -c core.safecrlf=false diff --ignore-cr-at-eol --exit-code -- $PATHS_INCLUDE >/dev/null || HAS_CHANGES=1; \
    # Check staged changes.
    git -C /src -c core.autocrlf=false -c core.safecrlf=false diff --cached --quiet --exit-code -- $PATHS_INCLUDE || HAS_CHANGES=1; \
    # Check untracked files; ignore packages.lock.json generated by restore.
    if [ -n "$(git -C /src ls-files --others --exclude-standard -- $PATHS_INCLUDE | grep -Ev '(^|/)packages\.lock\.json$' || true)" ]; then \
        HAS_CHANGES=1; \
    fi; \
    if [ "$HAS_CHANGES" -eq 0 ]; then \
        echo "No changes detected ($COMPARE_REF). Skipping nuget pack..." >&2; \
        exit 0; \
    fi; \
    echo "Changes detected. Running dotnet pack..." >&2; \
    pack

###################################
# Target: nuget
###################################
FROM scratch AS nuget
COPY --from=pre-nuget /output /