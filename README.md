# linkerd-dev

This repository contains utilities for managing the Linkerd development
environment, especially a [devcontainer](https://containers.dev/). It also
provides a foundation of reusable tools for CI (in GitHub Actions).

This repository is **NOT** intended to be used typical Linkerd development. This
repository includes submodules only for the purpose of validating and automating
dev tooling changes.

## Devcontainer

The *devcontainer* image provides all of the tools needed for interactive
development. Images are tagged as `ghcr.io/linkerd/dev:vNN` and is expected to
be set in a `.devcontainer.json` file. For example:

```jsonc
{
    "name": "linkerd-dev",
    "image": "ghcr.io/linkerd/dev:v43",
    "extensions": [
        "DavidAnson.vscode-markdownlint",
        "golang.go",
        "kokakiwi.vscode-just",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "NathanRidley.autotrim",
        "rust-lang.rust-analyzer",
        "samverschueren.final-newline",
        "tamasfe.even-better-toml",
        "zxh404.vscode-proto3"
    ],
    "settings": {
        "go.lintTool": "golangci-lint"
    },
    "runArgs": [
        "--init",
        // Limit container memory usage.
        "--memory=12g",
        "--memory-swap=12g",
        // Use the host network so we can access k3d, etc.
        "--net=host"
    ],
    "overrideCommand": false,
    "remoteUser": "code",
    "mounts": [
        {
            "source": "/var/run/docker.sock",
            "target": "/var/run/docker-host.sock",
            "type": "bind"
        },
        {
            "source": "${localEnv:HOME}/.docker",
            "target": "/home/code/.docker",
            "type": "bind"
        }
    ]
}
```

Several containers and setup actions are provided to configure GitHub Actions
workflows.

## Tools

This repository also includes additional tools in [bin/]. These scripts capture
some common build and testing tasks. Most of these scripts are implemented with
[`just`](https://just.systems/). Just is a tool for writing build and test
'recipes'. These recipes may be invoked in the course of development or from

- **just-dev** lints for Devcontainer configuration and GitHub Actions workflows
- **just-cargo** helpers for running cargo, especially in CI
- **just-k3d** helpers for interacting with k3d clusters used for testing
- **just-md** lints markdown in a repository
- **just-sh** lints shell scripts in a respository.

### `just-k3d`

When using `just-k3d` to create k3d clusters, the `K3S_CHANNEL` environment
variable controls the version of Kubernetes used in k3d. This value must
correspond to a channel in [`./k3s-images.json`](k3s-images.json), which is
stored statically in the tools and devcontainer images. This prevents the k3s
version from varying dynamically without updating the dev image.

This file is generated by running `just sync-k3s-images` in this repo.

### linkerd/dev/ctions/setup-tools

The `linkerd/dev/actions/setup-tools` action unpacks the `tools` container so
that its binaries and configuration is available to subsequent steps in the job.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: linkerd/dev/actions/setup-tools@v43
      - uses: actions/checkout@v3
      - run: just-sh lint
      - run: just-dev lint-actions
      - run: just-dev check-action-images
```

## Go

The `go` container provides a Go toolchain (linters and other Go tooling are
included with the Tools image). The container can be used as a base image when
building via docker.

### Go in GitHub Actions

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: ghcr.io/linkerd/dev:v43-go
    steps:
      - uses: actions/checkout@v3
      - run: go mod download
      - run: go test ./...
```

#### linkerd/dev/actions/setup-go

The `linkerd/dev/actions/setup-go` action configures the job to use the
default go version.

Linting and testing tooling must be setup via `linkerd/dev/actions/setup-tools`.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        k8s:
          - v1.21
          - v1.26
    steps:
      # Install just* tooling and Go linters
      - uses: linkerd/dev/actions/setup-tools@v43
      # Configure the default Go toolchain
      - uses: linkerd/dev/actions/setup-go@v43
      - uses: actions/checkout@v3
      - run: just-k3d K3S_CHANNEL=${{ matrix.k8s }} create
      - run: go mod download
      - run: go test ./...
```

## Rust

The `rust` container provides a Rust toolchain and associated utilities needed
to build and test Rust code. The `rust-musl` container provides the same
toolchain, but with the `musl` target installed with cross-compilation support
for arm64 and armv7.

### Rust in GitHub Actions

These containers can be used in a workflow like so:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: ghcr.io/linkerd/dev:v43-rust
    steps:
      - uses: actions/checkout@v3
      - run: just-cargo fetch
      - run: just-cargo test-build
      - run: just-cargo test
```

Or, to build a static binary:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container: ghcr.io/linkerd/dev:v43-rust-musl
    steps:
      - uses: actions/checkout@v3
      - run: just-cargo fetch
      - run: just-cargo build
```

Build failures are automatically reported as annotations in the GitHub UI.

#### linkerd/dev/actions/setup-rust

The above example is fine when we're just building software, but when we want to
start networked services in Docker (namely, k3d clusters), the containerized
approach no longer works.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        k8s:
          - v1.21
          - v1.25
    env:
      K3S_CHANNEL: ${{ matrix.k8s }}
    steps:
      # Install just* tooling
      - uses: linkerd/dev/actions/setup-tools@v43
      # Configure the default Rust toolchain
      - uses: linkerd/dev/actions/setup-rust@v43
      - run: just-k3d create
      - run: just-cargo fetch
      - run: just-cargo test-build
      - run: just-cargo test
```

## Building

Build a single target (e.g. while editing the Dockerfile):

```sh
:; just build --target=rust
```

Update k3s images for all channels:

```sh
:; just sync-k3s-images
```

Push all images:

```sh
:; just push=true version=vNN build
```
