# StreamingApp — Minikube Kubernetes Setup

This document covers everything needed to deploy the StreamingApp on a local Minikube cluster, the changes made compared to Docker Compose, the ingress routing decisions, and the key differences between Docker and Kubernetes hosting.


---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

---

## Folder Structure

```
kubernetes/
  configmap.yaml          — non-sensitive environment variables shared across all backend services
  secrets.yaml            — sensitive values (JWT_SECRET, AWS credentials)
  namespace.yaml          — isolates all resources under the "streamingapp" namespace
  ingress.yaml            — two ingress objects for routing (explained below)
  mongo/
    pv.yaml               — PersistentVolume using hostPath on the Minikube node
    pvc.yaml              — PersistentVolumeClaim bound to the PV
    statefulset.yaml      — MongoDB StatefulSet with readinessProbe
    service.yaml          — Headless ClusterIP service for MongoDB (mongo-service)
  auth/
    deployment.yaml       — 2 replicas, envFrom configmap + secret
    service.yaml          — ClusterIP on port 3001 (auth-service)
  admin/
    deployment.yaml       — 2 replicas, envFrom configmap + secret
    service.yaml          — ClusterIP on port 3003 (admin-service)
  streaming/
    deployment.yaml       — 2 replicas, envFrom configmap + secret
    service.yaml          — ClusterIP on port 3002 (streaming-service)
  chat/
    deployment.yaml       — 2 replicas, envFrom configmap + secret
    service.yaml          — ClusterIP on port 3004 (chat-service)
  frontend/
    deployment.yaml       — 3 replicas, no envFrom (REACT_APP_* baked at build time)
    service.yaml          — ClusterIP on port 80 (frontend-service)
```

---

## Step 1 — Start Minikube and Enable Ingress

```bash
minikube start
minikube addons enable ingress
```

The ingress addon installs the nginx ingress controller inside the cluster with the class name `nginx`.

---

## Step 2 — Point Shell to Minikube's Docker Daemon

```bash
eval $(minikube docker-env)
```

This redirects all `docker` commands in the current shell to Minikube's internal Docker daemon. Images built after this command are available to Kubernetes pods. To revert back to Mac Docker Desktop, run `eval $(minikube docker-env -u)`.

> This must be run in every new terminal session before building images for Minikube.

---

## Step 3 — Build All Images

Run from the project root after `eval $(minikube docker-env)`.

```bash
# Backend services
docker build -t streamingapp-auth:latest ./backend/authService
docker build -t streamingapp-streaming:latest -f backend/streamingService/Dockerfile ./backend
docker build -t streamingapp-admin:latest     -f backend/adminService/Dockerfile ./backend
docker build -t streamingapp-chat:latest      -f backend/chatService/Dockerfile ./backend

# Frontend — REACT_APP_* vars are baked into the JS bundle at build time
# URLs point to the Ingress paths, not direct service ports
docker build \
  --build-arg REACT_APP_AUTH_API_URL=http://localhost/api/auth \
  --build-arg REACT_APP_STREAMING_API_URL=http://localhost/api \
  --build-arg REACT_APP_STREAMING_PUBLIC_URL=http://localhost \
  --build-arg REACT_APP_ADMIN_API_URL=http://localhost/api/admin \
  --build-arg REACT_APP_CHAT_API_URL=http://localhost/api/chat \
  --build-arg REACT_APP_CHAT_SOCKET_URL=http://localhost \
  -t streamingapp-frontend:minikube \
  ./frontend
```

> The frontend image is tagged `minikube` to distinguish it from the `latest` tag used by Docker Compose. The `frontend/deployment.yaml` references `streamingapp-frontend:minikube`.

### Why No Port Numbers in the Frontend Build Args

In Docker Compose each service is directly exposed on its own port on your Mac. The frontend talks to each service independently:

```
http://localhost:3001/api       ← auth service, port open on Mac
http://localhost:3002/api       ← streaming service, port open on Mac
http://localhost:3003/api/admin ← admin service, port open on Mac
http://localhost:3004/api/chat  ← chat service, port open on Mac
http://localhost:3004            ← Socket.IO direct to chat service
```

