# Build charts for npm

`buildcharts` provides reusable Dockerfile-based charts for building, testing,
packaging, and publishing npm applications. The build chart installs dependencies
with pnpm. Docker, npm, and test targets reuse the automatic `build` context.

> [!NOTE]
> These charts is mostly for getting started. Preferably you bring your own charts for full control.

## File structure
```
npm/
├── build/
│   ├── Chart.yaml
│   └── Dockerfile
├── docker/
│   ├── Chart.yaml
│   └── Dockerfile
├── npm/
│   ├── Chart.yaml
│   └── Dockerfile
├── test/
│   ├── Chart.yaml
│   └── Dockerfile
```

## Dependency chart

The dependency chart at `charts/buildcharts/Chart.yaml` is used to resolve specific OCI images.

```yaml
apiVersion: v2
name: buildcharts
version: 0.0.1

description: A meta-chart to define build pipeline targets and templates

type: application

dependencies:
  - name: npm-build
    alias: build
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: npm-docker
    alias: docker
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: npm-npm
    alias: npm
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: npm-test
    alias: test
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts
```

## Usage

### `type: build`
Installs dependencies and builds the application using pnpm.

```yaml
targets:
  my-app:
    type: build
    with:
      contexts:
        base: docker-image://node:22-bookworm-slim
      args:
        npm_script: build-prod
```

### `type: docker`
Prepares an Nginx image using the compiled application.

```yaml
targets:
  my-app:
    type: docker
    with:
      contexts:
        base: docker-image://nginx:1.31.8-alpine-slim
      tags:
        - example.com/my-app:latest
```

### `type: npm`
Packs a compiled npm package using `pnpm pack`.

```yaml
targets:
  dist/my-library:
    type: npm
    with:
      contexts:
        git: .git
      args:
        pathsInclude: projects/my-library projects/shared
```

### `type: test`
Runs tests and collects results.

```yaml
targets:
  my-app:
    type: test
    with:
      contexts:
        base: docker-image://mcr.microsoft.com/playwright:v1.65.0-noble
      args:
        npm_script: test
```

## Links
- https://learn.microsoft.com/en-us/azure/devops/artifacts/npm/npmrc
