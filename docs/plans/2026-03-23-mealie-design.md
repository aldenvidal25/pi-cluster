# Mealie Deployment Design

## Overview

Deploy Mealie (recipe manager) to the avraham k3s cluster using FluxCD with the same Kustomize base/overlay pattern used by audiobookshelf. Mealie runs with a PostgreSQL backend and is exposed externally via a Cloudflare tunnel.

## Cluster Context

- **Controlplane:** `avraham-server` (HP-t620, label: `storage: m2-16gb`)
- **Worker:** `metuselah-worker` (A68HM, label: `storage: ssd`)
- **Target node:** Worker (`metuselah-worker`)
- **FluxCD:** Already bootstrapped, reconciles `./apps/staging` every 1 minute with SOPS decryption

## Directory Structure

```
apps/
  base/
    mealie/
      kustomization.yaml
      namespace.yaml
      configmap.yaml
      deployment.yaml       # Mealie app
      postgres.yaml          # PostgreSQL deployment + service
      service.yaml           # Mealie service
      storage.yaml           # PVCs for Mealie and Postgres
  staging/
    mealie/
      kustomization.yaml     # Overlay: refs base, adds cloudflare + secrets + patch
      cloudflare.yaml        # Cloudflared deployment + ConfigMap
      secret.yaml            # SOPS-encrypted tunnel credentials
      postgres-secret.yaml   # SOPS-encrypted Postgres password
      node-patch.yaml        # nodeSelector + resource limits
    kustomization.yaml       # Updated to include mealie
```

## Base Manifests

### Namespace

`mealie`

### Mealie Deployment

- **Image:** `hkotel/mealie:v2.7.1` (adjust version)
- **Port:** 9000
- **Env:** ConfigMap for DB connection, Secret for Postgres password
- **Volume:** `/app/data` via PVC
- **Security context:** `runAsUser: 1000`, `runAsGroup: 1000`, `fsGroup: 1000`

### PostgreSQL Deployment

- **Image:** `postgres:17` (adjust version)
- **Port:** 5432
- **Env:** ConfigMap for `POSTGRES_USER`, `POSTGRES_DB`; Secret for `POSTGRES_PASSWORD`
- **Volume:** `/var/lib/postgresql/data` via PVC
- **Security context:** matching pattern

### Services

- `mealie` — ClusterIP, port 9000
- `mealie-postgres` — ClusterIP, port 5432

### Storage

- `mealie-data-pvc` — 5Gi (adjust size)
- `mealie-postgres-pvc` — 5Gi (adjust size)

### ConfigMap (`mealie-config`)

- `DB_ENGINE: postgres`
- `POSTGRES_SERVER: mealie-postgres`
- `POSTGRES_PORT: "5432"`
- `POSTGRES_USER: mealie`
- `POSTGRES_DB: mealie`

## Staging Overlay

### Node Patch

- `nodeSelector: storage: ssd` (worker node)
- Resource requests/limits for Mealie and PostgreSQL (placeholder — adjust)

### Cloudflare Tunnel

- Cloudflared deployment with `nodeSelector: storage: ssd`
- Tunnel config: `mealie.aldenvidal.com` -> `http://mealie:9000`
- Credentials from SOPS-encrypted `secret.yaml`
- Resource limits (placeholder — adjust)

### Secrets (user-created, SOPS-encrypted)

- `secret.yaml` — Cloudflare tunnel credentials
- `postgres-secret.yaml` — Postgres password

### Staging Kustomization

- References `../../base/mealie`
- Adds `secret.yaml`, `postgres-secret.yaml`, `cloudflare.yaml`
- Patches with `node-patch.yaml`
- Sets `namespace: mealie`

## FluxCD Integration

No Flux config changes needed. The existing `clusters/staging/apps.yaml` points to `./apps/staging` with SOPS decryption. Adding `mealie` to `apps/staging/kustomization.yaml` is sufficient.

## Items to Adjust Manually

- Mealie image version (currently `hkotel/mealie:v2.7.1`)
- PostgreSQL image version (currently `postgres:17`)
- Mealie data PVC size (currently 5Gi)
- PostgreSQL PVC size (currently 5Gi)
- Resource requests/limits for all containers
