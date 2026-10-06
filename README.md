# Ved User Guide

> High-performance deployment compiler and orchestrator for developer environments.

Ved compiles declarative specifications (`Vedfile`, Starlark DSL, or shorthand syntax) into deterministic deployment manifests (Docker Compose, Kubernetes) and orchestrates local workflows.

Published documentation: [https://velos-io.github.io/veddocs](https://velos-io.github.io/veddocs)

---

## Table of Contents

- [Installation](#installation)
  - [Homebrew](#1-homebrew-recommended)
  - [Project Wrapper (`vew`)](#2-project-wrapper-vew)
  - [Pre-built Binaries](#3-pre-built-binaries)
  - [From Source](#4-from-source)
- [Project Setup](#project-setup)
  - [Initialize New Project](#initialize-a-new-project)
  - [Convert Docker Compose](#convert-an-existing-docker-compose-file)
  - [Auto-detect Multi-module Project](#scaffold-from-a-multi-module-repository)
- [Specification Guide (`Vedfile`)](#specification-guide-vedfile)
  - [Starlark DSL](#starlark-dsl-vedfile)
  - [Shorthand Syntax](#shorthand-syntax)
  - [Curated Stacks](#curated-stacks)
- [CLI Reference](#cli-reference)
  - [Commands Overview](#commands-overview)
- [Repository](#repository)

---

## Installation

### 1. Homebrew (Recommended)

Install the latest release via the official tap:

```bash
brew tap velos-io/tap
brew install velos-io/tap/ved
```

Verify the installation:

```bash
ved --version
```

### 2. Project Wrapper (`vew`)

Projects configured with Ved include the `vew` wrapper script. It automatically downloads and caches the project's pinned Ved version in `.ved/bin/` without requiring global system installation.

```bash
./vew compile
./vew up
```

Pin or update versions using `.ved/.ved-version`:

```bash
echo "0.1.15" > .ved/.ved-version
```

### 3. Pre-built Binaries

Download pre-compiled binaries from [GitHub Releases](https://github.com/velos-io/ved/releases):

```bash
# macOS ARM64 (Apple Silicon)
curl -fsSL https://github.com/velos-io/homebrew-tap/releases/download/v0.1.13/ved-darwin-arm64.tar.gz | tar -xz
sudo mv ved /usr/local/bin/

# Linux AMD64
curl -fsSL https://github.com/velos-io/homebrew-tap/releases/download/v0.1.13/ved-linux-amd64.tar.gz | tar -xz
sudo mv ved /usr/local/bin/
```

### 4. From Source

Install using the Rust toolchain:

```bash
cargo install --git https://github.com/velos-io/ved --bin ved
```

---

## Project Setup

### Initialize a New Project

Scaffold a clean workspace with the `vew` wrapper and a template `Vedfile`:

```bash
ved init
```

Generated structure:
```text
my-project/
├── .ved/
│   ├── .gitignore
│   ├── .ved-version
│   └── docker-compose.yml  # generated on compile
├── vew
└── Vedfile
```

### Convert an Existing Docker Compose File

Translate an existing `docker-compose.yml` into a native `Vedfile`:

```bash
# Auto-discover docker-compose.yml in project
ved init --compose

# Or specify a custom compose file path
ved init --from-compose docker-compose.yml
```

Ved parses services, build contexts, ports, volumes, health checks, and dependencies into Starlark DSL.

### Scaffold from a Multi-module Repository

Automatically scan your repository to detect modules (Maven, Gradle, Cargo, Go, Node, Python) and generate service definitions:

```bash
ved init --project
```

### Deployment Target Configuration

Ved reads `DEPLOYMENT_TARGET` from `config/.local.env` and `config/.env` (default: `compose`). Set it to `k8s` to automatically route workflows to Kubernetes:

```bash
# config/.env
DEPLOYMENT_TARGET=k8s
```

When set to `k8s`, commands (`ved compile`, `ved up`, `ved down`, `ved ps`, `ved reset`, `ved test`) orchestrate Kubernetes manifests in `installer/k8s/` without requiring `--target k8s`.

### Clean Up / Teardown

Remove Ved configuration, wrapper, and runtime state:

```bash
ved deinit
# or non-interactive:
ved deinit --force
```

---

## Specification Guide (`Vedfile`)

Ved supports both Starlark DSL and declarative YAML shorthand.

### Starlark DSL (`Vedfile`)

Starlark provides Python-like expressiveness with hermetic, deterministic execution:

```python
from ved import Service, Port, Mount, Health

app = Service(
    name="api",
    image="ghcr.io/my-org/api:v1.0",
    ports=[Port(8080, host=8080)],
    env={
        "PORT": "8080",
        "DATABASE_URL": "postgres://postgres:secret@db:5432/app",
    },
    mounts=[
        Mount.volume("api-data", "/data"),
    ],
    wait_for=["db"],
    health=Health(
        http_get="http://localhost:8080/health",
        interval=5,
        timeout=3,
        retries=5,
    ),
)

db = Service(
    name="db",
    image="postgres:16-alpine",
    ports=[Port(5432, host=5432)],
    env={
        "POSTGRES_PASSWORD": "secret",
        "POSTGRES_DB": "app",
    },
)

services = [app, db]
```

### Shorthand Syntax

For concise declarations, use `ved.yaml` or `.ved`:

```yaml
version: "1.0"
services:
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    env:
      POSTGRES_PASSWORD: secret

  api:
    image: my-org/api:latest
    ports:
      - "8080:8080"
    depends_on:
      - db
```

### Curated Stacks

Inject reusable dependency bundles (e.g. OpenTelemetry observability) with a single call:

```python
from ved import Stack

Stack.add("otel")
```

The `otel` stack automatically provisions:
- **Jaeger** (`16686`, `14317`, `14318`)
- **Prometheus** (`9090`)
- **OpenTelemetry Collector** (`4317`, `4318`, `8888`, `8889`)
- **Grafana** (`3000`)

---

## CLI Reference

### Commands Overview

| Command | Description |
|:---|:---|
| `ved compile` | Compile spec into target manifests (`.ved/docker-compose.yml`) |
| `ved up` | Compile and start stack services |
| `ved down` | Stop running stack services |
| `ved ps` | Show container statuses, health, and exposed port mappings |
| `ved reset` | Stop containers and wipe volumes/networks |
| `ved test` | Run smoke tests against active stack endpoints |
| `ved validate` | Validate specification against schema without compiling |
| `ved build` | Build detected project modules via native build wrappers |
| `ved sync` | Rescan workspace and refresh cached module state |
| `ved init` | Initialize project with wrapper and specification |
| `ved deinit` | Remove Ved files and state from project |
| `ved config` | Inspect and manage configuration files |
| `ved version` | Display compiler version |

### Options

```text
Options:
  -f, --file <FILE>   Path to Vedfile or spec [default: Vedfile]
  -v, --verbose       Enable verbose log output
  -h, --help          Print help
  -V, --version       Print version
```

---

## Repository

- Compiler & Orchestrator: [github.com/velos-io/ved](https://github.com/velos-io/ved)
- Documentation Source: [github.com/velos-io/veddocs](https://github.com/velos-io/veddocs)
- Homebrew Tap: [github.com/velos-io/homebrew-tap](https://github.com/velos-io/homebrew-tap)
- Stacks Catalog: [github.com/velos-io/stacks](https://github.com/velos-io/stacks)
