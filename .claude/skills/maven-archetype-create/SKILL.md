---
name: maven-archetype-create
description: Author a new Maven archetype (a project template that `mvn archetype:generate` can scaffold from). Use this skill whenever the user asks to create/build/author a Maven archetype, package an existing project as a reusable template, turn a starter project into a Maven archetype, write `archetype-metadata.xml`, lay out `archetype-resources/`, or install and test a custom archetype locally — even if they don't say the word "archetype" explicitly (e.g. "make our Spring starter into a `mvn archetype:generate`-able template").
---

# Creating a Maven Archetype

A Maven archetype is itself a Maven project whose `packaging` is `maven-archetype`. When installed, users can run `mvn archetype:generate` against it to scaffold new projects from the template files it ships.

This skill covers the manual construction path from the official guide (https://maven.apache.org/guides/mini/guide-creating-archetypes.html). Prefer the manual path when the user already has a concrete project they want to templatize — it gives you full control over filtering, packaging, and required properties.

## When to bootstrap vs. build from scratch

If the user is starting cold and has no template content yet, suggest bootstrapping with the `maven-archetype-archetype`:

```bash
mvn archetype:generate \
  -DgroupId=<their.group> \
  -DartifactId=<their-archetype-id> \
  -DarchetypeGroupId=org.apache.maven.archetypes \
  -DarchetypeArtifactId=maven-archetype-archetype
```

Then customize the generated `archetype-resources/` and `archetype-metadata.xml`. This saves typing but is otherwise identical to the structure below.

If the user already has a working sample project they want to convert, build the archetype layout by hand (sections below) — it's clearer than mutating a bootstrapped one.

## Required directory structure

```
<archetype-root>/
├── pom.xml                                            # the archetype's own pom
└── src/main/resources/
    ├── META-INF/maven/archetype-metadata.xml          # descriptor: what to include, filtering rules
    └── archetype-resources/                           # the template payload (becomes the generated project)
        ├── pom.xml                                    # generated project's pom — uses ${groupId} etc.
        └── src/
            ├── main/java/App.java                     # example source
            └── test/java/AppTest.java
```

Important constraint from the guide: **empty directories cannot be created** by the archetype. Every directory in `archetype-resources/` must contain at least one file. If the user needs an "empty" `resources/` dir in generated projects, drop in a `.gitkeep` or a placeholder file.

## The archetype's own pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <groupId>my.groupId</groupId>
  <artifactId>my-archetype-id</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>maven-archetype</packaging>

  <build>
    <extensions>
      <extension>
        <groupId>org.apache.maven.archetype</groupId>
        <artifactId>archetype-packaging</artifactId>
        <version>3.4.1</version>
      </extension>
    </extensions>
  </build>
</project>
```

The `maven-archetype` packaging and the `archetype-packaging` build extension are both required — without them the archetype JAR will not be built correctly, even if everything else looks right.

## archetype-metadata.xml

This descriptor controls what gets copied into generated projects and how variables are substituted. Minimum useful form:

```xml
<archetype-descriptor
    xmlns="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.2.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/plugins/maven-archetype-plugin/archetype-descriptor/1.2.0
                        https://maven.apache.org/xsd/archetype-descriptor-1.2.0.xsd"
    name="my-archetype-id">

  <requiredProperties>
    <!-- Optional. groupId/artifactId/version/package are always available; add custom ones here. -->
    <requiredProperty key="serviceName">
      <defaultValue>my-service</defaultValue>
    </requiredProperty>
  </requiredProperties>

  <fileSets>
    <fileSet filtered="true" packaged="true">
      <directory>src/main/java</directory>
      <includes>
        <include>**/*.java</include>
      </includes>
    </fileSet>
    <fileSet filtered="true" packaged="true">
      <directory>src/test/java</directory>
      <includes>
        <include>**/*.java</include>
      </includes>
    </fileSet>
    <fileSet filtered="true">
      <directory></directory>
      <includes>
        <include>pom.xml</include>
      </includes>
    </fileSet>
    <fileSet>
      <directory>src/main/resources</directory>
      <includes>
        <include>**/*</include>
      </includes>
    </fileSet>
  </fileSets>
</archetype-descriptor>
```

Match the descriptor `name` attribute to the archetype `artifactId` — the guide calls this out, and it avoids confusing diagnostics later.

### Two attributes you will reach for constantly

- `filtered="true"` — run the file's contents through Velocity so `${groupId}`, `${artifactId}`, `${version}`, `${package}`, and any `requiredProperty` you defined are interpolated. Use on every text file that should be templated (Java sources, `pom.xml`, YAML, `.properties`). **Do not** filter binary files — it will corrupt them.
- `packaged="true"` — applies only to Java-style sourceset filesets. When set, files under the fileset directory are relocated under the user-supplied `${package}` (default = `${groupId}`) at generation time. Use for `src/main/java` and `src/test/java`. Don't use for resource directories — you almost never want `application.yml` to land under `com/acme/foo/`.

### Multi-module archetypes

If the template scaffolds a multi-module project, use `<modules>` inside `<archetype-descriptor>`, one entry per child, each with its own `<fileSets>`. Set `partial="true"` on the descriptor's root element only when the archetype is meant to be merged into an existing project rather than creating a standalone one.

## archetype-resources content

Inside files marked `filtered="true"`, reference variables with Velocity syntax: `${groupId}`, `${artifactId}`, `${version}`, `${package}`, plus any custom required properties.

Common pitfall: a literal `$` in a filtered file (e.g. shell scripts, regexes) must be escaped as `${D}` with a `$D` property defined, or escaped per Velocity rules (`\$`). When in doubt, mark the file unfiltered and substitute by hand.

The generated `pom.xml` lives at `archetype-resources/pom.xml`. It will look like a normal pom but with template variables:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <groupId>${groupId}</groupId>
  <artifactId>${artifactId}</artifactId>
  <version>${version}</version>
  <packaging>jar</packaging>

  <!-- ... -->
</project>
```

Java sources under `archetype-resources/src/main/java/` should declare their package using the template variable so they relocate correctly when `packaged="true"`:

```java
package ${package};

public class App {
    public static void main(String[] args) {
        System.out.println("Hello from ${artifactId}");
    }
}
```

Note: do **not** create the `com/acme/foo/` directory tree yourself under `archetype-resources/src/main/java/` — the archetype plugin creates it from the user-supplied `${package}` when `packaged="true"`. Place the bare `App.java` directly in `src/main/java/`.

## Install and test

```bash
# In the archetype project root:
mvn install

# Then, from any empty directory, scaffold a project from it:
mvn archetype:generate \
  -DarchetypeCatalog=local \
  -DarchetypeGroupId=my.groupId \
  -DarchetypeArtifactId=my-archetype-id \
  -DarchetypeVersion=1.0-SNAPSHOT \
  -DgroupId=com.acme \
  -DartifactId=demo-service \
  -Dversion=0.1.0-SNAPSHOT \
  -Dpackage=com.acme.demo
```

`-DarchetypeCatalog=local` tells the plugin to look in the local repo instead of remote catalogs. Always pass `-DarchetypeVersion=...` explicitly — without it the plugin resolves `RELEASE`, which fails for `-SNAPSHOT` archetypes with a confusing "version:RELEASE not found" error.

For a non-interactive scaffold (useful in scripts and CI), add `-B` (batch mode) and ensure every required property has a default in the descriptor or is supplied on the command line.

## Iterating

When changing the archetype, the loop is: edit → `mvn install` → `mvn archetype:generate ...` in a fresh empty directory → inspect the generated tree → repeat. A common mistake is regenerating into the same directory and getting confused by stale files; always scaffold into a clean dir or `rm -rf` first.

## Quick troubleshooting

- **"could not resolve archetype"** — usually a missing `-DarchetypeVersion` or you forgot to `mvn install` after editing. Check `~/.m2/repository/<groupId>/<archetype-artifactId>/`.
- **Generated Java files land in the wrong package** — verify the fileset has `packaged="true"` and the source files use `package ${package};` (not a hardcoded package).
- **Variables show up literally as `${groupId}` in generated files** — the fileset is missing `filtered="true"`.
- **Binary files corrupted in generated project** — the fileset is wrongly `filtered="true"`; split binary assets into their own unfiltered fileset.
- **An expected empty directory is missing** — by design; add a placeholder file.
