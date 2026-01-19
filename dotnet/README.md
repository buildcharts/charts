# Build charts for .NET 

`buildcharts` includes first-class support for .NET projects, using a set of reusable pipeline templates - called **charts** - that automate common workflows such as building, testing, packaging, and publishing .NET application. Consist of a set of dockerfiles that follows docker guidance and best-practices.

> [!NOTE]
> These charts is mostly for getting started. Preferably you bring your own charts for full control.

## File structure
```
dotnet/
├── build/
│   ├── Chart.yaml
│   └── Dockerfile
├── docker/
│   ├── Chart.yaml
│   └── Dockerfile
├── nuget/
│   ├── Chart.yaml
│   └── Dockerfile
├── publish/
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
  - name: dotnet-build
    alias: build
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: dotnet-docker
    alias: docker
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: dotnet-nuget
    alias: nuget
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: dotnet-publish
    alias: publish
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: dotnet-test
    alias: test
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts
```

## Usage

### `type: build`
Builds the specified `.csproj` or `.sln` file using `dotnet build`.

```yaml
targets:
  src/MyApp/MyApp.csproj:
    type: build
```

### `type: docker`
Publishes a .NET project and prepares a runtime docker image using the published output.

```yaml
targets:
  src/MyApp/MyApp.csproj:
    type: docker
    with:
      base: mcr.microsoft.com/dotnet/aspnet:9.0
      tags: ["ghcr.io/your-org/myapp:latest"]
```

### `type: nuget`
Packs a .NET project into a NuGet package.

```yaml
targets:
  src/MyApp/MyApp.csproj:
    type: nuget
    with:
      # `pathsInclude` can be used to only pack nuget when git diffs in origin/HEAD includes specific paths.
      pathsInclude: src/MyDependentContractsNuget src/MyDependentContractsNuget
```

### `type: publish`
Publishes a .NET project using `dotnet publish`.

```yaml
targets:
  src/MyApp/MyApp.csproj:
    type: publish
```

### `type: test`
Runs tests for a .NET project and collects results.

```yaml
targets:
  tests/MyApp.Tests/MyApp.Tests.csproj:
    type: test
```

## Links
- https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-test-results-v2?view=azure-pipelinesl#docker
- https://learn.microsoft.com/en-us/dotnet/core/project-sdk/msbuild-props#continuousintegrationbuild
- https://learn.microsoft.com/en-us/dotnet/standard/library-guidance/sourcelink
- https://github.com/microsoft/artifacts-credprovider
- https://github.com/dotnet/dotnet-docker/blob/main/README.md
- https://github.com/dotnet/dotnet-docker/blob/main/documentation/scenarios/nuget-credentials.md
- https://github.com/dotnet/sourcelink/tree/main/docs#minimal-git-repository-metadata