# Official Container Image for Cruise Control

This proposal adds a `Dockerfile` to the Cruise Control repository so users can build and run Cruise Control as a container.
It also lays out a path toward publishing an official image as part of the release process.

## Current situation

Cruise Control is distributed as source code and as Maven artifacts (JARs).
To run it, users clone the repository, build it with Gradle (`./gradlew jar copyDependantLibs`), edit `config/cruisecontrol.properties`, and start it with `kafka-cruise-control-start.sh`.
This requires a JDK and a checkout of the repository on every host that runs Cruise Control.

The project does not provide a `Dockerfile` or a container image.
Users who run Cruise Control in containers (Docker Compose, Kubernetes, etc.) have to write and maintain their own image.
Several downstream images exist, for example the one in [Adobe's fork](https://github.com/adobe/cruise-control/tree/master/docker) or the Kafka image Strimzi builds with Cruise Control bundled for its operator.
None of them is maintained by the Cruise Control project, and each one makes its own choices about base image, Java version, file layout and configuration.

An official image has been requested in [#2263](https://github.com/cruise-control-for-kafka/cruise-control/issues/2263), and [#2348](https://github.com/cruise-control-for-kafka/cruise-control/pull/2348) proposes a first `Dockerfile`.
Other users have also asked for it in the comments on that pull request.

## Motivation

- **Lower barrier to entry**: Trying Cruise Control against a Kafka cluster should take a `docker run`, not a JDK install and a Gradle build.
- **Easier development and testing**: Contributors and users can start Cruise Control next to Kafka in a Docker Compose setup to test changes and reproduce issues.
- **Production deployments**: Many organizations run Kafka tooling on container platforms.
  An image maintained by the project gives them a known starting point, so each team does not have to build the same thing.
- **Faster security fixes for container users**: The [first release](./001-first-release.md) focuses on CVE fixes.
  Users of third-party images only get those fixes once the image maintainer rebuilds.
  An official image built as part of each release delivers them at the same time as the JARs.

## Proposal

The work is split into two phases.
Phase 1 is useful on its own and does not depend on the release and registry decisions that Phase 2 needs.

### Phase 1: `Dockerfile` in the repository

Add a multi-stage `Dockerfile` and a `.dockerignore` to the root of the `cruise-control` repository, starting from the one in [#2348](https://github.com/cruise-control-for-kafka/cruise-control/pull/2348):

```dockerfile
# Build Cruise Control in its own stage
FROM amazoncorretto:21-alpine-jdk AS build
WORKDIR /workspace
COPY . .
RUN ./gradlew clean jar copyDependantLibs --warning-mode all

# Fetch the jars, configs, and the startup script and run Cruise Control
FROM amazoncorretto:21-alpine AS runtime
RUN apk add --no-cache bash
WORKDIR /cc
COPY --from=build /workspace/cruise-control/build/ /cc/cruise-control/build/
COPY --from=build /workspace/config/ /cc/config/
COPY --from=build /workspace/kafka-cruise-control-start.sh /cc/
RUN chmod +x kafka-cruise-control-start.sh
EXPOSE 9090
CMD ["./kafka-cruise-control-start.sh", "config/cruisecontrol.properties", "9090"]
```

Design points:

- **Build from source inside the image**: The build stage runs the same Gradle tasks a user would run locally, so Docker is the only prerequisite.
  No JDK or Gradle install is needed on the host.
- **Small runtime image**: The runtime stage uses a JRE-only Alpine image and copies in only the built JARs, dependant libraries, default config and start script.
  `bash` is added because `kafka-cruise-control-start.sh` requires it.
- **Reuse the existing start script**: The container starts Cruise Control the same way a non-containerized install does, so there is one startup path to maintain.
  The same environment variables (`KAFKA_HEAP_OPTS`, `KAFKA_JVM_PERFORMANCE_OPTS`, `KAFKA_OPTS`, `JMX_PORT`, etc.) work in the container.
  In the foreground mode, the script `exec`s the JVM, so the JVM runs as PID 1 and receives `SIGTERM` directly when the container stops.
- **`.git` stays in the build context**: The Gradle build reads `.git` to derive the project version and commit id, so `.dockerignore` does not exclude it.
- **Java runtime version**: The project compiles to Java 17 bytecode, but CI only runs the tests on Java 17.
  The image should run on a Java version that CI tests.
  Either the image uses a Java 17 base, or Java 21 is added to the CI matrix before an image is published (see [Open questions](#open-questions)).

#### Runtime layout and configuration

The image establishes a layout that users depend on once they mount files into it:

| Path                                   | Purpose                                                                                       |
|:---------------------------------------|:----------------------------------------------------------------------------------------------|
| `/cc/config/cruisecontrol.properties`  | Main configuration. The default points `bootstrap.servers` at `localhost:9092`, which does not work in a container, so users must override it. |
| `/cc/config/`                          | Capacity files, `log4j2.properties`, and the optional `cruise_control_jaas.conf` that the start script picks up. |
| `/cc/fileStore/`                       | Default location of `failed.brokers.file.path`. Mount a volume here so the failed-broker list survives container restarts. |
| `/cc/logs/`                            | Default `LOG_DIR`. The default `log4j2.properties` writes rolling log files here in addition to the console. |
| `9090`                                 | REST API port.                                                                                |

Users configure the container by mounting files over these paths.
In most cases only `bootstrap.servers` (plus any security settings) and the capacity file need to change:

```sh
docker build -t cruise-control .
docker run -p 9090:9090 \
  -v $(pwd)/cruisecontrol.properties:/cc/config/cruisecontrol.properties \
  -v $(pwd)/capacity.json:/cc/config/capacityJBOD.json \
  -v cc-filestore:/cc/fileStore \
  cruise-control
```

Mounting a whole directory over `/cc/config` is also possible, but it then has to contain every file the configuration refers to, including `log4j2.properties`.

Phase 1 also adds a short section to the README or wiki that describes how to build and run the image, and the paths above.

### Phase 2: Publish an official image

Once the `Dockerfile` is in place, add a GitHub Actions workflow that builds the image and pushes it to a public registry as part of each release.

- **Registry**: GitHub Container Registry (`ghcr.io/cruise-control-for-kafka/cruise-control`).
  The workflow authenticates with its `GITHUB_TOKEN`, so no extra accounts or secrets are needed, and the image lives next to the source.
- **Tags**: One tag per release version (e.g. `3.1.0`) and `latest` pointing at the newest release.
  Optionally, a `main` tag built from the `main` branch for users who want to test unreleased changes.
  Release tags are never overwritten.
- **Architectures**: `linux/amd64` and `linux/arm64`, built with `docker buildx`, since both are common on container platforms and developer laptops.
- **Release trigger**: The image is built from the same commit as the release JARs, in the same workflow or one triggered by the release.
- **CI check**: Pull requests that touch the `Dockerfile`, `.dockerignore`, the start script or the build configuration build the image without pushing it, so the image does not break unnoticed.

### Hardening before publishing

Before an image is published as an official artifact (Phase 2), it should:

- run as a non-root user that owns `/cc/logs` and `/cc/fileStore`,
- pin base images by version (and ideally by digest), so builds are reproducible and base image updates are explicit changes,
- be scanned for CVEs in CI, in line with the CVE focus of the [first release](./001-first-release.md),
- log to the console only by default, which is the convention for containers, instead of also writing rolling files inside the container's writable layer,
- carry the standard OCI labels (`org.opencontainers.image.source`, `version`, `revision`, `licenses`).

These can be added to [#2348](https://github.com/cruise-control-for-kafka/cruise-control/pull/2348) or in follow-up pull requests before Phase 2 ships.

### Open questions

- **Java runtime**: Use a Java 17 base image to match current CI, or add Java 21 to the CI matrix and keep the Java 21 base?
- **Base image**: `amazoncorretto` (Alpine) as in #2348, or another distribution such as `eclipse-temurin`?
  CI currently tests with the Microsoft and Temurin JDKs.
- **Tag scheme**: Are floating minor tags (e.g. `3.1`) needed in addition to full version tags and `latest`?
- **Release timing**: Should Phase 2 be part of the release process right after the [first release](./001-first-release.md), or wait until the release process is finalized under the Linux Foundation?

### Out of scope

- **Helm chart / Kubernetes manifests**: Useful as a follow-up, but a separate proposal.
  Operators such as Strimzi already manage Cruise Control on Kubernetes.
- **Cruise Control UI**: The UI lives in a separate repository outside the `cruise-control-for-kafka` organization and is not bundled in the image.
  Users who want it can mount it at the path set by `webserver.ui.diskpath`.
- **Metrics reporter**: The `cruise-control-metrics-reporter` JAR runs inside the Kafka brokers, not in Cruise Control, so it does not get its own image.

## Affected/not affected projects

- **`cruise-control`**: Affected.
  Adds `Dockerfile` and `.dockerignore` (Phase 1), a GitHub Actions workflow to build and publish the image (Phase 2), and documentation.
- **`cruise-control-proposals`**: Not affected beyond this proposal.

The existing build and published JARs do not change.

## Compatibility

The change is additive.
It does not modify the Gradle build, the start script, the configuration format or the published JARs, so existing users are not affected.

Once an image is published, these details become part of its public contract:

- the image name and tag scheme,
- the install path (`/cc`) and the paths users mount over (`/cc/config/cruisecontrol.properties`, `/cc/fileStore`, `/cc/logs`),
- the exposed port (`9090`),
- the user the container runs as, since it decides who can write to mounted volumes.

Changes to these should be called out in the release notes.

The image also fixes the Java runtime version through its base image.
Moving the image to a newer Java runtime is a change to the image, not to the Java versions Cruise Control supports, and should also be called out in the release notes.

## Rejected alternatives

### Copying prebuilt JARs from the host into the image

The `Dockerfile` could copy JARs built on the host instead of building them in a build stage.
This was rejected because it requires a local JDK and Gradle build before `docker build`, and the result depends on the host environment.
Building inside the image makes `docker build` self-contained.

### Building the image with a Gradle plugin (e.g. Jib)

Jib can build images directly from Gradle without a `Dockerfile`.
This was rejected because it adds a build plugin to maintain, does not reuse `kafka-cruise-control-start.sh`, and a `Dockerfile` is more familiar to most contributors and users.

### Configuring the container through environment variables

Some images (e.g. the official Apache Kafka image) translate environment variables into configuration properties.
This was rejected for now because it adds a translation layer to maintain and test, while mounting a properties file works with every container platform and keeps the configuration format the same as for non-containerized installs.
It can be added later without breaking the file-based approach.

### Pointing users to existing third-party images

The project could document existing community images instead of providing its own.
This was rejected because those images are not maintained by the project, may lag behind releases and CVE fixes, and give users no guarantee about how they were built.

### Publishing to Docker Hub

Docker Hub is the best known registry, but publishing there requires a separate organization account and credentials stored as secrets, and anonymous pulls are rate limited.
GHCR is preferred for Phase 2 because it works with GitHub Actions without extra secrets.
Mirroring to Docker Hub later remains possible if there is demand.

### Bundling the Cruise Control UI

Bundling the UI would make the image more convenient for some users, but it would tie Cruise Control releases to a project outside the organization and increase the image size.
This can be revisited if the UI moves into the organization.
