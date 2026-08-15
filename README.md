# Rust Docker CI Cache Experiment

This is a small Axum project used to reproduce slow uncached Rust Docker builds.

## Reproduce locally

Run each build twice and compare the logs:

```sh
time docker buildx build --progress=plain -f Dockerfile.cargo-chef .
time docker buildx build --progress=plain -f Dockerfile.cache-mount .
```

`Dockerfile.cargo-chef` mirrors the common planner/cook/builder layout. It can
compile the dependency graph in `cargo chef cook`, then compile dependencies a
second time during the final `cargo build` because the target directory from
the cook stage is not persisted as a BuildKit cache mount.

`Dockerfile.cache-mount` uses one release build and persists Cargo's registry,
git checkout, and target directories through BuildKit cache mounts. GitHub
Actions exports the BuildKit layers with `type=gha`; the cache mounts are
therefore available to subsequent builds on the same workflow cache scope.

## Faber finding

Faber's API build showed the same pattern. In run
https://github.com/panitdev/faber/actions/runs/31878300640, `cargo chef cook`
took about 445 seconds and the final `cargo build` started compiling crates
again. Faber also has a materially larger graph than this sample, including
`deno_core`/V8 and a git dependency, so the sample isolates the Docker cache
behavior rather than reproducing Faber's exact duration.