In Minikube all services are `ClusterIP` — they have no external port at all. They only exist inside the cluster's private network. The only thing exposed to your Mac is the Ingress on port `80` via `minikube tunnel`:

```
http://localhost:3001  ← nothing listening, port not exposed
http://localhost:3002  ← nothing listening, port not exposed
http://localhost:3003  ← nothing listening, port not exposed
http://localhost:3004  ← nothing listening, port not exposed
http://localhost       ← only this works, Ingress on port 80
```

The Ingress uses the **URL path** instead of a port number to route to the correct backend service:

```
http://localhost/api/auth      → Ingress → auth-service:3001      (inside cluster)
http://localhost/api/streaming → Ingress → streaming-service:3002  (inside cluster)
http://localhost/api/admin     → Ingress → admin-service:3003      (inside cluster)
http://localhost/api/chat      → Ingress → chat-service:3004       (inside cluster)
http://localhost/socket.io     → Ingress → chat-service:3004       (inside cluster)
```

Docker Compose uses **ports** to separate services. Kubernetes Ingress uses **URL paths** to separate services. That is why port numbers are removed from the frontend build args when moving to Minikube.

---

## Step 4 — Apply Manifests

Apply in this order — namespace and config first, storage second, workloads last.

```bash
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/configmap.yaml
kubectl apply -f kubernetes/secrets.yaml
kubectl apply -f kubernetes/mongo/
kubectl apply -f kubernetes/auth/
kubectl apply -f kubernetes/admin/
kubectl apply -f kubernetes/streaming/
kubectl apply -f kubernetes/chat/
kubectl apply -f kubernetes/frontend/
kubectl apply -f kubernetes/ingress.yaml
```

---

## Step 5 — Verify the Cluster

```bash
kubectl get all -n streamingapp
kubectl get ingress -n streamingapp
```

All pods should show `1/1 Running`. Both ingress objects should have `192.168.49.2` in the ADDRESS column.

---

## Step 6 — Start the Tunnel and Access the App

The tunnel must run as a foreground process in its own dedicated terminal. Do not suspend it with Ctrl+Z.

```bash
minikube tunnel
```

Open `http://localhost` in your browser.

---

## Step 7 — Promote a User to Admin

To access the Admin Dashboard and upload videos, promote a registered user to admin directly in MongoDB:

```bash
kubectl exec -n streamingapp mongo-0 -- mongosh --quiet --eval \
  "db.getSiblingDB('streamingapp').users.updateOne(
    {email:'your-email@example.com'},
    {\$set:{role:'admin'}}
  )"
```

Then log out and log back in so a new JWT token is issued with the `admin` role.

---

## ConfigMap vs Secrets — What Goes Where

### ConfigMap (`configmap.yaml`)
Non-sensitive environment variables shared by all backend services.

| Key | Value | Notes |
|---|---|---|
| `MONGO_URI` | `mongodb://mongo-service:27017/streamingapp` | Uses K8s service name, not `localhost` |
| `AWS_REGION` | `us-west-1` | Must match your S3 bucket region |
| `AWS_S3_BUCKET` | `dineshb14-streamingapp` | Your S3 bucket name |
| `AWS_CDN_URL` | _(empty)_ | Optional — falls back to direct S3 URL |
| `CLIENT_URLS` | `http://localhost` | CORS allowed origin — Ingress serves on port 80 |
| `STREAMING_PUBLIC_URL` | `http://localhost` | Used to build video stream URLs |
| `NODE_ENV` | `development` | Controls Node.js behaviour |

### Secrets (`secrets.yaml`)
Sensitive values stored as base64-encoded strings.

| Key | Notes |
|---|---|
| `JWT_SECRET` | Used by all backend services to sign and verify JWT tokens. Must be identical across all services. |
| `AWS_ACCESS_KEY_ID` | IAM access key for S3 uploads |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key for S3 uploads |

To encode a value:
```bash
echo -n "your-value-here" | base64
```

