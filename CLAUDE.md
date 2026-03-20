# CLAUDE.md

This file provides context for Claude Code when working in this repository.

## Purpose

This is a fork of [frappe/frappe_docker](https://github.com/frappe/frappe_docker). The primary goal here is deploying ERPNext v15 to a local Umbrel server running Portainer.

## Deployment Target: Umbrel + Portainer

### Infrastructure

- **Host:** `umbrel.local` — SSH via `ssh umbrel@umbrel.local`
- **Portainer:** Accessible at `http://umbrel.local:9000`
- **ERPNext target URL:** `http://umbrel.local:7777`

### Docker Architecture (Important)

Portainer on Umbrel runs using **Docker-in-Docker (dind)**:

- `portainer_portainer_1` — the Portainer UI container (no shell tools inside)
- `portainer_docker_1` — the dind container (`docker:27.2.0-dind`) where all Portainer-managed stacks run
- The dind Docker socket is at `/data/docker.sock` inside `portainer_docker_1`

To run docker commands against the dind environment from the Umbrel host:
```bash
docker exec portainer_docker_1 sh -c 'docker -H unix:///data/docker.sock <command>'
```

The dind bind-mount for compose files: `/data/compose/` (maps to `/home/umbrel/umbrel/app-data/portainer/data/docker/compose/` on host).

### Portainer Stack Deployment

To deploy `prod_compose.yaml` as a Portainer stack:
1. Go to `http://umbrel.local:9000`
2. Stacks → Add stack → Web editor
3. Name: `erpnext`
4. Paste the contents of `prod_compose.yaml`
5. Deploy the stack

To update an existing stack:
1. Stacks → click the stack → Editor tab
2. Update content → Update the stack

### Checking Container Status

```bash
# From local machine:
ssh umbrel@umbrel.local "docker exec portainer_docker_1 sh -c 'docker -H unix:///data/docker.sock ps -a'"
```

## prod_compose.yaml

`prod_compose.yaml` is a self-contained single-file compose for Umbrel/Portainer. Unlike the upstream `compose.yaml` (which uses env vars and overlay files), this file:

- Hardcodes image version `frappe/erpnext:v15.91.1`
- Includes all required services: MariaDB, Redis (cache + queue), all Frappe workers
- Creates the ERPNext site automatically on first deploy via `create-site` service
- Exposes the app on port **7777** (`http://umbrel.local:7777`)
- Default credentials: `administrator` / `admin`

### First Deploy Notes

- `create-site` runs once and creates the site. On subsequent deploys it will fail (exit 1) because the site already exists — this is expected and harmless.
- If you need a clean install, delete the named Docker volumes (`db-data`, `sites`, `logs`, `redis-cache-data`, `redis-queue-data`) from Portainer → Volumes before redeploying.

### Known Issues Diagnosed

The original ERPNext stack deployed in Portainer was missing Redis and MariaDB containers. Root cause:
- The compose.yaml used only contained app services, not infrastructure
- `common_site_config.json` had empty Redis URLs (`redis://`), causing workers to attempt `localhost:6379`
- `queue-short`, `queue-long`, and `websocket` containers were crash-looping as a result
- No site had been created yet

`prod_compose.yaml` fixes all of these.
