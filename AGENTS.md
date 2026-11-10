# rest-build-parent

Parent POM (`pom.xml`) and BOM (`rest-bom/`) for the Minerest Maven repositories, plus the
reusable CI workflows they call. No Java code, no tests.

## Blast radius

- Every merge to `master` publishes `1.0.0-SNAPSHOT` (`deploy-snapshot.yml`). Consumers resolve
  it with `updatePolicy=always`, so their next local or CI build picks it up. Treat a merge as a
  release to minecore, network, the proxy and every plugin.
- `ci-build.yml`, `ci-deploy-snapshot.yml` and `ci-release.yml` are called by the other
  repositories at `@master`. Keep their inputs and secrets backward compatible.
- Shared versions are properties in the root `pom.xml` (`minecore.version`, `network.version`,
  `restserver.version`, ...) that `rest-bom` references. Consumers often override them (plugins
  set their own `minecore.version`; minecore and the proxy pin `network` directly), so check the
  consumer's pom before assuming a bump here reaches it.

## Build

- `mvn -B clean install` installs the parent and the BOM into `~/.m2` for local consumers.
- Resolving anything needs GitHub Packages credentials in `~/.m2/settings.xml`
  (README.md, "Maven settings").

## Git

Trunk is `master`, not `stable`. Commits, branches and PRs are in English; the rest of the
workflow is in the `git` skill.
