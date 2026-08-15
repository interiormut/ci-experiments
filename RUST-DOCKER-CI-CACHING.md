# Fixing Very Slow Rust Docker Builds in CI

This guide explains why a Rust Docker image can spend many minutes compiling
the same dependency graph more than once, why `cargo-chef` does not
automatically prevent it, and how to build a reliable cache for GitHub Actions
and other BuildKit-based CI systems.

The short version is:

> A cargo-chef recipe is not a compiled dependency cache. Persist and reuse
> Cargo's `target` directory, and use the same cache mount for both
> `cargo chef cook` and the final `cargo build`.

The verified reproduction is available at
[interiormut/ci-experiments](https://github.com/interiormut/ci-experiments).

## Symptoms

You are likely seeing this issue when CI has one or more of these symptoms:

- A Rust Docker build takes several minutes despite a small application.
- Logs show `cargo chef cook --release` compiling a large dependency graph.
- A later `cargo build --release` starts compiling many of the same crates.
- Docker reports cache hits for the planner and recipe steps, but Rust still
  compiles dependencies.
- A local build is fast after the first run, while a fresh GitHub-hosted runner
  is slow on every run.
- `cache-from: type=gha` and `cache-to: type=gha,mode=max` are configured, but
  the Rust compilation time does not materially improve.

The important distinction is that Docker can reuse a `RUN` layer while Cargo
still decides that previously generated compilation artifacts are unavailable
or invalid. Docker layer hits and Cargo incremental/release artifacts are
different caches.

## Root Cause


The usual cargo-chef layout has these stages:

1. `planner` copies manifests and source metadata, then creates `recipe.json`.
2. `cacher` runs `cargo chef cook`, compiling a synthetic project containing
   the dependency graph but not the real application source.
3. `builder` copies the real source and runs `cargo build`.
4. `runner` receives only the final binary.

The recipe makes dependency changes less likely to invalidate the dependency
build layer when application source changes. It does not guarantee that the
final Cargo invocation will reuse every artifact produced by `cook`.

### Why the second compilation happens

Cargo's reusable compilation state is primarily under:

- `/app/target`
- `/usr/local/cargo/registry`
- `/usr/local/cargo/git`

The most important directory is `/app/target`. It contains compiled libraries,
build-script output, metadata, fingerprints, and the release binary. If that
state is not shared by the `cook` and final build commands, Cargo has to rebuild
the dependency graph.

Even when the stages appear related through `FROM cacher`, relying only on
ordinary image layers is fragile. The synthetic project used by `cargo chef
cook` and the real project used by the final build have different source and
fingerprint inputs. The final Cargo invocation can therefore invalidate or fail
to use artifacts from the synthetic build. A persistent BuildKit cache mount
gives both commands the same Cargo target directory and makes the intended
reuse explicit.

### What happened in Faber

Faber's API Dockerfile had this structure:

```dockerfile
RUN cargo chef cook --release --locked --recipe-path recipe.json
RUN cargo build --release --locked -p api
```

The API dependency graph is unusually expensive. It includes Axum, Reqwest,
Diesel, Russh, a git dependency, and the `harness` path dependency, which pulls
in Deno/V8-related crates.

In [Faber run 31878300640](https://github.com/panitdev/faber/actions/runs/31878300640):

- `cargo chef cook` finished after approximately 445 seconds.
- The final `cargo build` then compiled dependencies again.
- The API image step took approximately 13 minutes.

The problem was not that Axum was intrinsically too slow. Axum made a useful
small reproduction, while Faber's graph amplified the cost of the duplicate
compilation.

## Conditions That Trigger It

The issue is most visible when all or most of these conditions apply:

1. The Dockerfile runs both `cargo chef cook` and a final `cargo build`.
2. The two commands do not mount the same `/app/target` cache.
3. The CI runner is fresh or its BuildKit state is not persistent.
4. The release dependency graph contains native, procedural-macro, or large
   code-generation crates.
5. The workspace includes path dependencies, git dependencies, or multiple
   binaries and libraries.
6. The build target, Rust toolchain, platform, or important Cargo features
   change between builds.

The problem can also be hidden locally because a developer's Docker builder has
already accumulated a warm cache. GitHub-hosted runners do not retain local
builder state between jobs, so the external cache configuration must be
correct.

## Diagnose It

### Inspect the Docker build log

Use plain progress output so individual Cargo commands are visible:

```sh
docker buildx build \
  --progress=plain \
  --file crates/api/Dockerfile \
  --tag faber-api:debug \
  --load \
  .
```

Look for this pattern:

```text
[cacher] RUN cargo chef cook --release --locked --recipe-path recipe.json
    Compiling axum ...
    Compiling tokio ...
    Finished `release` profile ...

[builder] RUN cargo build --release --locked -p api
    Compiling axum ...
    Compiling tokio ...
```

Seeing the same third-party crates under both commands is the decisive signal.

### Verify that the target directory is being reused

Temporarily add diagnostics to both build commands:

```dockerfile
RUN --mount=type=cache,id=target-api,target=/app/target,sharing=locked \
    find /app/target -maxdepth 2 -type f | sort | head -50 && \
    cargo build --release --locked
```

Do not use the file listing as the permanent fix. It is only a way to prove
whether the command sees the expected target state.

### Distinguish dependency work from application work

On a correctly shared target cache, the final build after `cargo chef cook`
should look approximately like this:

```text
Compiling your-application v0.1.0 (/app)
Finished `release` profile ...
```

It should not list most of the registry dependency graph again. A small amount
of rebuilding is normal when source files, features, toolchains, or dependency
versions changed.

## The Recommended Fix

Keep cargo-chef if its recipe improves Docker layer reuse, but mount the same
Cargo caches in both stages.

```dockerfile
# syntax=docker/dockerfile:1

FROM rust:bookworm AS chef
WORKDIR /app
RUN cargo install cargo-chef --locked

FROM chef AS planner
COPY Cargo.toml Cargo.lock ./
COPY crates ./crates
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS cacher
COPY --from=planner /app/recipe.json recipe.json
RUN --mount=type=cache,id=faber-cargo-registry,target=/usr/local/cargo/registry,sharing=locked \
    --mount=type=cache,id=faber-cargo-git,target=/usr/local/cargo/git,sharing=locked \
    --mount=type=cache,id=faber-api-target,target=/app/target,sharing=locked \
    cargo chef cook --release --locked --recipe-path recipe.json

FROM cacher AS builder
COPY Cargo.toml Cargo.lock ./
COPY crates ./crates
RUN --mount=type=cache,id=faber-cargo-registry,target=/usr/local/cargo/registry,sharing=locked \
    --mount=type=cache,id=faber-cargo-git,target=/usr/local/cargo/git,sharing=locked \
    --mount=type=cache,id=faber-api-target,target=/app/target,sharing=locked \
    cargo build --release --locked -p api \
    && mkdir -p /out \
    && cp target/release/api /out/faber-api

FROM debian:bookworm-slim AS runner
COPY --from=builder /out/faber-api /usr/local/bin/faber-api
ENTRYPOINT ["/usr/local/bin/faber-api"]
```

### Why the mount must appear twice

BuildKit cache mounts are attached to individual `RUN` instructions. The
mount in `cacher` is not automatically attached to the `RUN` instruction in
`builder`. Both commands must use:

```dockerfile
--mount=type=cache,id=faber-api-target,target=/app/target,sharing=locked
```

The `id` and `target` must match. If the IDs differ, the two commands receive
different caches and the duplicate compilation returns.

### Why the binary must be copied out of the mount

Files created only inside a cache mount are not part of the image layer. The
final image must copy the binary to a normal path such as `/out/faber-api`
inside the same `RUN` instruction, then copy that normal path into the runtime
stage:

```dockerfile
RUN --mount=type=cache,id=faber-api-target,target=/app/target \
    cargo build --release --locked -p api \
    && cp target/release/api /out/faber-api

COPY --from=builder /out/faber-api /usr/local/bin/faber-api
```

Without this step, the runtime stage may not contain the binary because the
cache mount is intentionally external to the committed image filesystem.

## Simpler Alternative: One Build

If cargo-chef is not providing a measurable benefit, remove the synthetic cook
stage and perform one release build with persistent caches:

```dockerfile
# syntax=docker/dockerfile:1
FROM rust:bookworm AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY crates ./crates
RUN --mount=type=cache,id=faber-cargo-registry,target=/usr/local/cargo/registry,sharing=locked \
    --mount=type=cache,id=faber-cargo-git,target=/usr/local/cargo/git,sharing=locked \
    --mount=type=cache,id=faber-api-target,target=/app/target,sharing=locked \
    cargo build --release --locked -p api \
    && mkdir -p /out \
    && cp target/release/api /out/faber-api

FROM debian:bookworm-slim
COPY --from=builder /out/faber-api /usr/local/bin/faber-api
ENTRYPOINT ["/usr/local/bin/faber-api"]
```

This is often the best choice for a small service. It avoids the extra
synthetic compilation entirely. For a large workspace, cargo-chef plus a
shared target cache can still be useful, particularly when application source
changes frequently but manifests and dependency features remain stable.

## GitHub Actions Configuration

Use Docker Buildx and export the external cache with a stable, image-specific
scope:

```yaml
name: Build API image

on:
  push:
    branches: [main]
  pull_request:

jobs:
  api:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v6
        with:
          context: .
          file: crates/api/Dockerfile
          push: false
          tags: faber-api:ci
          cache-from: type=gha,scope=faber-api
          cache-to: type=gha,mode=max,scope=faber-api
```

### Cache scope rules

- Use a different scope for each materially different image, such as `api`
  and `ui`.
- Keep the scope stable across commits so later runs can import it.
- Include the target platform in the cache ID if multiple architectures share
  the same builder cache.
- Use distinct target cache IDs for different Rust toolchains, feature sets,
  or incompatible build profiles.
- Use `mode=max` when the cache needs intermediate layers and cache mounts,
  not only the final image layers.

For a matrix build, avoid concurrent writers corrupting or replacing the same
logical cache. Use separate scopes per platform, or serialize jobs that share a
scope.

## Common Incorrect Fixes

### Adding `cargo-chef` without sharing `target`

This changes the dependency planning strategy but does not guarantee reuse of
the artifacts produced by `cargo chef cook`. It is the exact failure mode
investigated here.

### Caching only the Cargo registry

The registry cache avoids downloading crates again. It does not avoid compiling
them. `/app/target` is the cache that contains compiled output.

### Caching only Docker's final image

The final runtime image contains the application binary, not the intermediate
Rust compilation artifacts. It cannot help the next source build compile
dependencies.

### Using different cache IDs in the two stages

These are separate caches:

```dockerfile
id=target-cook
id=target-build
```

The final build will not see the cooked artifacts. Use one stable ID for the
compatible commands.

### Mounting the target directory but not copying the binary out

The resulting image can be missing the binary because cache mounts are not
normal image filesystem content. Copy it to `/out` during the build step.

### Running `cargo clean`

`cargo clean` deliberately deletes the state needed for reuse. It should not be
part of a normal Docker build. If a clean build is needed for a special release
or security procedure, treat it as an intentional cache bypass.

### Rebuilding every architecture into one cache

Compiled artifacts are platform and toolchain dependent. Sharing one target
cache between incompatible `linux/amd64`, `linux/arm64`, Rust versions, or
feature configurations can cause misses or unsafe cache interactions. Separate
the IDs and GitHub cache scopes.

## Validation Checklist

Use this checklist after changing the Dockerfile:

- [ ] `Cargo.lock` is committed and the build uses `--locked`.
- [ ] `cargo chef cook` and the final `cargo build` use the same target cache ID.
- [ ] Cargo registry and git caches are mounted when downloads are expensive.
- [ ] The final binary is copied from the cache-mounted target directory to a
      normal image path in the same `RUN` instruction.
- [ ] `docker/build-push-action` has both `cache-from` and `cache-to`.
- [ ] The GitHub Actions cache scope is stable and image-specific.
- [ ] `mode=max` is used when intermediate build cache is needed.
- [ ] CI logs show dependencies compiling once, not once in `cook` and again in
      the final build.
- [ ] A second build with unchanged manifests and source is mostly `CACHED` or
      compiles only changed application crates.
- [ ] The runtime image contains and can execute the copied binary.

## Measuring the Result

Always compare the relevant Cargo commands, not only the total workflow time:

```sh
time docker buildx build \
  --progress=plain \
  --file Dockerfile.cargo-chef \
  --tag ci-experiments:baseline \
  --load \
  .

time docker buildx build \
  --progress=plain \
  --file Dockerfile.cargo-chef-fixed \
  --tag ci-experiments:fixed \
  --load \
  .
```

For the fixed build, the important evidence is not that `cargo chef cook`
becomes free. The important evidence is that the final `cargo build` no longer
recompiles the third-party dependency graph.

## Faber Recommendation

Apply the shared-cache pattern to `crates/api/Dockerfile`:

1. Add registry, git, and target cache mounts to `cargo chef cook`.
2. Add the same mounts, with the same IDs, to `cargo build`.
3. Copy `target/release/api` to a normal `/out` path before the runtime stage.
4. Keep the API GitHub Actions cache scope separate from the UI scope.
5. Re-run the image workflow and inspect the API log for duplicate
   `Compiling axum`, `Compiling deno_core`, V8, and other dependency lines.

The fix does not make Faber's dependency graph small. It ensures that the
expensive graph is compiled once per cold cache, then reused by the final
application build and subsequent compatible CI builds.

## Limitations

Caching cannot eliminate all build time:

- A genuinely cold cache still has to compile every dependency once.
- A Rust toolchain change invalidates many artifacts.
- A `Cargo.lock` or dependency feature change can invalidate the graph.
- Native dependencies may rebuild when system libraries or target platforms
  change.
- Large code-generation crates such as V8 can remain expensive on a cold build.
- GitHub Actions cache eviction and quota limits can cause occasional cold
  builds.

The goal is to prevent accidental duplicate compilation and make the remaining
cold-build cost explicit and measurable.
