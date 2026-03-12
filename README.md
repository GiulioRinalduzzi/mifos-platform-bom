# mifos-platform-bom

> Bill of Materials for the Mifos Payment Hub EE (PH-EE) connector ecosystem.

A single Gradle `java-platform` project that pins every shared dependency version used across the PH-EE connector suite. Connectors import this BOM once and get consistent, Jakarta EE 10-compatible versions of Spring Boot 3.4, Apache Camel 4, Zeebe/Camunda 8, and 30+ supporting libraries — with no version declarations of their own.

## Why a BOM?

The Payment Hub connector ecosystem currently has **42+ connector repositories**, each declaring its own dependency versions independently. This leads to:

- Version drift between connectors (e.g., Spring Boot 3.1 in one repo, 3.2 in another)
- `javax.*` vs `jakarta.*` classpath conflicts from incompatible transitive dependencies
- Security patch rollouts requiring PRs to dozens of repositories
- No single source of truth for the platform stack

`mifos-platform-bom` solves this by providing a single import that governs the entire stack. A platform-wide upgrade (e.g., Spring Boot 3.4 → 3.5) becomes a one-line change here, and all connectors inherit it on their next build.

## Stack

| Layer | Artifact | Version |
|---|---|---|
| Spring Boot | `org.springframework.boot:spring-boot-dependencies` | 3.4.4 |
| Apache Camel | `org.apache.camel.springboot:camel-spring-boot-bom` | 4.4.1 |
| Zeebe / Camunda | `io.camunda:spring-zeebe-starter` | 8.4.1 |
| Jakarta Validation | `jakarta.validation:jakarta.validation-api` | 3.0.2 |
| Jakarta Servlet | `jakarta.servlet:jakarta.servlet-api` | 6.0.0 |
| Hibernate Validator | `org.hibernate.validator:hibernate-validator` | 8.0.1.Final |
| JAXB Runtime | `org.glassfish.jaxb:jaxb-runtime` | 4.0.5 |
| Jackson | `com.fasterxml.jackson.core:jackson-databind` | 2.18.2 |
| Apache CXF | `org.apache.cxf:cxf-core` | 4.0.4 |
| SpringDoc OpenAPI | `org.springdoc:springdoc-openapi-starter-webmvc-ui` | 2.6.0 |
| Flyway | `org.flywaydb:flyway-core` | 10.18.0 |
| Lombok | `org.projectlombok:lombok` | 1.18.36 |

See [`gradle.properties`](gradle.properties) for the full version catalogue.

## Usage

### 1. Add the BOM to a connector

In the connector's `build.gradle`, import the BOM as an `enforcedPlatform`:

```groovy
repositories {
    mavenLocal()
    maven { url = uri('https://maven.pkg.github.com/openMF/mifos-platform-bom') }
    mavenCentral()
}

dependencies {
    implementation enforcedPlatform('org.mifos.platform:mifos-platform-bom:1.0.0-SNAPSHOT')

    // No version needed — governed by the BOM
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.apache.camel.springboot:camel-spring-boot-starter'
    implementation 'io.camunda:spring-zeebe-starter'
}
```

`enforcedPlatform` (vs `platform`) ensures that BOM constraints override any conflicting transitive dependency declarations. This is the correct choice for enforcing Jakarta EE 10 purity — it prevents legacy `javax.*` artifacts from being pulled in by third-party libraries.

### 2. Remove redundant version declarations

Once the BOM is imported, remove explicit versions from any dependency already listed in [`build.gradle`](build.gradle). Keeping explicit versions defeats the BOM.

## Project structure

```
mifos-platform-bom/
├── build.gradle          # BOM definition — all constraints live here
├── gradle.properties     # Version catalogue — the only file to edit on upgrades
├── settings.gradle       # Includes bom-verification subproject
├── renovate.json         # Renovate bot config for automated dependency PRs
└── bom-verification/
    └── build.gradle      # Verification subproject — resolves all deps and runs purity check
```

### bom-verification

The `bom-verification` subproject has no source code. It imports this BOM and resolves a representative sample of every declared dependency against the real Maven repositories. It also runs a **classpath purity check** that fails the build if any `javax.*` artifact is detected on the classpath.

This means problems are caught before the BOM is published — not after connectors have already consumed a broken version.

```bash
# Run the verification locally
./gradlew :bom-verification:verifyBom
```

## Publishing

The BOM is published to **GitHub Packages** via the `maven-publish` plugin.

```bash
# Publish to Maven Local (development)
./gradlew publishToMavenLocal

# Publish to GitHub Packages (CI)
./gradlew publish -Pgpr.user=<github-user> -Pgpr.key=<github-token>
```

CI publishes a new SNAPSHOT on every merge to `main`. Connectors referencing `1.0.0-SNAPSHOT` will pick up the latest version automatically on their next build refresh.

## Upgrading the platform stack

All version pins live in [`gradle.properties`](gradle.properties). To upgrade the entire platform:

1. Update the relevant version property (e.g., `springBootVersion=3.5.0`)
2. Run `./gradlew :bom-verification:verifyBom` locally to confirm all artifacts resolve
3. Open a PR — the CI Verify workflow runs the same check and the purity check
4. Merge; the new SNAPSHOT is published automatically

Renovate opens automated PRs for patch and minor version bumps every weekend, following the rules in [`renovate.json`](renovate.json). Spring Boot, Camel, Zeebe, and Jakarta EE upgrades are grouped and require manual review before merge.

## License

[Mozilla Public License 2.0](LICENSE)
