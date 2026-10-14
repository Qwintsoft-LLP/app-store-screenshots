# Coolify Production Deployment Guide

This document explains how to deploy the App Store & Google Play Screenshots Editor to **Coolify** on production servers.

---

## 1. Architecture Overview

- **Application Framework**: Next.js 16 (App Router) + React 19 + Tailwind CSS + Lucide
- **Runtime & Package Manager**: [Bun](https://bun.sh/) (`oven/bun:1-slim`)
- **Container Build**: Multi-stage Dockerfile (`deps` -> `builder` -> `runner`) running as a non-privileged `bun` user (UID 1000)
- **Networking**: Internal Docker container networking attached to Coolify's `coolify` network; reverse-proxied by Caddy
- **State & Storage**:
  - `app-store-screenshots.json`: Canvas and project configuration state (configured via `PROJECT_STATE_PATH`)
  - `public/screenshots/uploaded`: Uploaded screenshots
  - `public/fonts/imported`: Custom uploaded font files

---

## 2. Mandatory Files Summary

The following deployment files are maintained:
- `Dockerfile`: Multi-stage reproducible container build
- `docker-compose.yml`: Local development and standalone Docker testing
- `compose.coolify.yaml`: Production Coolify configuration (uses `expose:`, persistent volumes, and healthchecks)
- `.dockerignore`: Excludes cache, logs, git history, and local node_modules
- `.env.example`: Documented environment variable definitions with safe defaults
- `DEPLOY.md`: This deployment documentation

---

## 3. Environment Variables

| Variable | Default | Purpose |
| :--- | :--- | :--- |
| `NODE_ENV` | `production` | Enables Next.js production optimizations |
| `PORT` | `3000` | Internal container port (exposed to Coolify reverse proxy) |
| `HOSTNAME` | `0.0.0.0` | Listen address inside the container |
| `STORAGE_PATH` | `/data/coolify/applications/app-store-screenshots` | Host server root for persistent mounts |
| `PROJECT_STATE_PATH` | `/app/storage/app-store-screenshots.json` | Container path where project state JSON is stored |
| `NEXT_TELEMETRY_DISABLED` | `1` | Disables telemetry data collection |

---

## 4. Persistent Storage Configuration

The container filesystem is ephemeral. To ensure user projects and uploaded assets persist across container updates, mount persistent volumes in Coolify:

| Host Directory | Container Mount Path | Contents |
| :--- | :--- | :--- |
| `/data/coolify/applications/app-store-screenshots/uploads` | `/app/public/screenshots/uploaded` | Uploaded app screenshots |
| `/data/coolify/applications/app-store-screenshots/fonts` | `/app/public/fonts/imported` | Uploaded custom fonts |
| `/data/coolify/applications/app-store-screenshots/storage` | `/app/storage` | Persistent `app-store-screenshots.json` |

---

## 5. Health Check

The application exposes a lightweight readiness check endpoint at `/api/health`:

- **Path**: `/api/health`
- **Method**: `GET`
- **Expected Status**: `200 OK`
- **Command**:
  ```sh
  curl -f http://127.0.0.1:3000/api/health || exit 1
  ```
- **Interval**: 30s
- **Timeout**: 5s
- **Retries**: 3
- **Start Period**: 20s
- **Note**: `curl` and `ca-certificates` are installed in the container runner image to support Coolify's automated HTTP health checks.

---

## 6. Reverse Proxy & Domain Routing

- The container listens internally on port `3000` (`0.0.0.0:3000`).
- Coolify manages TLS termination and routing via Caddy.
- The application's `rejectCrossSiteWrite` security guard automatically recognizes `x-forwarded-proto` and `x-forwarded-host` headers sent by Coolify's reverse proxy so all save and upload operations succeed securely over HTTPS.

---

## 7. Step-by-Step Deployment in Coolify

1. **Select Server and Project**:
   - Server: `contabo-i2`
   - Project: `99 - R&D` (Environment: `production`)
2. **Add Resource**:
   - Choose **Public Repository** or **Private Repository (GitHub App)**.
   - Repository: `Qwintsoft-LLP/app-store-screenshots`
   - Branch: `main`
3. **Configure Build Settings**:
   - **Build Pack**: `Dockerfile`
   - **Dockerfile Location**: `/Dockerfile`
   - **Exposed Port**: `3000`
4. **Configure Storage Mounts**:
   - Add the persistent storage directories as listed in Section 4.
5. **Configure Domain**:
   - Set the desired domain / FQDN (e.g. `https://<uuid>.3.shreylink.in` or custom domain).
6. **Deploy**:
   - Click **Deploy**. Coolify builds the multi-stage image, mounts the volumes, starts the container, and verifies health status.
