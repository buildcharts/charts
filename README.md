# `buildcharts`

`buildcharts` are OCI images with charts for generating declarative CI/CD pipelines.

- Specified in the dependency chart `chart/buildcharts/Chart.yaml`.
- Integrates with Renovate/dependabot for maintenance updates.

## Built-in variables

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

## Output

All Dockerfiles may produce build output by creating files inside the `/output` directory.  In addition, the generated Bake file defines a separate `output` build target that automatically copies the `/output` directory to `./buildcharts/output` on the host during `docker buildx bake`.  This makes it easy to persist artifacts – such as unit-test results or generated NuGet packages – out of the container and into your working directory for further processing.

## Documentation
- [dotnet/README.md](dotnet/README.md) – Charts for building .NET applications
