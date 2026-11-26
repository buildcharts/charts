# syntax=docker/dockerfile:1

###################################
# Target: pre-maven
###################################
FROM build AS pre-maven

ARG BUILDCHARTS_SRC
ARG VERSION
ARG COMMIT
ARG PATHS_INCLUDE
ARG VERSIONS_PLUGIN_VERSION=2.25.3

# Stamp the POM and JAR with SCM metadata when the Maven target has Git context.
COPY <<'JAVA' /tmp/StampMavenScm.java
import java.nio.file.*;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.transform.TransformerFactory;
import javax.xml.transform.dom.DOMSource;
import javax.xml.transform.stream.StreamResult;
import javax.xml.xpath.*;
import org.w3c.dom.Element;

class StampMavenScm {
    private static final XPath XPATH = XPathFactory.newInstance().newXPath();

    public static void main(String[] args) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        factory.setNamespaceAware(true);
        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        Path pom = Paths.get(args[0]);
        Element project = factory.newDocumentBuilder().parse(pom.toFile()).getDocumentElement();

        Element scm = child(project, "scm");
        scm.setAttribute("child.scm.url.inherit.append.path", "false");
        child(scm, "url").setTextContent(args[1]);
        child(scm, "tag").setTextContent(args[2]);

        Element plugins = child(child(project, "build"), "plugins");
        String jarArtifact = element("artifactId") + "[normalize-space()='maven-jar-plugin']";
        String group = element("groupId");
        String defaultGroup = "not(" + group + ") or " + group + "[normalize-space()='org.apache.maven.plugins']";
        String jarSelector = element("plugin") + "[" + jarArtifact + " and (" + defaultGroup + ")]";
        Element jar = (Element) XPATH.evaluate(jarSelector, plugins, XPathConstants.NODE);
        if (jar == null) {
            jar = append(plugins, "plugin");
            child(jar, "artifactId").setTextContent("maven-jar-plugin");
        }

        Element entries = child(child(child(jar, "configuration"), "archive"), "manifestEntries");
        child(entries, "SCM-Repository").setTextContent(args[1]);
        child(entries, "SCM-Revision").setTextContent(args[2]);

        TransformerFactory.newInstance().newTransformer().transform(new DOMSource(project.getOwnerDocument()), new StreamResult(pom.toFile()));
    }

    private static String element(String name) {
        return "*[local-name()='" + name + "' and namespace-uri()=namespace-uri(..)]";
    }

    private static Element child(Element parent, String name) throws XPathExpressionException {
        Element existing = (Element) XPATH.evaluate(element(name), parent, XPathConstants.NODE);
        return existing == null ? append(parent, name) : existing;
    }

    private static Element append(Element parent, String name) {
        return (Element) parent.appendChild(parent.getOwnerDocument().createElementNS(parent.getNamespaceURI(), name));
    }
}
JAVA

# Package the reactor and export locally installed artifacts.
RUN --mount=type=bind,from=git,source=.,target=/src/.git,readonly \
    --mount=type=secret,id=settings.xml,target=/root/.m2/settings.xml \
    --mount=type=cache,id=maven,target=/root/.m2/repository/cached,sharing=shared \
    set -eu; \
    if ! command -v git >/dev/null 2>&1; then \
        echo "Git is required in the Maven base image." >&2; \
        exit 1; \
    fi; \
    mkdir -p /output; \
    pack() { \
        repository_url="$(git config --file /src/.git/config --get remote.origin.url 2>/dev/null | sed -E 's#^([[:alpha:]][[:alnum:]+.-]*://)[^/]*@#\1#; s/[?#].*$//')"; \
        if [ -n "$repository_url" ]; then \
            java /tmp/StampMavenScm.java "/src/$BUILDCHARTS_SRC" "$repository_url" "$COMMIT"; \
        fi; \
        set -- --file "/src/$BUILDCHARTS_SRC" \
            -Drevision="$VERSION" \
            -Dcommit="$COMMIT" \
            -DrepositoryUrl="$repository_url"; \
        mvn "$@" "org.codehaus.mojo:versions-maven-plugin:$VERSIONS_PLUGIN_VERSION:set" \
            -DnewVersion="$VERSION" \
            -DprocessAllModules=true \
            -DprocessFromLocalAggregationRoot=false \
            -DgenerateBackupPoms=false \
            -DupdateBuildOutputTimestampPolicy=never; \
        mvn "$@" -DskipTests install; \
        duplicates="$(find /root/.m2/repository/installed -type f ! -name 'maven-metadata-*' ! -name '_remote.repositories' -printf '%f\n' | sort | uniq -d)"; \
        if [ -n "$duplicates" ]; then echo "Artifacts with the same file name: $duplicates" >&2; exit 1; fi; \
        find /root/.m2/repository/installed -type f ! -name 'maven-metadata-*' ! -name '_remote.repositories' -exec cp -t /output/ {} +; \
    }; \
    PATHS_INCLUDE="${PATHS_INCLUDE:-}"; \
    if [ -z "$PATHS_INCLUDE" ]; then \
        echo "PATHS_INCLUDE is empty. Running mvn install..." >&2; \
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
        echo "- impact: PATHS_INCLUDE diff detection may incorrectly skip mvn install." >&2; \
        echo "Running mvn install to avoid a false skip." >&2; \
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
            echo "No parent commit found for default-branch HEAD. Running mvn install..." >&2; \
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
        echo "No origin compare ref found. Running mvn install..." >&2; \
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
    # Check untracked files.
    if [ -n "$(git -C /src ls-files --others --exclude-standard -- $PATHS_INCLUDE || true)" ]; then \
        HAS_CHANGES=1; \
    fi; \
    if [ "$HAS_CHANGES" -eq 0 ]; then \
        echo "No changes detected ($COMPARE_REF). Skipping mvn install..." >&2; \
        exit 0; \
    fi; \
    echo "Changes detected. Running mvn install..." >&2; \
    pack

###################################
# Target: maven
###################################
FROM scratch AS maven
COPY --from=pre-maven /output /