> Never commit real credentials to Git. Add `kubernetes/secrets.yaml` to `.gitignore`.

---

## Ingress — Routing Rules and Why Two Objects

### The Problem

All backend services mount their routes internally under `/api`:

| Service | Internal route prefix |
|---|---|
| auth | `/api/login`, `/api/register` |
| admin | `/api/admin/...` |
| streaming | `/api/streaming/...` |
| chat | `/api/chat/...` |

The frontend was built with these base URLs:

| Variable | Value baked into frontend image |
|---|---|
| `REACT_APP_AUTH_API_URL` | `http://localhost/api/auth` |
| `REACT_APP_ADMIN_API_URL` | `http://localhost/api/admin` |
| `REACT_APP_STREAMING_API_URL` | `http://localhost/api/streaming` |
| `REACT_APP_CHAT_API_URL` | `http://localhost/api/chat` |

When the frontend calls `POST /login`, the full URL becomes `http://localhost/api/auth/login`. The Ingress receives `/api/auth/login` and must forward it to the auth service. But the auth service only knows `/api/login` — it has no `/auth` segment internally. This causes a 404.

Admin, streaming and chat do not have this problem because their internal paths already include their service name (`/api/admin/...`, `/api/streaming/...`, `/api/chat/...`).

### The Solution — Two Ingress Objects

The nginx ingress controller applies the `rewrite-target` annotation to **all paths** in an Ingress object — you cannot set different rewrites per path in a single object. The solution is two separate Ingress objects:

**`streamingapp-ingress-auth`** — auth only, with path rewrite:
```
Browser: /api/auth/login
  → $2 captures: login
  → rewrite-target /api/$2 → /api/login
  → auth pod receives: /api/login  ✅
```

**`streamingapp-ingress`** — all other services, no rewrite:
```
Browser: /api/admin/videos  → admin pod receives: /api/admin/videos  ✅
Browser: /api/streaming/... → streaming pod receives: /api/streaming/... ✅
Browser: /api/chat/...      → chat pod receives: /api/chat/... ✅
Browser: /                  → frontend nginx serves React app ✅
```

### Additional Annotations Explained

| Annotation | Value | Reason |
|---|---|---|
| `proxy-read-timeout` | `3600` | Prevents WebSocket (chat) connections from being dropped after nginx's default 60s idle timeout |
| `proxy-send-timeout` | `3600` | Same — keeps long-running upload and stream connections alive |
| `proxy-body-size` | `2048m` | Allows video uploads up to 2GB. nginx default is 1MB which rejects any video file. Initially set to 500m but increased after a 603MB upload was rejected with HTTP 413 |

### Socket.IO WebSocket Routing

Socket.IO (used by the chat service) connects to `http://localhost/socket.io/...`. Without an explicit rule this matches the catch-all `/` path and gets routed to the frontend nginx pod which returns 404.

A dedicated `/socket.io` path rule is added before the `/` catch-all in `streamingapp-ingress` pointing to `chat-service:3004`. Path order matters — more specific paths must come before the catch-all `/`:

```
http://localhost/socket.io/... → chat-service:3004  ✅
http://localhost/              → frontend-service:80 ✅
```

The `proxy-read-timeout: 3600` and `proxy-send-timeout: 3600` annotations handle WebSocket keepalive — nginx automatically passes `Upgrade` and `Connection` headers for WebSocket connections when these timeouts are set.

---

## MongoDB — PV, PVC and StatefulSet

MongoDB uses a `StatefulSet` instead of a `Deployment` because it requires a stable network identity and persistent storage.

- **PersistentVolume (pv.yaml)** — cluster-scoped resource that provisions 5Gi of storage using `hostPath` on the Minikube node at `/data/mongo`
- **PersistentVolumeClaim (pvc.yaml)** — namespace-scoped request for storage, bound to the PV above
- **StatefulSet** — mounts the PVC via a `volumes` block (not `volumeClaimTemplates`) so the same PVC is reused across pod restarts
- **readinessProbe** — performs a TCP check on port 27017 every 10 seconds. Kubernetes only marks the pod Ready after the probe succeeds, preventing backend services from crashing while MongoDB is still initialising
- **Headless service** (`clusterIP: None`) — required by StatefulSets for stable DNS. The pod is reachable at `mongo-0.mongo-service.streamingapp.svc.cluster.local`

