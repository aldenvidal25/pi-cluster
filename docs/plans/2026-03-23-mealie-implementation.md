# Mealie Deployment Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Deploy Mealie with PostgreSQL to the avraham k3s cluster via FluxCD, exposed through a Cloudflare tunnel at `mealie.aldenvidal.com`.

**Architecture:** Kustomize base/overlay pattern mirroring the existing audiobookshelf deployment. Base holds reusable manifests (namespace, deployments, services, storage, configmap). Staging overlay adds node scheduling, resource limits, Cloudflare tunnel, and SOPS-encrypted secrets.

**Tech Stack:** k3s, FluxCD, Kustomize, Cloudflare Tunnels, SOPS/age, PostgreSQL

---

### Task 1: Create the base directory and namespace

**Files:**
- Create: `apps/base/mealie/namespace.yaml`

**Step 1: Create the directory**

```bash
mkdir -p apps/base/mealie
```

**Step 2: Create `apps/base/mealie/namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mealie
```

**Step 3: Commit**

```bash
git add apps/base/mealie/namespace.yaml
git commit -m "feat(mealie): add namespace"
```

---

### Task 2: Create the ConfigMap

**Files:**
- Create: `apps/base/mealie/configmap.yaml`

**Step 1: Create `apps/base/mealie/configmap.yaml`**

Mealie reads database config from environment variables. The ConfigMap provides all non-secret values.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mealie-config
data:
  DB_ENGINE: postgres
  POSTGRES_SERVER: mealie-postgres
  POSTGRES_PORT: "5432"
  POSTGRES_USER: mealie
  POSTGRES_DB: mealie
```

**Step 2: Commit**

```bash
git add apps/base/mealie/configmap.yaml
git commit -m "feat(mealie): add configmap for database connection"
```

---

### Task 3: Create the storage (PVCs)

**Files:**
- Create: `apps/base/mealie/storage.yaml`

**Step 1: Create `apps/base/mealie/storage.yaml`**

Two PVCs: one for Mealie's data directory (uploads, media) and one for PostgreSQL data. **Adjust sizes to your preference.**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mealie-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mealie-postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**Step 2: Commit**

```bash
git add apps/base/mealie/storage.yaml
git commit -m "feat(mealie): add PVCs for app data and postgres"
```

---

### Task 4: Create the PostgreSQL deployment and service

**Files:**
- Create: `apps/base/mealie/postgres.yaml`

**Step 1: Create `apps/base/mealie/postgres.yaml`**

This file contains both the Deployment and Service for PostgreSQL. The password comes from a Secret (created in the staging overlay). **Adjust the image version if needed.**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie-postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mealie-postgres
  template:
    metadata:
      labels:
        app: mealie-postgres
    spec:
      securityContext:
        fsGroup: 999
        runAsUser: 999
        runAsGroup: 999

      containers:
        - name: postgres
          image: postgres:17

          ports:
            - containerPort: 5432
              protocol: TCP

          envFrom:
            - configMapRef:
                name: mealie-config

          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mealie-postgres-secret
                  key: POSTGRES_PASSWORD

          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata

      restartPolicy: Always

      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: mealie-postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mealie-postgres
spec:
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: mealie-postgres
  type: ClusterIP
```

> **Note:** The `subPath: pgdata` on the volume mount is important — PostgreSQL requires the data directory to be a subdirectory when using PVCs, otherwise it fails on init because the mount point isn't empty.

> **Note:** PostgreSQL's official image runs as UID 999. The security context matches this. If you change the image, verify the expected UID.

**Step 2: Commit**

```bash
git add apps/base/mealie/postgres.yaml
git commit -m "feat(mealie): add postgres deployment and service"
```

---

### Task 5: Create the Mealie deployment

**Files:**
- Create: `apps/base/mealie/deployment.yaml`

**Step 1: Create `apps/base/mealie/deployment.yaml`**

Mealie connects to PostgreSQL using env vars from the ConfigMap and the password from the same Secret. **Adjust the image version if needed.**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mealie
  template:
    metadata:
      labels:
        app: mealie
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsGroup: 1000

      containers:
        - name: mealie
          image: hkotel/mealie:v2.7.1

          ports:
            - containerPort: 9000
              protocol: TCP

          envFrom:
            - configMapRef:
                name: mealie-config

          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mealie-postgres-secret
                  key: POSTGRES_PASSWORD

          volumeMounts:
            - name: mealie-data
              mountPath: /app/data

      restartPolicy: Always

      volumes:
        - name: mealie-data
          persistentVolumeClaim:
            claimName: mealie-data-pvc
