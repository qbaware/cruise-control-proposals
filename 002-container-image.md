# Official Container Image for Cruise Control

This proposal adds a `Dockerfile` to the Cruise Control repository and later publishes an official image with each release.

## Current situation

To run Cruise Control, users clone the repository, build it with Gradle, edit `config/cruisecontrol.properties` and run `kafka-cruise-control-start.sh`.
This needs a JDK and a repository checkout on every host.

The project provides no `Dockerfile` or image, so container users build their own.
Downstream images exist (e.g. [Adobe's fork](https://github.com/adobe/cruise-control/tree/master/docker), Strimzi's Kafka image), but the project maintains none of them.

An official image was requested in [#2263](https://github.com/cruise-control-for-kafka/cruise-control/issues/2263), and [#2348](https://github.com/cruise-control-for-kafka/cruise-control/pull/2348) proposes a first `Dockerfile`.

## Motivation

- **Easier to try**: Running Cruise Control should take a `docker run`, not a JDK and a Gradle build.
- **Easier to develop and test**: Cruise Control can run next to Kafka in Docker Compose.
- **Production use**: Teams on container platforms get a known-good image instead of each maintaining their own.
- **Security fixes**: Users of an official image get CVE fixes at release time, not when a third party rebuilds.

## Proposal

### Phase 1: `Dockerfile` in the repository

Add a multi-stage `Dockerfile` and a `.dockerignore`, based on [#2348](https://github.com/cruise-control-for-kafka/cruise-control/pull/2348):

```dockerfile
FROM amazoncorretto:21-alpine-jdk AS build
WORKDIR /workspace
COPY . .
RUN ./gradlew clean jar copyDependantLibs --warning-mode all

FROM amazoncorretto:21-alpine AS runtime
RUN apk add --no-cache bash
WORKDIR /cc
COPY --from=build /workspace/cruise-control/build/ /cc/cruise-control/build/
COPY --from=build /workspace/kafka-cruise-control-start.sh /cc/
COPY docker/log4j2.properties /cc/log4j2.properties
ENV KAFKA_LOG4J_OPTS="-Dlog4j.configurationFile=file:/cc/log4j2.properties"
EXPOSE 9090
CMD ["./kafka-cruise-control-start.sh", "config/cruisecontrol.properties", "9090"]
```

- **Self-contained build**: Gradle runs inside the build stage, so Docker is the only prerequisite.
  `.git` stays in the build context because the build derives the version and commit id from it.
- **Existing start script**: The image starts Cruise Control the same way as a regular install, so `KAFKA_HEAP_OPTS`, `KAFKA_OPTS`, `JMX_PORT`, etc. work unchanged.
  The script `exec`s the JVM, so it receives `SIGTERM` when the container stops.
- **Console logging**: The image ships its own `log4j2.properties` that logs to the console only, outside the user's config directory.

#### Configuration

The image contains no Cruise Control configuration.
Users must provide their own, and can use environment variables on top of it.

**Config files (required)**: Mount a config directory at `/cc/config` with:

- `cruisecontrol.properties`,
- the capacity file (e.g. `capacityJBOD.json`) and any other files Cruise Control reads (e.g. `clusterConfigs.json`, `brokerSets.json`),
- optionally `cruise_control_jaas.conf`, which the start script picks up.

The easiest start is a copy of the repository's `config/` directory.
Its values cannot be used as-is: `bootstrap.servers` points at `localhost` and the capacity files describe example brokers.
Without a mounted config, the container fails at startup instead of running against the wrong cluster or wrong capacities.

**Environment variables (optional)**:

- Any property can reference an environment variable with `${env:NAME}`, e.g. `bootstrap.servers=${env:BOOTSTRAP_SERVERS}`.
  This keeps per-environment values and secrets out of the config files.
- The start script reads `KAFKA_HEAP_OPTS`, `KAFKA_JVM_PERFORMANCE_OPTS`, `KAFKA_OPTS`, `JMX_PORT` and `KAFKA_LOG4J_OPTS` for JVM settings.

```sh
docker run -p 9090:9090 \
  -v $(pwd)/my-config:/cc/config \
  -v cc-filestore:/cc/fileStore \
  -e BOOTSTRAP_SERVERS=kafka:9092 \
  -e KAFKA_HEAP_OPTS=-Xmx2G \
  cruise-control
```

`/cc/fileStore` is optional and keeps the failed-broker list across restarts.

The `cruise-control` repository documents this in `docker/README.md`.

### Phase 2: Publish an official image

Add a GitHub Actions workflow that builds and publishes the image with each release.

- **Registry**: GHCR (`ghcr.io/cruise-control-for-kafka/cruise-control`), using the workflow's `GITHUB_TOKEN`.
- **Tags**: The release version (e.g. `3.1.0`) and `latest`.
  Optionally `main` for unreleased builds.
- **Architectures**: `linux/amd64` and `linux/arm64`.
- **CI**: Pull requests that change the `Dockerfile` or the Gradle build also build the image, without pushing it.

### Before publishing

- Run as a non-root user that can write to `/cc/fileStore` and `/cc/logs` (the start script creates the latter).
- Pin base images by version and digest.
- Scan the image for CVEs in CI.
- Add OCI labels (`source`, `version`, `revision`, `licenses`).

### Open questions

- **Java version**: CI only tests on Java 17.
  Should the image use Java 17, or should CI add Java 21?
- **Base image**: `amazoncorretto` (Alpine) or `eclipse-temurin`?
- **Tags**: Are floating minor tags (e.g. `3.1`) needed?

### Out of scope

- **Helm chart / Kubernetes manifests**: A separate proposal.
- **Cruise Control UI**: Lives outside the organization; users can mount it at `webserver.ui.diskpath`.
- **Metrics reporter**: Runs inside the Kafka brokers, not in Cruise Control.

## Affected/not affected projects

- **`cruise-control`**: Adds `Dockerfile`, `.dockerignore`, `docker/log4j2.properties`, `docker/README.md` and a release workflow.
- **`cruise-control-proposals`**: Only this proposal.

## Compatibility

The change is additive: the Gradle build, start script, configuration format and JARs do not change.

Once published, these become part of the image's contract and changes to them go in the release notes:

- image name and tags,
- `/cc/config` and `/cc/fileStore`,
- port `9090`,
- container user,
- Java runtime version.

## Rejected alternatives

### Baking a default configuration into the image

The default `bootstrap.servers` and capacity file only work for a local, non-containerized setup.
A container started with them would either fail to connect or plan against example capacities.
Requiring a mounted config makes this explicit.

### Copying prebuilt JARs from the host

Requires a JDK and a Gradle build on the host, and makes the result depend on the host environment.

### Jib or another Gradle image plugin

Adds a build plugin to maintain and does not reuse the start script.

### Configuration through environment variables only

Mapping every property to its own environment variable (e.g. `CRUISE_CONTROL_BOOTSTRAP_SERVERS`) adds a translation layer to maintain.
`${env:NAME}` in the mounted properties already covers values that change per environment.

### Pointing users to third-party images

Not maintained by the project and may lag behind releases and CVE fixes.

### Docker Hub

Needs a separate account and secrets, and rate-limits anonymous pulls.
Mirroring there later remains possible.

### Bundling the UI

Ties releases to a project outside the organization and grows the image.
