# rest-build-parent

Parent POM and BOM for shared Maven configuration.

## Modules

- `rest-build-parent` — parent POM (Java version, encoding, repositories, plugin management, distribution).
- `rest-bom` — Bill of Materials with managed dependency versions.

## Maven settings

To bootstrap the parent itself, add the GitHub Packages repository and matching server credentials to `~/.m2/settings.xml`. The PAT needs `read:packages` (plus `write:packages` to deploy):

```xml
<settings>
  <profiles>
    <profile>
      <id>restship</id>
      <repositories>
        <repository>
          <id>restship</id>
          <url>https://maven.pkg.github.com/luizroddev/rest-build-parent</url>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>
  <activeProfiles><activeProfile>restship</activeProfile></activeProfiles>
  <servers>
    <server>
      <id>restship</id>
      <username>YOUR_GITHUB_USERNAME</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

Once the parent is resolved, child projects inherit the full `<repositories>` list from `pom.xml`.

## Usage

Inherit the parent:

```xml
<parent>
    <groupId>com.restship</groupId>
    <artifactId>rest-build-parent</artifactId>
    <version>1.0.0</version>
</parent>
```

Or import the BOM:

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

Override the Java version (default `22`) per child:

```xml
<properties><java.version>17</java.version></properties>
```

Override the deploy target per child:

```xml
<properties>
    <github.owner>your-org</github.owner>
    <github.repo>your-repo</github.repo>
</properties>
```

## CI workflows

Reusable workflows under `.github/workflows/`. All require a `PACKAGES_TOKEN` secret (GitHub PAT).

| Workflow | Purpose | PAT scopes |
|---|---|---|
| `ci-build.yml` | Compile and test | `read:packages` |
| `ci-deploy-snapshot.yml` | Deploy `-SNAPSHOT` | `read:packages`, `write:packages` |
| `ci-release.yml` | Strip `-SNAPSHOT`, deploy, tag, bump | `read:packages`, `write:packages`, `repo` |

Caller example:

```yaml
jobs:
  build:
    uses: <owner>/rest-build-parent/.github/workflows/ci-build.yml@master
    with:
      java-version: '22'        # optional
      maven-args: ''            # optional
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
```

The release workflow accepts optional `release-version` and `next-snapshot-version` inputs; otherwise it strips `-SNAPSHOT` and increments the patch.
