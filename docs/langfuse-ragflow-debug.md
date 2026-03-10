# Langfuse ↔ RAGFlow Debug Log

Status: _Working_ (2026-03-10)  
Owners: infra team / whoever touches Langfuse integration next.

This file documents the end-to-end debugging we ran to make a single Langfuse deployment serve both **RAGFlow** and **LightRAG** on the same host. It captures the symptoms we saw, the root causes, and the fixes we applied so future changes don’t regress the setup.

---

## Environment Snapshot

- Host: single Linux box running multiple Docker Compose stacks.
- RAGFlow stack (`docker/docker-compose.yml`) already exposes the external bridge network `docker_ragflow` (10.201.0.0/16).
- Langfuse stack runs from `~/Downloads/source_code/langfuse/docker-compose.yml` with services: `langfuse-web`, `langfuse-worker`, Postgres, ClickHouse, Redis, MinIO.
- LightRAG stack uses Langfuse via env vars (`LANGFUSE_HOST`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`) but otherwise runs independently.

Goal: both RAGFlow and LightRAG send traces to one Langfuse instance while the host browser can still reach Langfuse at `http://localhost:3300`.

---

## Issue 1 – `ConnectError('[Errno 111] Connection refused')` on Langfuse auth

**Symptoms**
- Saving Langfuse keys in RAGFlow UI failed with `[Errno 111] Connection refused`.
- Inside `docker-ragflow-cpu-1`, `curl http://localhost:3300/api/public-key` failed.

**Root Cause**
- RAGFlow runs inside Docker, so `localhost` points to the container, not the host. Langfuse was running in a separate Compose project, inaccessible from that namespace.

**Fix**
1. Temporarily connected `docker-ragflow-cpu-1` to Langfuse’s network and used the fully qualified DNS name (`langfuse-langfuse-web-1.langfuse_langfuse_net`).
2. Permanent solution:
   - Declared `docker_ragflow` as an external network in `langfuse/docker-compose.yml`.
   - Attached `langfuse-web` to both `langfuse_net` and `docker_ragflow`.
   - Set `container_name: langfuse-web` and `HOSTNAME=0.0.0.0` so it listens on every interface.
3. Updated RAGFlow’s Langfuse Host to `http://langfuse-web:3300`.

**Verification**
- `docker exec docker-ragflow-cpu-1 curl -I http://langfuse-web:3300/api/public-key` returns HTTP 404 (expected).
- Langfuse auth check in RAGFlow UI succeeds.

---

## Issue 2 – Browser lost access to `http://localhost:3300`

**Symptoms**
- Chrome could no longer open Langfuse after we added extra networks.

**Root Cause**
- The Langfuse Next.js app only bound to its “primary” interface (on `langfuse_net`). After attaching `docker_ragflow`, Docker chose that interface for port publishing, so the `3300` port mapping pointed to an address without a listener.

**Fix**
- Set `HOSTNAME=0.0.0.0` in the `langfuse-web` service to force binding on all interfaces. (Already part of Issue 1’s permanent fix.)

**Verification**
- `curl -I http://localhost:3300` succeeds on the host.
- Browser loads Langfuse UI.

---

## Issue 3 – Redis hostname collision (`connect ECONNREFUSED …:6379`)

**Symptoms**
- `langfuse-web` log showed `Redis error connect ECONNREFUSED 10.201.0.5:6379`.
- That IP belongs to RAGFlow’s Redis service on the shared network.

**Root Cause**
- Both stacks export a service named `redis`. Docker DNS picked RAGFlow’s instance when Langfuse tried to resolve `redis`, so Langfuse attempted to auth against the wrong server.

**Fix**
- Added `container_name: langfuse-redis` plus a `langfuse_net` alias in Langfuse compose.
- Overrode `REDIS_HOST` in the shared env block (`REDIS_HOST=${REDIS_HOST:-langfuse-redis}`).

**Verification**
- `docker exec langfuse-web getent hosts langfuse-redis` resolves to 10.202.x.x.
- `langfuse-web` logs stop reporting Redis connection errors.

---

## Issue 4 – Traces still missing: `Failed to upload JSON to S3`

**Symptoms**
- RAGFlow log spam: `Failed to export span batch code: 500 ... Failed to upload JSON to S3`.
- `langfuse-web` showed `ECONNREFUSED 10.201.0.2:9000` when uploading to `events/otel/...`.

**Root Cause**
- Langfuse’s S3 endpoints were set to `http://localhost:9090` or `http://minio:9000`. After joining `docker_ragflow`, the hostname `minio` resolved to **RAGFlow’s** MinIO service (10.201.0.2), not Langfuse’s internal MinIO (10.202.0.3). Access was refused because the port isn’t exposed there.

**Fix**
- Renamed the Langfuse MinIO container to `langfuse-minio` and added a network alias.
- Updated all Langfuse S3 env vars (`LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT`, `LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT`, `LANGFUSE_S3_BATCH_EXPORT_ENDPOINT`) to `http://langfuse-minio:9000`.
- Kept `LANGFUSE_S3_BATCH_EXPORT_EXTERNAL_ENDPOINT` pointing to `http://localhost:9090` for browser downloads.

**Verification**
- `docker exec langfuse-web curl -sI http://langfuse-minio:9000/minio/health/live` returns 200.
- RAGFlow log no longer shows the S3 upload errors.
- New chats produce traces visible under Langfuse → Traces (filter `name ~ ragflow-`).

---

## Current Working Instructions

1. Bring Langfuse up:
   ```bash
   cd ~/Downloads/source_code/langfuse
   docker compose up -d langfuse-minio langfuse-redis langfuse-postgres langfuse-clickhouse langfuse-worker langfuse-web
   ```
2. Confirm networking:
   ```bash
   docker exec langfuse-web getent hosts langfuse-minio langfuse-redis
   docker exec docker-ragflow-cpu-1 curl -I http://langfuse-web:3300/api/public-key
   ```
3. In RAGFlow UI → Settings → Langfuse Configuration:
   - Host: `http://langfuse-web:3300`
   - Public / Secret keys: your Langfuse project keys.
4. In LightRAG `.env`:
   ```env
   LANGFUSE_HOST=http://langfuse-web:3300
   LANGFUSE_PUBLIC_KEY=pk-...
   LANGFUSE_SECRET_KEY=sk-...
   ```
   Restart LightRAG if the container was already running.
5. Run a chat in each app and verify new traces appear in Langfuse.

---

## Lessons Learned / Future Guardrails

- When multiple stacks coexist, give every cross-stack-facing service a unique container name or alias to prevent DNS collisions (`langfuse-web`, `langfuse-redis`, `langfuse-minio`, etc.).
- Any host service that combines browser + container access must bind to `0.0.0.0`; otherwise port mappings break when new networks are added.
- Prefer referencing dependencies via their unique service names instead of `localhost` so the configuration survives multi-network deployments.
- Keep this doc updated if we change ports, networks, or add new consumers (e.g., more apps that emit traces).

