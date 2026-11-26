# Build charts for Java

`buildcharts` provides reusable Dockerfile-based charts for Java applications.

> [!NOTE]
> These charts are for getting started. Bring your own charts when you need full control.

## File structure

```
java/
├── build/
│   ├── Chart.yaml
│   └── Dockerfile
├── maven/
│   ├── Chart.yaml
│   └── Dockerfile
└── test/
    ├── Chart.yaml
    └── Dockerfile
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
  - name: java-build
    alias: build
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: java-maven
    alias: maven
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts

  - name: java-test
    alias: test
    version: 0.0.1
    repository: oci://registry-1.docker.io/buildcharts
```

## Usage

Use a base image with Maven 3.9 or later, Java 11 or later, GNU coreutils, and GNU findutils.

### `type: build`

Builds the specified Maven project.

```yaml
targets:
  pom.xml:
    type: build
    with:
      contexts:
        base: docker-image://maven:3.9.11-eclipse-temurin-21
```

### `type: maven`

Packs a Java project into a Maven package. Use a base image with Git. Set `VERSION` and target a POM that declares its own version. Set `contexts.git` to `.git`. Use `COMMIT` to set the SCM tag in the POM and the SCM revision in the JAR.
The chart writes `VERSION` into the exported POM with `versions:set`. This handles both a literal version such as `1.0.0-SNAPSHOT` and `${revision}` without requiring the Flatten Maven Plugin for version resolution.

```yaml
targets:
  pom.xml:
    type: maven
    with:
      contexts:
        git: .git
      args:
        # `pathsInclude` checks these paths for changes against `origin/HEAD` before packing.
        pathsInclude: pom.xml src
```

### `type: test`

Runs Maven tests and exports JUnit and Cobertura reports.

If your POM sets Surefire or Failsafe’s `argLine`, start it with `@{argLine}` so Maven passes the JaCoCo agent to the test JVM. An empty property default lets plain `mvn test` run without JaCoCo.

```xml
<properties>
    <argLine/>
</properties>
<!-- In build/plugins -->
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>@{argLine} -javaagent:${org.mockito:mockito-core:jar}</argLine>
    </configuration>
</plugin>
```

```yaml
targets:
  pom.xml:
    type: test
```

## Feed authentication

Enable `MavenAuthenticate@v1` for Azure Artifacts feeds in `pom.xml`.

```yaml
plugins:
  - MavenAuthenticate@v1
```

In Azure Pipelines, pass `$(System.AccessToken)` as `SYSTEM_ACCESSTOKEN` and grant the build identity access to the feed. Keep credentials out of `pom.xml` and `.mvn/`. Exclude `.buildcharts/` and any `settings.xml` containing credentials from the Docker context.

## Links

- https://learn.microsoft.com/en-us/azure/devops/artifacts/maven/project-setup-maven
