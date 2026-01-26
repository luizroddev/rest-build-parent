# Minerest Build Parent

Central parent POM and Bill of Materials (BOM) for the Minerest server network. This repository is the **single source of truth** for dependency versions, plugin configuration, and deployment settings shared across all Minerest modules and plugins.

## What This Project Contains

```
rest-build-parent/          <-- Parent POM (this project)
  rest-bom/                 <-- Bill of Materials (BOM)
  .github/workflows/        <-- Reusable CI/CD workflows
```

### `rest-build-parent` (Parent POM)

The root POM that every Minerest module inherits from. It provides:

- **Java version** &mdash; Defaults to Java 22. Child POMs can override to Java 17 with `<java.version>17</java.version>`.
- **Encoding** &mdash; UTF-8 source encoding for all projects.
- **Repository declarations** &mdash; Pre-configured access to GitHub Packages (Restship), JitPack, Aikar, PaperMC, SpigotMC, and Sonatype Snapshots. Any child project automatically resolves dependencies from these sources.
- **Distribution management** &mdash; All artifacts deploy to [GitHub Packages](https://github.com/luizroddev/rest-build-parent/packages) under the `luizroddev` owner.
- **Plugin management** &mdash; Locked versions of `maven-compiler-plugin`, `maven-shade-plugin`, `maven-source-plugin`, `maven-deploy-plugin`, and `maven-release-plugin`. Child POMs can reference these plugins without specifying versions.

### `rest-bom` (Bill of Materials)

A `<dependencyManagement>` POM that locks versions for **all** third-party and internal dependencies. When a child project imports the BOM, it can declare dependencies **without specifying a version** &mdash; the BOM provides the correct one.

#### Managed Dependencies

| Category | Artifacts |
|---|---|
| **Internal Modules** | `restserver`, `logger`, `concurrency`, `network`, `minecore` |
| **Database** | jOOQ, HikariCP, Hibernate ORM, PostgreSQL, MySQL, MariaDB, resultset-mapper |
| **Serialization** | Jackson, Gson |
| **Redis** | Lettuce |
| **Minecraft APIs** | Adventure (API + Bukkit platform), ACF-Paper, LuckPerms, Velocity API |
| **Utilities** | Lombok, Reflections, ANTLR4, Configurate-HOCON, AsciiTable |
| **Test** | Mockito (core + inline) |

## How It Impacts Other Projects

Every Minerest module and plugin depends on this project. When you change something here, it affects **all downstream repositories** on their next build:

1. **Bumping a dependency version** in the parent properties (e.g., `<jackson.version>`) will update that dependency for every project that imports the BOM.
2. **Changing plugin configuration** (e.g., compiler source/target) will change how every inheriting project is compiled.
3. **Adding/removing repositories** will affect artifact resolution for every child.
4. **Changing `distributionManagement`** will change where every child project publishes its artifacts.

> **Important**: Treat changes to this project with care. A broken parent POM or BOM will break builds across the entire Minerest ecosystem.

## How to Use

### 1. Configure Maven Settings

Add the GitHub Packages server credentials to your `~/.m2/settings.xml`:

```xml
<settings>
  <activeProfiles>
    <activeProfile>restship</activeProfile>
  </activeProfiles>

  <profiles>
    <profile>
      <id>restship</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://repo1.maven.org/maven2</url>
        </repository>
        <repository>
          <id>restship</id>
          <url>https://maven.pkg.github.com/romulofebasi/restship</url>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <servers>
    <server>
      <id>restship</id>
      <username>YOUR_GITHUB_USERNAME</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

The `GITHUB_TOKEN` must have `read:packages` scope (add `write:packages` if you need to deploy).

### 2. Inherit the Parent POM

In your module's `pom.xml`, set `rest-build-parent` as the parent:

```xml
<parent>
    <groupId>com.restship</groupId>
    <artifactId>rest-build-parent</artifactId>
    <version>1.0.0</version>
</parent>
```

This gives your project all inherited properties, repositories, plugin management, and distribution settings.

### 3. Import the BOM

If your project does **not** directly inherit the parent (e.g., it has its own parent), import the BOM instead:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.restship</groupId>
            <artifactId>rest-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 4. Declare Dependencies Without Versions

Once the parent is inherited or the BOM is imported, declare dependencies without a `<version>` tag:

```xml
<dependencies>
    <dependency>
        <groupId>com.restship.module</groupId>
        <artifactId>minecore</artifactId>
        <!-- version provided by the BOM -->
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <!-- version provided by the BOM -->
    </dependency>
</dependencies>
```

### 5. Use Pre-configured Plugins

The parent provides `pluginManagement` entries so child POMs can use plugins without declaring versions:

```xml
<build>
    <plugins>
        <!-- Version and default config inherited from parent -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <!-- Add project-specific configuration here -->
        </plugin>
    </plugins>
</build>
```

### 6. Override Java Version

Projects that need Java 17 instead of the default 22:

```xml
<properties>
    <java.version>17</java.version>
</properties>
```

## CI/CD Workflows

This repository ships three reusable GitHub Actions workflows under `.github/workflows/`. Other repos call them with `workflow_call`.

### `ci-build.yml`

Compiles and runs tests. Caller example:

```yaml
jobs:
  build:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-build.yml@master
    with:
      java-version: '22'
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

### `ci-deploy-snapshot.yml`

Deploys SNAPSHOT versions to GitHub Packages. Enforces that the project version ends with `-SNAPSHOT`; fails otherwise.

```yaml
jobs:
  deploy:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-deploy-snapshot.yml@master
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

### `ci-release.yml`

Full release workflow: strips `-SNAPSHOT`, deploys, creates a Git tag (`v1.0.0`), and bumps to the next SNAPSHOT version. Optionally accepts `release-version` and `next-snapshot-version` inputs.

```yaml
jobs:
  release:
    uses: luizroddev/rest-build-parent/.github/workflows/ci-release.yml@master
    with:
      release-version: '1.2.0'          # optional
      next-snapshot-version: '1.3.0-SNAPSHOT'  # optional
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

The `PACKAGES_TOKEN` secret requires `read:packages`, `write:packages`, and `repo` scopes for the release workflow.

## Deploying This Project

```bash
# Deploy parent + BOM to GitHub Packages
mvn deploy

# Deploy only the BOM (if parent is already published)
mvn deploy -rf :rest-bom
```

> GitHub Packages does not allow overwriting a released (non-SNAPSHOT) version. To re-deploy, you must bump the version first.

## Updating a Dependency Version

1. Edit the version property in `pom.xml` (e.g., change `<jackson.version>2.17.0</jackson.version>` to `2.18.0`).
2. Run `mvn deploy` to publish the updated parent and BOM.
3. Downstream projects will pick up the new version on their next build (or force it with `mvn -U clean install`).