---

## Key Changes from Docker Compose to Minikube

### Environment Variables

| Docker Compose | Minikube |
|---|---|
| Read from root `.env` file at runtime | Injected via `envFrom: configMapRef` and `envFrom: secretRef` in each deployment |
| `MONGO_URI: mongodb://mongo:27017/...` — uses compose service name | `MONGO_URI: mongodb://mongo-service:27017/...` — uses K8s service name |
| `CLIENT_URLS: http://localhost:3000` — frontend served on port 3000 | `CLIENT_URLS: http://localhost` — Ingress serves everything on port 80 |
| `.env` files in backend service folders used for local dev | Not needed in K8s — all env vars come from ConfigMap and Secret |

### Networking

| Docker Compose | Minikube |
|---|---|
| Each service exposed on its own port (3000, 3001, 3002...) | All services are `ClusterIP` — not exposed externally |
| Frontend on `localhost:3000`, auth on `localhost:3001` etc. | Single entry point at `http://localhost` via Ingress |
| Services communicate using compose service names (`mongo`, `auth`) | Services communicate using K8s service names (`mongo-service`, `auth-service`) |
| No routing rules needed | Ingress rules route by URL path to the correct backend service |

### Images

| Docker Compose | Minikube |
|---|---|
| `docker-compose up --build` builds and runs in one step | Images must be built into Minikube's Docker daemon with `eval $(minikube docker-env)` first |
| Images live in Mac Docker Desktop daemon | Images live in Minikube's isolated internal daemon |
| Frontend `REACT_APP_*` vars passed as `args` in compose file | Frontend must be rebuilt with `--build-arg` flags pointing to Ingress URLs |
| `imagePullPolicy` defaults to `IfNotPresent` | Must set `imagePullPolicy: Never` — images are local, not in a registry |

### Storage

| Docker Compose | Minikube |
|---|---|
| Named volume `mongo-data` managed by Docker | PersistentVolume + PersistentVolumeClaim + `hostPath` on Minikube node |
| Volume created automatically | PV and PVC must be declared explicitly |

### Scaling

| Docker Compose | Minikube |
|---|---|
| Single instance of each service | Multiple replicas per service (auth: 2, admin: 2, streaming: 2, chat: 2, frontend: 3) |
| No health checks by default | `readinessProbe` on MongoDB ensures backends only connect when DB is ready |

---

## Useful Commands

```bash
# Check all resources in the namespace
kubectl get all -n streamingapp

# Check ingress routing
kubectl get ingress -n streamingapp

# View logs for a service
kubectl logs -n streamingapp -l app=auth --tail=50
kubectl logs -n streamingapp -l app=admin --tail=50

# Describe a pod for events and errors
kubectl describe pod -n streamingapp <pod-name>

# Get a shell inside a running pod
kubectl exec -it -n streamingapp <pod-name> -- sh

# Verify env vars are injected correctly
kubectl exec -n streamingapp <pod-name> -- env | grep -E "MONGO|JWT|AWS"

# Restart a deployment (e.g. after config change)
kubectl rollout restart deployment/auth -n streamingapp

# Delete and reapply everything
kubectl delete namespace streamingapp
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/
```

---

## Application Code Changes for Minikube

These changes were made to the application source code during Minikube deployment. They are not Kubernetes-specific configuration — they fix bugs that were hidden in Docker Compose because each service had its own port.

### `frontend/nginx.conf` — Added

New file created to fix React Router 404s. Without this, refreshing any page other than `/` returns 404 because nginx looks for a file matching the URL path.

### `frontend/Dockerfile` — Modified

Added `COPY nginx.conf /etc/nginx/conf.d/default.conf` to the production stage so the custom nginx config is included in the image.

