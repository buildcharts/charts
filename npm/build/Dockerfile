###################################
# Target: build
###################################
FROM base AS build

WORKDIR /src

# Enable pnpm through Corepack.
RUN corepack enable

# Copy project files and restore as distinct layer.
COPY --link package.json pnpm-lock.yaml ./

# Restore with secrets.
RUN --mount=type=secret,id=npmrc,target=/src/.npmrc \
    --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --strict-peer-dependencies=false --prefer-offline

# Copy code.
COPY --link . .

ARG NPM_SCRIPT=build-prod

# Execute build.
RUN pnpm run "$NPM_SCRIPT" \
    --output-path=dist
