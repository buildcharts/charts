# syntax=docker/dockerfile:1

###################################
# Target: pre-test
###################################
FROM build AS pre-test

ARG BUILDCHARTS_SRC
ARG VERSION
ARG JACOCO_VERSION=0.8.16

# Configure Testcontainers to talk to the daemon used by TestcontainersDinD@v1.
ARG DOCKER_HOST=tcp://host.docker.internal:2375
ARG TESTCONTAINERS_HOST_OVERRIDE=

# Execute tests and collect JUnit and JaCoCo reports, including failed tests.
RUN --mount=type=secret,id=settings.xml,target=/root/.m2/settings.xml \
    --mount=type=cache,id=maven,target=/root/.m2/repository/cached,sharing=shared \
    set -eu; \
    mvn --file "/src/$BUILDCHARTS_SRC" \
        -Drevision="$VERSION" \
        -Dmaven.test.failure.ignore=true \
        "org.jacoco:jacoco-maven-plugin:$JACOCO_VERSION:prepare-agent" \
        verify \
        "org.jacoco:jacoco-maven-plugin:$JACOCO_VERSION:report"; \
    name="$(dirname "$BUILDCHARTS_SRC")"; [ "$name" = . ] && name=root; \
    results="/output/$(printf '%s' "$name" | tr '/' '-')_$(date +%Y%m%d-%H%M%S)"; \
    mkdir -p "$results"; \
    find . -type f \( \
        -path '*/target/surefire-reports/TEST-*.xml' -o \
        -path '*/target/failsafe-reports/TEST-*.xml' -o \
        -path '*/target/site/jacoco/jacoco.xml' \
    \) -exec cp --parents --target-directory="$results" {} +

###################################
# Target: test-reports
###################################
FROM python:3.13-slim AS test-reports

COPY --from=pre-test /output/ /output/

# Convert JaCoCo line and branch counts to Cobertura.
RUN --mount=type=bind,from=pre-test,source=/src,target=/src,readonly <<'PY' python
from pathlib import Path
from time import time
import xml.etree.ElementTree as xml


def coverage_counts(lines):
    return {
        "lines-covered": sum(int(line.attrib["ci"]) > 0 for line in lines),
        "lines-valid": len(lines),
        "branches-covered": sum(int(line.attrib["cb"]) for line in lines),
        "branches-valid": sum(int(line.attrib["cb"]) + int(line.attrib["mb"]) for line in lines),
    }


def coverage_rates(counts):
    line_total = counts["lines-valid"]
    branch_total = counts["branches-valid"]
    return {
        "line-rate": str(counts["lines-covered"] / line_total if line_total else 0),
        "branch-rate": str(counts["branches-covered"] / branch_total if branch_total else 0),
        "complexity": "0",
    }


def source_file(module, package_path, name):
    relative = Path(package_path) / name
    base = Path("/src") / module
    roots = [base / "src/main/java", base / "src/main/kotlin", *sorted((base / "target/generated-sources").glob("*"))]
    for root in roots:
        if (root / relative).is_file():
            return (root / relative).relative_to("/src")
    print(f"warning: no source file for {module / relative}, using src/main/java.")
    return module / "src/main/java" / relative


output = Path("/output")
for junit in output.glob("*/**/target/*-reports/TEST-*.xml"):
    target = junit.parent.parent
    classes = Path("/src").joinpath(*target.relative_to(output).parts[1:], "classes")
    if not (target / "site/jacoco/jacoco.xml").is_file() and any(classes.rglob("*.class")):
        if any(case.find("skipped") is None for case in xml.parse(junit).iter("testcase")):
            print(f"warning: no JaCoCo XML report for {classes.parent.parent.relative_to('/src')}. Start the Surefire and Failsafe argLine with @{{argLine}} and declare an empty <argLine/> property.")

reports = list(output.glob("*/**/target/site/jacoco/jacoco.xml"))
if not reports:
    print("No JaCoCo reports, so no Cobertura coverage is written.")
    raise SystemExit(0)

for report in reports:
    module = Path(*report.relative_to(output).parts[1:-4])
    jacoco = xml.parse(report).getroot()
    counts = coverage_counts(jacoco.findall("./package/sourcefile/line"))
    coverage = xml.Element("coverage", coverage_rates(counts))
    coverage.attrib.update({name: str(count) for name, count in counts.items()})
    coverage.attrib.update(version="1.0", timestamp=str(int(time())))
    xml.SubElement(xml.SubElement(coverage, "sources"), "source").text = "."
    packages = xml.SubElement(coverage, "packages")

    for package in jacoco.findall("package"):
        package_path = package.attrib["name"]
        package_name = package_path.replace("/", ".")
        package_rates = coverage_rates(coverage_counts(package.findall("./sourcefile/line")))
        package_element = xml.SubElement(packages, "package", name=package_name, **package_rates)
        classes = xml.SubElement(package_element, "classes")

        for source in package.findall("sourcefile"):
            filename = source_file(module, package_path, source.attrib["name"])
            source_lines = source.findall("line")
            source_rates = coverage_rates(coverage_counts(source_lines))
            class_name = ".".join(part for part in (package_name, filename.stem) if part)
            source_element = xml.SubElement(
                classes, "class", name=class_name, filename=filename.as_posix(), **source_rates
            )
            xml.SubElement(source_element, "methods")
            lines = xml.SubElement(source_element, "lines")

            for line in source_lines:
                covered = int(line.attrib["cb"])
                total = covered + int(line.attrib["mb"])
                element = xml.SubElement(
                    lines, "line", number=line.attrib["nr"],
                    hits="1" if int(line.attrib["ci"]) > 0 else "0",
                    branch="true" if total else "false",
                )
                if total:
                    element.set("condition-coverage", f"{100 * covered // total}% ({covered}/{total})")

    xml.indent(coverage)
    xml.ElementTree(coverage).write(report.with_name("coverage.cobertura.xml"), encoding="utf-8", xml_declaration=True)
PY

###################################
# Target: test
###################################
FROM scratch AS test
COPY --from=test-reports /output/ /