### `backend/streamingService/controllers/streaming.controller.js` — Modified

Fixed `getThumbnail` to prepend `thumbnails/` to the S3 key when it is missing. The route wildcard `req.params[0]` only captures the filename after `/thumbnails/` in the URL, but the actual S3 object key always includes the `thumbnails/` folder prefix.

### `backend/adminService/.dockerignore` and `backend/chatService/.dockerignore` — Added

These files were missing. Without them the `.env` files in those service folders were being copied into the Docker images. Added with the same content as the other services:
```
.env
.env.*
node_modules
```

---

## Troubleshooting — Issues Encountered and Fixed

### 1. Login returning "An error occurred during login"

**Cause:** Path mismatch between the frontend and the auth service internal routes.

The frontend was built with `REACT_APP_AUTH_API_URL=http://localhost/api/auth`. When it calls `POST /login` the full URL becomes `/api/auth/login`. The Ingress forwarded this to the auth pod as-is, but the auth service mounts routes at `app.use('/api', router)` so it only knows `/api/login` — not `/api/auth/login`.

**Fix:** Split into two Ingress objects. `streamingapp-ingress-auth` uses `rewrite-target: /api/$2` with a regex capture group to strip `/auth` from the path before forwarding to the auth pod:
```
/api/auth/login  →  rewrite  →  /api/login  ✔
```

---

### 2. "Unable to load videos right now" on the browse page

**Cause:** `REACT_APP_STREAMING_API_URL` was baked as `http://localhost/api/streaming`. The `streaming.service.js` appends `/streaming/videos` to this base URL, resulting in `/api/streaming/streaming/videos` — `streaming` doubled.

**Fix:** Rebuild the frontend with `REACT_APP_STREAMING_API_URL=http://localhost/api` (without `/streaming`). The service already appends `/streaming/videos` itself so the base URL must stop at `/api`.

---

### 3. Chat panel stuck on "Connecting..."

**Cause:** Socket.IO connects to `http://localhost/socket.io/...`. No explicit Ingress rule existed for `/socket.io` so it matched the catch-all `/` rule and was routed to the frontend nginx pod, which returned 404.

**Fix:** Added a `/socket.io` path rule in `streamingapp-ingress` pointing to `chat-service:3004`, placed before the `/` catch-all rule.

---

### 4. Thumbnails not loading

**Cause:** The `thumbnailKey` stored in MongoDB is `thumbnails/filename.jpg`. The `buildPublicUrl` function in `streamingService/util/s3.js` builds the URL as `/api/streaming/thumbnails/thumbnails/filename.jpg` — `thumbnails/` doubled. The streaming controller's `getThumbnail` handler then received only `filename.jpg` as `req.params[0]` (the wildcard after `/thumbnails/` in the route) and passed that directly to S3 as the key, causing `NoSuchKey`.

**Fix:** Updated `getThumbnail` in `streaming.controller.js` to prepend `thumbnails/` back to the key before calling S3:
```js
const key = filename.startsWith('thumbnails/')
  ? filename
  : `thumbnails/${filename}`;
```

---

### 5. Video upload failing with HTTP 413

**Cause:** The `proxy-body-size` annotation was set to `500m` but the video file was 603MB. nginx rejected the request before it reached the admin pod.

**Fix:** Increased `proxy-body-size` to `2048m` in both Ingress objects.

---

### 6. React Router routes returning 404 (e.g. /browse, /login)

**Cause:** The frontend Dockerfile had no custom nginx config. By default nginx tries to serve `/browse` as a static file — it doesn't exist so it returns 404. React is a Single Page Application — all routes must serve `index.html` and let React Router handle navigation client-side.

**Fix:** Added `frontend/nginx.conf` with `try_files $uri $uri/ /index.html` and updated the Dockerfile to copy it into the nginx container:
```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

---

## Teardown

```bash
# Stop the tunnel (Ctrl+C in the tunnel terminal)

# Delete all resources
kubectl delete namespace streamingapp

# Stop Minikube
minikube stop

# Optional — delete the Minikube cluster entirely
minikube delete
```
