# `buildcharts` for .NET 

`buildcharts` includes first-class support for .NET projects, using a set of reusable pipeline templates—called **charts**—that automate common workflows such as building, testing, packaging, and publishing .NET applications.

> Note that these charts is mostly for getting started. Preferably you bring your own charts for full control.

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

  - name: dotnet-test
    alias: test
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts
```

## Usage

### `type: build`
Builds the specified `.csproj` or `.sln` file using `dotnet build`.

```yaml
- src: src/MyApp/MyApp.csproj
  type: build
```

### `type: test`
Runs tests for a .NET project and collects results.

```yaml
- src: tests/MyApp.Tests/MyApp.Tests.csproj
  type: test
```

### `type: nuget`
Packs a .NET project into a NuGet package.


```yaml
- src: src/MyApp/MyApp.csproj
  type: nuget
  with:
    source: https://api.nuget.org/v3/index.json
    apiKey: ${{ secrets.NUGET_API_KEY }}
```

### `type: docker`
Publishes a .NET project and prepares a runtime docker image using the published output.

```yaml
- src: src/MyApp/MyApp.csproj
  type: docker
  with:
    base: mcr.microsoft.com/dotnet/aspnet:9.0
    tags: ["ghcr.io/your-org/myapp:latest"]
```