```

**Step 2: Commit**

```bash
git add apps/base/mealie/deployment.yaml
git commit -m "feat(mealie): add mealie deployment"
```

---

### Task 6: Create the Mealie service

**Files:**
- Create: `apps/base/mealie/service.yaml`

**Step 1: Create `apps/base/mealie/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mealie
spec:
  ports:
    - port: 9000
      targetPort: 9000
  selector:
    app: mealie
  type: ClusterIP
```

**Step 2: Commit**

```bash
git add apps/base/mealie/service.yaml
git commit -m "feat(mealie): add mealie service"
```

---

### Task 7: Create the base Kustomization

**Files:**
- Create: `apps/base/mealie/kustomization.yaml`

**Step 1: Create `apps/base/mealie/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - configmap.yaml
  - storage.yaml
  - postgres.yaml
  - deployment.yaml
  - service.yaml
```

**Step 2: Validate the base builds correctly**

```bash
kubectl kustomize apps/base/mealie/
```

Expected: all 7 resources rendered (Namespace, ConfigMap, 2x PVC, 2x Deployment, 2x Service). You'll see a warning about the missing Secret — that's expected since it comes from the staging overlay.

**Step 3: Commit**

```bash
git add apps/base/mealie/kustomization.yaml
git commit -m "feat(mealie): add base kustomization"
```

---

### Task 8: Create the staging overlay — node patch

**Files:**
- Create: `apps/staging/mealie/node-patch.yaml`

**Step 1: Create the directory**

```bash
mkdir -p apps/staging/mealie
```

**Step 2: Create `apps/staging/mealie/node-patch.yaml`**

This patches both the Mealie and PostgreSQL deployments to schedule on the worker node and set resource limits. **Adjust resource values to your preference.**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie
spec:
  template:
    spec:
      nodeSelector:
        storage: ssd
      containers:
        - name: mealie
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie-postgres
spec:
  template:
    spec:
      nodeSelector:
        storage: ssd
      containers:
        - name: postgres
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

**Step 3: Commit**

```bash
git add apps/staging/mealie/node-patch.yaml
git commit -m "feat(mealie): add node selector and resource limits patch"
```

---

### Task 9: Create the Cloudflare tunnel

**Files:**
- Create: `apps/staging/mealie/cloudflare.yaml`

**Step 1: Create the Cloudflare tunnel**

Run this from a machine with `cloudflared` authenticated:

```bash
cloudflared tunnel create mealie
```

This outputs a tunnel ID and creates a credentials JSON file. You'll need both for the next steps.

**Step 2: Create `apps/staging/mealie/cloudflare.yaml`**

Replace `mealie` in the tunnel field with your actual tunnel name (or ID if different). This file contains both the Cloudflared Deployment and its ConfigMap.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  replicas: 1
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      nodeSelector:
        storage: ssd
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 50m
              memory: 128Mi
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config/config.yaml
            - run
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            failureThreshold: 1
            initialDelaySeconds: 10
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config
              readOnly: true
            - name: creds
              mountPath: /etc/cloudflared/creds
              readOnly: true
      volumes:
        - name: creds
          secret:
            secretName: tunnel-credentials
        - name: config
          configMap:
            name: cloudflared
            items:
              - key: config.yaml
                path: config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared
data:
  config.yaml: |
    tunnel: mealie
    credentials-file: /etc/cloudflared/creds/credentials.json
    metrics: 0.0.0.0:2000
    no-autoupdate: true

    ingress:
    - hostname: mealie.aldenvidal.com
      service: http://mealie:9000
    - service: http_status:404
```

**Step 3: Create the DNS route**

```bash
cloudflared tunnel route dns mealie mealie.aldenvidal.com
```

**Step 4: Commit**

```bash
git add apps/staging/mealie/cloudflare.yaml
git commit -m "feat(mealie): add cloudflare tunnel deployment"
```

---

### Task 10: Create the SOPS-encrypted secrets

**Files:**
- Create: `apps/staging/mealie/secret.yaml` (tunnel credentials)
- Create: `apps/staging/mealie/postgres-secret.yaml` (database password)

**Step 1: Create the tunnel credentials secret**

Create the plaintext secret first, then encrypt with SOPS. Use the credentials JSON from `cloudflared tunnel create`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-credentials
type: Opaque
stringData:
  credentials.json: |
    <paste your tunnel credentials JSON here>
