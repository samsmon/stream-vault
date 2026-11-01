# StreamVault Service Specification & Homelab Integration

Container specification and homelab integration documentation for the `stream-vault` service. This document is ready to be added to, or referenced from, the main homelab service inventory (`docs/services.md`).

---

## 📋 Service Overview

| Attribute | Detail |
|---|---|
| **Service Name** | StreamVault |
| **Container Name** | `stream-vault` |
| **Docker Image** | `stream-vault-stream-vault` |
| **Host / Target** | LXC `docker-host` (Proxmox LXC 4 vCPU / 12GB RAM) |
| **Port Mapping** | `8090:8090` |
| **Healthcheck URL** | `http://localhost:8090/api/health` |
| **Restart Policy** | `unless-stopped` |
| **Default Creds** | `admin` / `admin` (Change immediately after deployment) |

---

## ⚡ Resource Philosophy & Architecture

- **Video/Audio Playback**: **100% Direct Play / Zero Server-Side Transcode**. The Node.js backend acts purely as an RFC 7233 HTTP 206 Byte-Range streaming origin. Server CPU stays at 0% for video playback transcoding. `.ass` subtitles are rendered 100% on the browser client side via WebAssembly (JASSUB/libass).
- **Server Subprocess Helpers**:
  FFmpeg & FFprobe are used **only** for:
  1. **Single-frame thumbnail snapshot** (`ffmpeg -frames:v 1`).
  2. **Softsub extraction stream copying** (`ffmpeg -c:s copy` without re-encoding).
  3. **Embedded subtitle metadata probing** (`ffprobe -select_streams s`).
- **Resource & Zombie Process Guard**:
  - **Init Process Daemon (`tini` / `init: true`)**: Runs as PID 1 inside the container to automatically reap the exit status of all dead child processes, preventing zombie / defunct processes.
  - **PID Table Limiter (`pids_limit: 100`)**: Caps the container cgroup at 100 processes. Prevents fork bombs or PID slot exhaustion on the LXC host (the 25k tasks incident cannot happen again).
  - **Process Queue / Concurrency Limiter**: At most 1 concurrent FFmpeg/FFprobe child process (`FFMPEG_MAX_CONCURRENCY=1`).
  - **Hard Timeout**: 15 seconds for thumbnails & ffprobe, 30 seconds for subtitle copy. Processes are force-killed (`SIGKILL`) when they exceed the limit.
  - **Single Thread Flag**: `-threads 1` limits decoder threads so memory buffers are not multiplied.
  - **No Font Demuxing**: `-sn -an -dn` prevents FFmpeg from reading dozens of `.ttf` font attachments in MKV containers.
  - **Negative Cache & Circuit Breaker**: Broken/failed files automatically get a permanent SVG placeholder and are blacklisted from further FFmpeg calls.
  - **Memory Safeguard Check**: The process is aborted if remaining system memory is `< 45MB`.

---

## 💾 Homelab Resource Allocation Recommendations

| Parameter | Minimum | Homelab Recommendation (Production) | Notes |
|---|---|---|---|
| **Container RAM (`limits.memory`)** | 256 MB | **512 MB** | 75MB Node.js runtime + 1 single-thread FFmpeg worker (100–180MB) + OS buffer |
| **RAM Reservation (`reservations.memory`)** | 64 MB | **128 MB** | Guarantees a baseline allocation that other LXC processes cannot take away |
| **PIDs Limit (`pids_limit`)** | 50 | **100** | Protects the host kernel from fork exhaustion |
| **CPU Limit** | 1 Core | **2 Cores** (Shared) | Used briefly (<1s) during initial thumbnail frame extraction |

---

## 📂 Volume & Storage Mounts

```yaml
volumes:
  # Media Storage (Must be :ro / Read-Only to protect the collection)
  - /mnt/storage/media:/media:ro

  # Persistent Cache (Prevents thumbnail regeneration & subtitle re-extraction on container restart)
  - stream-vault-cache:/app/.cache

  # Persistent Data (Local user database, folder customization, bookmarks)
  - stream-vault-data:/app/data
```

---

## 🔧 Key Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8090` | Fastify server listening port |
| `MEDIA_ROOT` | `/media` | Path of the mounted media directory |
| `CACHE_DIR` | `/app/.cache` | Cache directory for thumbnail frames (`thumbnails/`) and subtitles (`subtitles/`) |
| `DATA_DIR` | `/app/data` | Persistent data directory (folder metadata, sqlite/json store) |
| `FFMPEG_MAX_CONCURRENCY` | `1` | Limit on parallel FFmpeg/FFprobe processes (`1` recommended for a 512MB container) |
| `FFMPEG_TIMEOUT_MS` | `15000` | Hard child process timeout in milliseconds |
| `AUTH_ENABLED` | `true` | Enable user authentication |
| `OIDC_ENABLED` | `false` | SSO integration (Authelia, Authentik, Keycloak) |

---

## 🚀 Homelab Operational Commands

```bash
# Start/update container
cd /root/stream-vault
docker compose up -d --build

# Check health & resource usage
curl -s http://localhost:8090/api/health | jq .

# Check runtime logs
docker logs -f stream-vault --tail 100

# Clear the thumbnail cache if needed
docker compose exec stream-vault rm -rf /app/.cache/thumbnails/*
```
