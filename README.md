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
  - [Dependency Conditions (`wait_for`)](#dependency-conditions-wait_for)
  - [Curated Stacks](#curated-stacks)
- [CLI Reference](#cli-reference)
  - [Commands Overview](#commands-overview)
- [Plugins & Extensions](#plugins--extensions)
  - [Plugin Architecture](#plugin-architecture)
  - [Plugin Management Commands](#plugin-management-commands)
  - [Official First-Party Plugins](#official-first-party-plugins)
    - [ved-kit: PKI & Cryptography (`ved pki`, `ved crypt`)](#1-ved-kit-pki--cryptography)
    - [ved-mirror: Local Multi-Registry Cache (`ved mirror`)](#2-ved-mirror-local-multi-registry-cache)
- [Repositories](#repositories)

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
echo "0.2.19" > .ved/.ved-version
```

### 3. Pre-built Binaries

Download pre-compiled binaries from [GitHub Releases](https://github.com/velos-io/ved/releases):

```bash
# macOS ARM64 (Apple Silicon)
curl -fsSL https://github.com/velos-io/homebrew-tap/releases/download/v0.2.19/ved-darwin-arm64.tar.gz | tar -xz
sudo mv ved /usr/local/bin/

# Linux AMD64
curl -fsSL https://github.com/velos-io/homebrew-tap/releases/download/v0.2.19/ved-linux-amd64.tar.gz | tar -xz
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

### Docker Compose env_file Resolution

When compiling Docker Compose files (such as `.ved/docker-compose.yml`), Ved automatically rebases all `env_file` paths relative to the compose file directory. This complies with Docker Compose path resolution rules and ensures Compose finds referenced files without requiring extra flags.

### Configuration Directory Resolution

Ved supports declaring the configuration directory via the `config_dir` attribute in `App(...)` or `ved.yaml`. If not set, it defaults to `.` (with `{root}/config` fallback for legacy setups). Relative paths in `env_files` resolve against `config_dir`:

```python
app = App(
    name="my-app",
    version="0.1.0",
    config_dir="config", # or custom directory like "my-config", "."
    env_files=[".env", ".local.env"] # automatically resolves to config/.env, config/.local.env
)
```

The configuration directory is resolved in the following priority order:
1. `--config-dir <DIR>` (or `--config_dir <DIR>`) CLI flag (overrides `app.config_dir`)
2. `VED_CONFIG_DIR` environment variable
3. Specification attribute: `app.config_dir` in `Vedfile` or `config_dir` in `ved.yaml`
4. Project directory: `{root}/config`
5. Parent workspace fallback: `{root}/../config` (automatically used when running nested subprojects if `{root}/config` is absent)

### Directory Scopes and Path Resolution

Ved cleanly decouples working directory (`-d`), configuration directory (`--config-dir`), and runtime state (`--work-dir`):
- **Working directory (`--dir` / `-d`, default `.`):** Anchors the workspace root. Docker Compose receives this root via `--project-directory`; all relative paths declared within the generated Compose manifest (such as `env_file` and build contexts) anchor to this root.
- **Configuration directory (`--config-dir`, default `{root}/config`):** Holds specification environments (`.env`, `.local.env`) and catalogs. Paths declared in specifications resolve relative to the working directory root.
- **Work directory (`--work-dir`, default `{root}/.ved`):** Holds ephemeral runtime state, `.generated.env`, and default compiled Compose manifests (`.ved/docker-compose.yml`). The work directory location does not shift Compose path resolution.
- **Output directory (`--output-dir` / `-o`):** Redirects final compiled deployment manifests when specified.

### Deployment Target Configuration

Ved reads `DEPLOYMENT_TARGET` from `config/.local.env` and `config/.env` (default: `compose`). Set it to `k8s` to automatically route workflows to Kubernetes:

```bash
# config/.env
DEPLOYMENT_TARGET=k8s
```

When set to `k8s`, commands (`ved compile`, `ved up`, `ved down`, `ved ps`, `ved reset`, `ved test`, `ved verify`) orchestrate Kubernetes manifests in `installer/k8s/` without requiring `--target k8s`.

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


### Entrypoint and Command (`entrypoint`, `command`)

Services and jobs support overriding container entrypoints and command arguments:

```python
cerbos = Service(
    name="cerbos",
    image="ghcr.io/cerbos/cerbos:0.39.0",
    entrypoint=["cerbos"],
    command=["server", "--config=/dev/null"],
)
```

Ved reconciles runtime differences across targets:

- **Docker Compose**: Emits `entrypoint` directly as container entrypoint, and `command` as default arguments.
- **Kubernetes**: Emits `entrypoint` as container `command`, and emits `command` as container `args`. When only `command` is supplied without `entrypoint`, Ved populates `command` directly for backwards compatibility.

### Health Checks (`Health`)

Ved supports HTTP, TCP, and Exec probes across both Compose and Kubernetes targets:

- **HTTP GET**: `Health.http_get("http://localhost:8080/health", interval=5, timeout=3, retries=5)`
- **TCP Socket**: `Health.tcp_socket(5432)`
- **Shell Command**: `Health.exec("redis-cli ping")` (uses `CMD-SHELL` / `sh -c`)
- **Direct Exec (Distroless / Shell-less)**: `Health.exec(["cerbos", "healthcheck"])` (uses direct `CMD` exec form without invoking `/bin/sh`, ideal for distroless images like Cerbos)

### Dependency Conditions (`wait_for`)

Services can declare startup dependencies using `wait_for`. Ved supports both dot (`service.condition`) and colon (`service:condition`) syntax:

| Syntax | Condition | Behavior |
|:---|:---|:---|
| `["db"]` or `["db.healthy"]` | `healthy` (default) | Waits for service health check to pass. If no health check is defined, falls back to `started`. |
| `["db.started"]` | `started` | Waits for container to start. |
| `["flyway.completed"]` or `["flyway.success"]` | `completed` / `success` | Waits for container to exit successfully with code 0 (`service_completed_successfully` in Compose). Ideal for database migrations and one-off seed tasks. |
| `["db:5432"]` or `["ext:5432"]` | TCP probe | Probes the TCP port until it accepts connections before starting the service. |

### Resource Limits, Ulimits, and Profiles

Services and jobs support resource allocations, shared memory sizes, kernel ulimits, and multi-profile membership:

```python
worker = Service(
    name="worker",
    image="ghcr.io/velos-io/worker:latest",
    profiles=["core", "batch"],
    shm_size="1gb",
    deploy={
        "resources": {
            "limits": {"cpus": "2.0", "memory": "2G"},
            "reservations": {"cpus": "0.5", "memory": "512M"},
        }
    },
    ulimits={
        "nofile": {"soft": 65536, "hard": 65536}
    },
)
```

- **Project Isolation**: Ved omits `container_name` by default so Docker Compose namespaces containers under `COMPOSE_PROJECT_NAME` without name collisions across parallel projects. Set `container_name="custom-name"` only when an explicit global container name is required.

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
| `ved verify` | Run full verification lifecycle (`reset` -> `up` -> `ps` -> `test`) |
| `ved plugin` | Manage external subcommand plugins (`install`, `list`, `remove`) |
| `ved validate` | Validate specification against schema without compiling |
| `ved build` | Build detected project modules via native build wrappers |
| `ved sync` | Rescan workspace and refresh cached module state |
| `ved init` | Initialize project with wrapper and specification |
| `ved deinit` | Remove Ved files and state from project |
| `ved config` | Inspect and manage configuration files |
| `ved version` | Display compiler version |

### Global Options

These options apply across all subcommands:

```text
Options:
  -d, --dir <DIR>         Working directory for command execution [default: .]
      --config-dir, --config_dir <DIR>  Configuration directory [default: {root}/config, fallback: ../config] [env: VED_CONFIG_DIR=]
      --work-dir <DIR>    Mutable runtime state directory [default: {root}/.ved] [env: VED_WORK_DIR=]
  -o, --output-dir <DIR>  Unified artifact directory [default: legacy layout] [env: VED_OUTPUT_DIR=]
  -h, --help              Print help
  -V, --version           Print version
```

---

## Plugins & Extensions

Ved features an extensible subcommand plugin architecture. Any executable named `ved-<cmd>` located in `~/.ved/plugins/bin` or discoverable on `$PATH` seamlessly extends Ved as `ved <cmd>`.

### Plugin Architecture

When an external command like `ved pki` or `ved mirror` is dispatched:
1. **Lookup Hierarchy**: Ved inspects `~/.ved/plugins/bin/ved-<cmd>` first. If absent, it queries `$PATH`.
2. **Context Injection**: Ved transparently passes the active workspace context to plugin child processes via environment variables:
   - `VED_WORKSPACE_ROOT`: Absolute path to the resolved workspace root.
   - `VED_CONFIG_DIR`: Absolute path to configuration files (`--config-dir` or default).
   - `VED_WORK_DIR`: Absolute path to the mutable runtime directory (`--work-dir` or `.ved`).
   - `VED_OUTPUT_DIR`: Path to the unified artifact destination, if specified.
3. **Transparent Execution**: Exit codes, standard input, standard output, and standard error streams are attached directly to your terminal.

---

### Plugin Management Commands

The built-in `ved plugin` command provides lifecycle management for installed extensions:

#### 1. Install Plugins (`ved plugin install`)

Install first-party plugins from GitHub or link a local binary during development:

```bash
# Install official plugins via shorthand
ved plugin install kit
ved plugin install mirror

# Link a local development binary into ~/.ved/plugins/bin
ved plugin install kit --path ./target/release/ved-kit
ved plugin install mirror --path ./target/release/ved-mirror
ved plugin install custom-tool --path /usr/local/bin/my-binary
```

#### 2. List Installed Plugins (`ved plugin list`)

Inspect all plugin binaries registered in `~/.ved/plugins/bin`:

```bash
ved plugin list
```

Output:
```text
Installed plugins in /Users/username/.ved/plugins/bin:
  - ved crypt
  - ved crypto
  - ved mirror
  - ved pki
```

#### 3. Remove Plugins (`ved plugin remove`)

Uninstall a plugin and clean up its executable symlinks:

```bash
ved plugin remove kit
ved plugin remove mirror
```

---

### Official First-Party Plugins

#### 1. `ved-kit`: PKI & Cryptography

The `kit` plugin provides enterprise PKI management and authenticated local file encryption via two subcommands: `ved pki` and `ved crypt`.

##### `ved pki` (Public Key Infrastructure)
- **`ved pki trust <HOST:PORT> [--cacerts]`**: Probes an upstream endpoint over TLS, downloads its certificate chain, detects missing CA trust anchors, and optionally injects the root certificate directly into Java's system keystore (`$JAVA_HOME/lib/security/cacerts`) to permanently resolve Java `PKIX path building failed` errors.
- **`ved pki ca [--cn <NAME>] [-o <DIR>]`**: Bootstraps an offline local Development Root CA certificate and private key in `.ved/certs/ca/`.
- **`ved pki issue <NAME> [--san <DNS_OR_IP>]...`**: Issues an X.509 leaf certificate and key signed by the local Development CA, complete with Subject Alternative Names (SANs).

```bash
# Diagnose and fix Java PKIX handshake errors
ved pki trust internal-nexus.corp:8443 --cacerts

# Create a local development root CA
ved pki ca --cn "My Project Root CA"

# Mint a local TLS certificate for local ingress domains
ved pki issue localhost --san localhost --san 127.0.0.1 --san api.local.dev
```

##### `ved crypt` (Local Cryptography)
- **`ved crypt enc <FILE> [-k <KEY>] [-o <OUT>]`**: Encrypts arbitrary files using authenticated AES-256-GCM. If no key is supplied, securely generates and displays a random 256-bit hexadecimal key.
- **`ved crypt dec <FILE> -k <KEY> [-o <OUT>]`**: Decrypts and verifies the ciphertext against authentication tags.
- **`ved crypt split <FILE> [--chunk-size <BYTES>] [-o <DIR>]`**: Splits large datasets or container layers into chunk files, generating a cryptographic SHA-256 integrity manifest.
- **`ved crypt merge <DIR> [-o <FILE>]`**: Verifies manifest chunk hashes and reassembles the original file.

```bash
# Encrypt environment configuration
ved crypt enc app.env -o app.env.enc

# Decrypt configuration
ved crypt dec app.env.enc -k <KEY_HEX> -o app.env

# Split a 1GB dataset into 50MB chunks for transfer
ved crypt split dataset.tar --chunk-size 52428800 -o chunks/

# Reassemble and verify integrity
ved crypt merge chunks/ -o dataset.tar
```

#### 2. `ved-mirror`: Local Multi-Registry Cache

`ved-mirror` runs a lightweight, high-performance pull-through caching server for package managers:
- **Unified Endpoints**: Exposes `/maven/*`, `/gradle/*`, and `/npm/*` under a single HTTP server.
- **Offline Reliability**: Caches downloaded artefacts to disk with atomic writes. When offline (`--offline`), packages continue serving from cache.
- **Client Configuration Generation**: Generates ready-to-use snippets for `settings.xml`, `build.gradle.kts`, and `.npmrc`.

```bash
# Start cache server on default port 8081
ved mirror serve --port 8081 --dir ~/.ved/mirror-cache

# Run in strictly offline mode (no upstream calls)
ved mirror serve --offline --dir ~/.ved/mirror-cache

# Print configuration instructions for Maven, Gradle, and npm
ved mirror client-config --port 8081
```

---

## Repositories

- Compiler & Orchestrator: [github.com/velos-io/ved](https://github.com/velos-io/ved)
- PKI & Cryptography Plugin (`ved-kit`): [github.com/velos-io/ved-kit](https://github.com/velos-io/ved-kit)
- Multi-Registry Mirror Plugin (`ved-mirror`): [github.com/velos-io/ved-mirror](https://github.com/velos-io/ved-mirror)
- Documentation Source: [github.com/velos-io/veddocs](https://github.com/velos-io/veddocs)
- Homebrew Tap: [github.com/velos-io/homebrew-tap](https://github.com/velos-io/homebrew-tap)
- Stacks Catalog: [github.com/velos-io/stacks](https://github.com/velos-io/stacks)
