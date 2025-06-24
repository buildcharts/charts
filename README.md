# `buildcharts`

`buildcharts` are OCI images with charts for generating declarative CI/CD pipelines.

- Specified in the dependency chart `chart/buildcharts/Chart.yaml`.
- Integrates with Renovate/dependabot for maintenance updates.

## Built-in default environment variables

Every target in a metadata specification automatically gets a set of build-provided variables. These default variables are injected during generation and can be accessed in your build charts during processing.

```yaml
targets:
  src/MyApp/MyApp.csproj:
    type: build
```

### `BUILDCHARTS_SRC`
The target source path, eg. `src/MyApp/MyApp.csproj`.

### `BUILDCHARTS_TYPE`
The target source type, eg. `build`. Resolves the chart identified by `alias: build`.

## Documentation
- [dotnet/README.md](dotnet/README.md) – Charts for building dotnet applications
