# slsa-maven

Scaffold reproducible-build, signed-release CI for Maven
projects. Sibling project to
[slsa-autotools](https://github.com/anthropics/slsa-autotools);
same architecture, swap target tooling.

## What it does

We add GitHub workflows that build your project in a container
once, capture every byte the build touched (apt packages from
the system + Maven artifacts from Central), and bake those into
a new Dockerfile pinned by version and SHA256. From then on,
releases run against that Dockerfile fully offline. The release
process builds the jar / sources / javadoc / pom in two parallel
container instances and bit-to-bit compares them before SLSA
attestation and sigstore signing. The release is then created
as a draft for the maintainer to optionally GPG-sign before
publishing.

## Why

Most Maven releases are pushed manually from a maintainer's
machine via `mvn deploy`. Even when GPG-signed, those signatures
attest only "this maintainer approved this artifact" — not "this
artifact was built reproducibly from a specific commit." A
compromised maintainer machine forges everything. Moving the
build into GitHub Actions with a pinned, offline-capable builder
image gives anyone the means to verify bytes against the source
without trusting the maintainer.

In addition to SLSA provenance and sigstore signatures,
slsa-maven proves reproducibility on every release: each
artifact is built twice in independent containers and bit-
compared before signing. We don't just set
`project.build.outputTimestamp` and call it reproducible — we
verify it on every build.

## How

Two scaffolding scripts and three workflows do the work. The
scripts run once per project against a target repo; they
generate workflow files, they don't continuously manage
anything. The workflows then live in the target repo.

The flow:

1. `scripts/init-container` writes `scan.yml` and
   `build-container.yml` into the target repo on a topic branch.
2. `scan.yml` runs the project's real `mvn verify source:jar
   javadoc:jar` under strace, maps every file path the build
   touched to its source apt package, walks the populated
   `~/.m2/repository` and computes a SHA256 for every Central
   jar / pom / module file, and commits a `Dockerfile` back
   that pins both surfaces.
3. After the topic branch merges to the default branch,
   `build-container.yml` builds that Dockerfile, pushes the
   image to `ghcr.io/<owner>/<repo>/builder`, and signs it
   with cosign.
4. `scripts/init-release` looks up the digest of that image and
   writes `release.yml` pinned to it.
5. After that PR merges, pushing a tag fires `release.yml`. Two
   independent build jobs produce the artifacts in parallel
   (fully offline), a verify job byte-compares them, and if they
   match the bytes are attested, signed, and uploaded to a
   draft GitHub release.

## Quickstart

You'll need `git`, `gh` (with `repo` and `read:packages`
scopes), `curl`, and `bash` in your local shell.

Step 1 — scaffold the builder container:

```
./scripts/init-container /path/to/your-maven-project
```

This pushes a branch named `slsa-maven-init` to your fork and
prints a PR URL. Open the PR. The scan workflow runs against
the branch and adds a `Dockerfile` commit within a few minutes.
Review the Dockerfile and merge.

Once the merge lands on the default branch,
`build-container.yml` fires and publishes the pinned image to
GHCR.

Step 2 — scaffold the release workflow:

```
./scripts/init-release /path/to/your-maven-project
```

This looks up the digest of
`ghcr.io/<owner>/<repo>/builder:latest` and writes a
`release.yml` pinned to that exact digest. It pushes a second
branch (`slsa-maven-init-release`) and prints the PR URL. Merge
it.

Step 3 — cut a release:

```
git tag v1.0.0 && git push origin v1.0.0
```

The release workflow fires, produces the jar / sources /
javadoc / pom (and .module if your project publishes Gradle
module metadata), runs the two-pass byte-compare, attests
provenance, signs with cosign, and creates a GitHub draft
release. You review the draft, optionally add a GPG signature
alongside the cosign one, and publish.

## Scope

We work with Maven projects that have a single top-level
`pom.xml`. We produce the four (or five) standard release
artifacts that `mvn install` would copy into a local repo:

- `<artifactId>-<version>.jar`
- `<artifactId>-<version>-sources.jar`
- `<artifactId>-<version>-javadoc.jar`
- `<artifactId>-<version>.pom`
- `<artifactId>-<version>.module` (only if the project publishes
  Gradle Module Metadata)

We don't add or modify any file in your repo outside
`Dockerfile` and `.github/workflows/`. Project files (pom.xml,
src/, etc.) are detected and read, never edited.

A few things we don't do:

- Gradle. Maven only.
- Multi-module reactor releases as a unit. Single-artifact
  projects only.
- `mvn deploy` to Maven Central / Sonatype OSSRH. Releases are
  GitHub-side; pushing to Central is the maintainer's separate
  decision.
- GPG signing inside the workflow. The draft-release flow lets
  the maintainer sign locally if they want; sigstore + SLSA
  cover the bytes either way.
- Non-Central artifact repositories. The Dockerfile fetches from
  `repo1.maven.org/maven2/` only.

## Under the hood

### Reproducibility

Every artifact builds twice in independent containers and we
byte-compare the results. Mismatch fails the job, no artifact
ever leaves the runner. The reproducibility envelope:

- `SOURCE_DATE_EPOCH` from `git log -1 --format=%ct <tag>`,
  exported to `$GITHUB_ENV` in every job — process-inherited by
  any tool Maven shells out to.
- `-Dproject.build.outputTimestamp=<ISO-8601>` overriding
  whatever the pom committed, so a stale pom timestamp can't
  drift between passes.
- `-Dnotimestamp=true` for the Javadoc plugin, suppressing the
  `dc.created` HTML `<meta>` stamp.
- Per-pass container isolation: both passes run at the same
  absolute pwd inside their own container. Anything that bakes
  the build path into output bakes the *same* path in both
  passes by construction.

### Pinning

Two pinning surfaces, one Dockerfile:

1. **apt** — sources swapped to `snapshot.ubuntu.com/<timestamp>`
   so every apt fetch resolves to a fixed historical version.
   The emitted Dockerfile lists every package the strace touched
   with the version string and the .deb's SHA256.
2. **Maven Central** — every jar / pom / module file the build
   resolved into `~/.m2/repository` is pinned by SHA256 against
   `repo1.maven.org/maven2/`. Maven Central is content-
   addressable and immutable by convention, so the SHA pin is
   durable indefinitely.

The builder image then carries the full pre-resolved local
repo. Release-time `mvn -o` (offline) succeeds without network;
if anything tries to reach out, `mvn -o` fails loudly first.

### Signing

Every artifact gets a `cosign sign-blob` signature using
keyless OIDC against the workflow's own identity. No long-lived
key on a maintainer machine. Every signature lands in Rekor's
public transparency log, so anyone can independently audit the
issuance history. SLSA build provenance is attached separately
via `actions/attest-build-provenance`.

### JDK selection

The resolver checks the pom in priority order:

1. `maven-enforcer-plugin` `<requireJavaVersion>` — a real
   build-time pin, honored exactly.
2. `<prerequisites><jdk>…</jdk></prerequisites>` — deprecated
   but still seen, treated as a minimum.
3. `maven-toolchains-plugin` `<jdkToolchain>` selector.
4. Fall back to newest LTS available in the chosen Ubuntu base.

Profile-activation `<jdk>[N,)</jdk>` blocks are NOT considered
pins — they gate a profile if the JDK is N+, but don't require
it. Treating them as pins would force adopters onto unnecessary
JDK versions.

## Status

Work in progress.

## Notes

Real-world gaps surfaced during local validation against the
Apache Commons Lang3 fork — recording them here because each is
a non-obvious adopter-facing decision.

### Maven from `archive.apache.org`, not from apt

apt's `maven` package on Ubuntu LTS lags upstream Maven by
years. noble (24.04 LTS) carries 3.8.7 (Jan 2023). Many projects
declare `<requireMavenVersion>[3.9,)</requireMavenVersion>` via
the enforcer plugin and refuse to build under 3.8.x; Apache
Commons Lang3 is one such project. We sidestep the
LTS-vs-upstream cadence mismatch by installing Maven from
`archive.apache.org/dist/maven/maven-3/` by version + SHA512.
Apache's archive is content-addressable and immutable, same
trust model as cosign in `build-container.yml`. Bump
`MVN_VERSION` + `MVN_SHA512` in `scan.yml` together when
refreshing slsa-maven.

### Maven pin block emitted in chunks

A non-trivial Maven dep graph (commons-lang3 transitively
resolves 1,390 jar/pom files) produces a single RUN whose `&&`
chain exceeds Linux's `MAX_ARG_STRLEN` (~128 KB) — `exec` aborts
with "argument list too long". The emit step batches the pin
list into 100-artifact groups, one RUN per batch. ~14 layers for
a project the size of commons-lang, well under Docker's 127-
layer ceiling.

### Apache 2.0 license header on the emitted Dockerfile

When `scan.yml` commits the generated Dockerfile back to the
target repo, the next CI run that touches `mvn verify` on a
project with `apache-rat-plugin` (every Apache project, plus
many non-Apache ones) fails the `rat-check` goal because
Dockerfile has no recognised license header. We prepend the
generic Apache 2.0 boilerplate — *not* the ASF-attributed
"Licensed to the Apache Software Foundation" form, since
adopters aren't necessarily Apache projects. The phrasing
"Licensed under the Apache License, Version 2.0" is what RAT's
default ApacheV2 matcher recognises.
