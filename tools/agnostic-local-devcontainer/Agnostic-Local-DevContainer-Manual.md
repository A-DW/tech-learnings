# Agnostic Local DevContainer Manual

## Overview

This manual documents a **language-agnostic Dev Container template** for isolated, reproducible local development, built on Docker Compose and the [Dev Containers](https://containers.dev/) spec. The container setup *is* the product: a consistent workspace that can host any project, with project files living at the repository root next to `.devcontainer/` and `docs/`.

The full template lives in its own repository: [A-DW/agnostic-local-devcontainer](https://github.com/A-DW/agnostic-local-devcontainer). This file is the quick-reference version.

The shipped baseline is a Node 20 (Debian Bookworm) image, but the layout is meant to be adapted to Node, Next, Vite or Python via the Dockerfile.

---

## Repository layout

```text
.
├── .devcontainer/
│   ├── devcontainer.json      ← VS Code / Dev Containers entry point
│   ├── docker-compose.yml     ← service, volumes, ports, env
│   ├── Dockerfile             ← base image + system packages
│   └── post-create.sh         ← ownership fix + dependency install
├── docs/                      ← setup, workflow, customization, structure, troubleshooting
├── .gitignore                 ← multi-stack baseline (Node, Next, Vite, Python)
└── README.md
```

---

## Why this approach was chosen

- **Isolated:** runtimes, dependencies and tooling live in the container, not on the host.
- **Reproducible:** every machine gets the same environment.
- **Safer experimentation:** untrusted code and tooling run inside a container instead of against the host.
- **Convenient:** one command starts it, one command attaches a shell.

The trade-off is that Docker must be running, and any tool missing from the image has to be added to the `Dockerfile` and the container rebuilt.

---

## How the pieces fit together

| File | What it does |
|------|--------------|
| `Dockerfile` | Starts from `mcr.microsoft.com/devcontainers/javascript-node:1-20-bookworm`, adds `git`, `ca-certificates`, `curl`, `bash`, `procps`, `less`, `iproute2`, then switches to the `node` user with `/workspace` as workdir |
| `docker-compose.yml` | Builds the image, mounts the repo at `/workspace`, runs `sleep infinity` to keep the container alive, publishes ports, loads `../.env`, sets polling env vars and `restart: unless-stopped` |
| `devcontainer.json` | Points VS Code at the compose service `app`, sets `workspaceFolder` to `/workspace`, `remoteUser` to `node`, runs `post-create.sh`, forwards ports 3000/5173/8080, installs ESLint/Prettier/TypeScript extensions |
| `post-create.sh` | Creates and `chown`s the volume mount points, then runs `npm ci` (if `package-lock.json` exists) or `npm install` (if `package.json` exists) as `node` |

### Key design choices

- **Named volumes** `node_modules` and `npm_cache` keep dependencies off the bind mount, which avoids slow host I/O and host/container platform mismatches.
- **Root start, `node` runtime:** the container starts as `root` so `post-create.sh` can repair ownership on freshly created named volumes before work happens as `node`.
- **Polling file watchers:** `CHOKIDAR_USEPOLLING` and `WATCHPACK_POLLING` are set to `true` so hot reload works over bind mounts.
- **Language-agnostic by design:** `post-create.sh` only installs dependencies if a `package*.json` exists, so the template stays neutral for other stacks.

---

## Exact procedure

### 1. Prerequisites

Docker Desktop (running), a terminal, and any editor (VS Code recommended). No language runtimes are needed on the host.

### 2. Clone the repository

```bash
git clone https://github.com/A-DW/agnostic-local-devcontainer.git
cd agnostic-local-devcontainer
```

### 3. Start the container

```bash
docker compose -f .devcontainer/docker-compose.yml up -d
```

### 4. Attach a shell

```bash
docker exec -it devcontainer-app-1 bash
```

The container name depends on the Compose project name, so confirm it with `docker ps` if the command fails.

### 5. Work inside the container

Add or initialise project files at the repository root from the container shell.

### 6. Full reset

```bash
docker compose -f .devcontainer/docker-compose.yml down -v
```

The `-v` flag also deletes the named volumes (`node_modules`, `npm_cache`), so dependencies are reinstalled on the next start.

---

## Adapting it to another stack

| Stack | What to change |
|-------|----------------|
| Node / Next / Vite | Works as shipped; ports 3000, 5173 and 8080 are already published |
| Python | Swap the base image or add Python in the `Dockerfile`, adjust `post-create.sh` and the VS Code extensions |
| Any other | Install the runtime in the `Dockerfile`, add ports to `docker-compose.yml` and `devcontainer.json`, then reset and rebuild |

Further examples are in the repo's `docs/customization.md`.

---

## Common mistakes to avoid

| Mistake | Why it breaks things |
|---------|----------------------|
| Editing `Dockerfile` or `docker-compose.yml` without a rebuild | The running container keeps using the cached image; run `down -v` then `up -d` |
| Dev server bound to `127.0.0.1` | It is unreachable from the host; bind to `0.0.0.0` inside the container |
| Port missing from `docker-compose.yml` `ports` | The host cannot reach the service |
| `post-create.sh` not executable or not referenced | Dependencies silently do not install; check `postCreateCommand` and `chmod +x` |
| Missing `../.env` file | `docker-compose.yml` lists it under `env_file`, so create it (even empty) before `up` |
| Assuming a tool exists in the container | If it is not in the image, add it to the `Dockerfile` |

> **Security note:** the compose file publishes ports as `"3000:3000"` style mappings, which bind to all host interfaces by default. On an untrusted network, restrict them to loopback (for example `"127.0.0.1:3000:3000"`). Also keep `.env` out of version control and out of images.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Cannot connect to Docker daemon` | Docker Desktop not running | Start it and wait until it reports running |
| Build fails | Failing `RUN` step (bad package name, network) | Read the error, fix the `Dockerfile`, then `down -v` and `up -d` |
| Tools missing in the shell | Not installed in the image | Add to `Dockerfile`, reset and rebuild |
| `post-create.sh` did not run | Not referenced, or not executable | Check `postCreateCommand`, `chmod +x`, rebuild |
| Config changes ignored | Cached image | Reset and rebuild |
| `Permission denied` in `/workspace` | Named volume initialised as root-owned before the ownership fix | Full reset recreates the volumes cleanly |
| Port not reachable on host | Not published, or server bound to `127.0.0.1` | Add to `ports`, bind to `0.0.0.0` |

Standard reset-and-rebuild sequence:

```bash
docker compose -f .devcontainer/docker-compose.yml down -v
docker compose -f .devcontainer/docker-compose.yml up -d
```

---

## Final working state

```text
Host:       Docker Desktop + terminal/editor only
Container:  Node 20 (Bookworm), user node, /workspace = repo root
Volumes:    node_modules, npm_cache (named, persistent until down -v)
Ports:      3000, 5173, 8080 → host
Lifecycle:  up -d → exec bash → work → down -v (full reset)
```

Full template and docs: [A-DW/agnostic-local-devcontainer](https://github.com/A-DW/agnostic-local-devcontainer)