```

Encrypt it:

```bash
sops --encrypt --in-place apps/staging/mealie/secret.yaml
```

**Step 2: Create the Postgres password secret**

Create plaintext, then encrypt:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mealie-postgres-secret
type: Opaque
stringData:
  POSTGRES_PASSWORD: <your-chosen-password>
```

Encrypt it:

```bash
sops --encrypt --in-place apps/staging/mealie/postgres-secret.yaml
```

**Step 3: Commit**

```bash
git add apps/staging/mealie/secret.yaml apps/staging/mealie/postgres-secret.yaml
git commit -m "feat(mealie): add SOPS-encrypted secrets"
```

---

### Task 11: Create the staging Kustomization and register the app

**Files:**
- Create: `apps/staging/mealie/kustomization.yaml`
- Modify: `apps/staging/kustomization.yaml`

**Step 1: Create `apps/staging/mealie/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: mealie
resources:
  - ../../base/mealie
  - secret.yaml
  - postgres-secret.yaml
  - cloudflare.yaml
patches:
  - path: node-patch.yaml
```

**Step 2: Update `apps/staging/kustomization.yaml`**

Add `mealie` to the resources list:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - linkding
  - audiobookshelf
  - mealie
```

**Step 3: Validate the full staging build**

```bash
kubectl kustomize apps/staging/mealie/
```

Expected: all resources rendered with `namespace: mealie` applied, node selectors patched in, secrets included.

**Step 4: Commit**

```bash
git add apps/staging/mealie/kustomization.yaml apps/staging/kustomization.yaml
git commit -m "feat(mealie): add staging overlay and register app"
```

---

### Task 12: Deploy and verify

**Step 1: Push to main**

```bash
git push origin main
```

FluxCD will detect the change within 1 minute.

**Step 2: Watch Flux reconciliation**

```bash
KUBECONFIG=kubeconfig kubectl get kustomization -n flux-system -w
```

Wait until the `apps` kustomization shows `Ready: True`.

**Step 3: Verify pods are running on the worker node**

```bash
KUBECONFIG=kubeconfig kubectl get pods -n mealie -o wide
```

Expected: `mealie`, `mealie-postgres`, and `cloudflared` pods all Running on node `metuselah-worker`.

**Step 4: Check Mealie logs**

```bash
KUBECONFIG=kubeconfig kubectl logs -n mealie deployment/mealie --tail=20
```

Look for successful startup and database connection messages.

**Step 5: Check PostgreSQL logs**

```bash
KUBECONFIG=kubeconfig kubectl logs -n mealie deployment/mealie-postgres --tail=20
```

Look for "database system is ready to accept connections".

**Step 6: Verify external access**

Open `https://mealie.aldenvidal.com` in your browser. You should see the Mealie setup page.

**Step 7: Verify PVCs are bound**

```bash
KUBECONFIG=kubeconfig kubectl get pvc -n mealie
```

Expected: both PVCs show `Bound` status.

---

## Summary of files to create

| File | Purpose |
|------|---------|
| `apps/base/mealie/namespace.yaml` | Namespace |
| `apps/base/mealie/configmap.yaml` | DB connection config |
| `apps/base/mealie/storage.yaml` | PVCs for app + postgres |
| `apps/base/mealie/postgres.yaml` | PostgreSQL deployment + service |
| `apps/base/mealie/deployment.yaml` | Mealie deployment |
| `apps/base/mealie/service.yaml` | Mealie service |
| `apps/base/mealie/kustomization.yaml` | Base kustomization |
| `apps/staging/mealie/node-patch.yaml` | Node selector + resource limits |
| `apps/staging/mealie/cloudflare.yaml` | Cloudflared deployment + config |
| `apps/staging/mealie/secret.yaml` | SOPS-encrypted tunnel creds |
| `apps/staging/mealie/postgres-secret.yaml` | SOPS-encrypted DB password |
| `apps/staging/mealie/kustomization.yaml` | Staging overlay kustomization |

| File | Purpose |
|------|---------|
| `apps/staging/kustomization.yaml` | Add `mealie` to resources list |

## Items to adjust before deploying

- Mealie image version in `apps/base/mealie/deployment.yaml` (placeholder: `hkotel/mealie:v2.7.1`)
- PostgreSQL image version in `apps/base/mealie/postgres.yaml` (placeholder: `postgres:17`)
- PVC sizes in `apps/base/mealie/storage.yaml` (placeholder: 5Gi each)
- Resource requests/limits in `apps/staging/mealie/node-patch.yaml`
