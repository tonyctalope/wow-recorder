# WC-Recorder (Warcraft Recorder) — Developer Context

## What is this project?

An Electron + React (TypeScript) open-source app that records World of Warcraft gameplay. It auto-detects activities (raids, M+, arena, BGs) via combat log parsing and records them using OBS (via the `noobs` library).

The official app has a paid "Pro" tier powered by a Cloudflare Workers + R2 backend. We created a **self-hosted replacement server** (`wcr-server/`) that implements the same API protocol so the client works with our own infrastructure.

## Architecture overview

```
wc-recorder/              # Electron client (existing, open-source)
  src/
    main/                 # Electron main process (recording, config, IPC)
    renderer/             # React UI (video player, settings, etc.)
    storage/
      CloudClient.ts      # THE key file — all server communication lives here
      DiskClient.ts       # Local storage
      StorageClient.ts    # Interface
    config/configSchema.ts
    types/api.ts          # Zod schemas for API responses (TAffiliation, TChatMessageWithId)
    main/types.ts         # Core types (Metadata, CloudMetadata, CloudSignedMetadata, CloudConfig, etc.)
    utils/configUtils.ts  # Config accessors (getCloudConfig, etc.)

wcr-server/               # Self-hosted server (we created this)
  src/
    index.ts              # Express + HTTP server + WebSocket setup
    config.ts             # Env-based configuration
    db/
      schema.ts           # SQLite schema (8 tables)
      connection.ts       # better-sqlite3 singleton
    middleware/
      auth.ts             # HTTP Basic Auth with bcrypt
      guildAccess.ts      # Guild permission middleware (read/write/del)
    routes/
      user.ts             # GET /api/user/affiliations
      video.ts            # Video CRUD, upload (single + multipart), bulk ops
      guild.ts            # Usage, limit, mtime, housekeeper
      chat.ts             # Chat correlators + messages
      link.ts             # Shareable links + public HTML player page
    services/
      storage.ts          # MinIO/S3 operations (signed URLs, multipart, list, delete)
      websocket.ts        # WebSocket server — guild-scoped broadcast (key:value format)
      housekeeper.ts      # Auto-cleanup of oldest unprotected videos when over limit
    types/index.ts        # Server-side type definitions
    seed.ts               # CLI script to create users/guilds/affiliations
  docker-compose.yml      # 3 services: wcr-server, minio, minio-init
  Dockerfile
```

## What we changed in the client

All changes are minimal and backwards-compatible (empty `cloudServerUrl` = use official server):

1. **`src/main/types.ts`** — Added `cloudServerUrl: string` to `CloudConfig`
2. **`src/config/configSchema.ts`** — Added `cloudServerUrl` field (string, default: `''`)
3. **`src/utils/configUtils.ts`** — Added `cloudServerUrl` to `getCloudConfig()`
4. **`src/storage/CloudClient.ts`** — Made `api`, `poll`, `website` URLs instance properties instead of static. In `configure()`, if `cloudServerUrl` is set, derives all 3 URLs from it (e.g., `http://myserver:3000` → api: `.../api`, poll: `ws://myserver:3000/poll`, website: same base)
5. **`src/renderer/CloudSettings.tsx`** — Added "Server URL (optional)" input field in the Pro settings UI, wired to config

## API protocol (server must implement)

The client (`CloudClient.ts`) is the single source of truth. All endpoints use HTTP Basic Auth (`Authorization: Basic base64(user:pass)`).

### REST Endpoints (all under `/api`)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/user/affiliations` | List user's guilds + permissions |
| GET | `/api/guild/:guild/video` | List videos (response includes `signedVideoKey` = presigned GET URL) |
| POST | `/api/guild/:guild/video` | Create video metadata |
| POST | `/api/guild/:guild/bulk/delete` | Delete videos (body: `string[]`) |
| POST | `/api/guild/:guild/bulk/protect` | Protect/unprotect (body: `{videos, protect}`) |
| POST | `/api/guild/:guild/bulk/tag` | Tag videos (body: `{videos, tag}`) |
| POST | `/api/guild/:guild/upload` | Get signed PUT URL (body: `{key, bytes}`) |
| POST | `/api/guild/:guild/create-multipart-upload` | Init multipart (body: `{key, total, part}`) |
| POST | `/api/guild/:guild/complete-multipart-upload` | Complete multipart (body: `{key, etags}`) |
| GET | `/api/guild/:guild/video/:videoKey/size` | Get file size |
| GET | `/api/guild/:guild/usage` | Guild storage bytes used |
| GET | `/api/guild/:guild/limit` | Guild storage limit |
| GET | `/api/guild/:guild/mtime` | Last modification timestamp |
| POST | `/api/guild/:guild/housekeeper` | Trigger cleanup |
| POST | `/api/guild/:guild/video/:videoName/link` | Create shareable link → `{id}` |
| POST | `/api/guild/:guild/chat/:hash/:start` | Get/create chat correlator |
| POST | `/api/guild/:guild/named-chat/:name` | Get/create named correlator |
| GET | `/api/guild/:guild/chat/:correlator` | Get chat messages |
| POST | `/api/guild/:guild/chat/:correlator` | Post message (body: `{message}`) |
| DELETE | `/api/guild/:guild/chat/:id` | Delete message |

### WebSocket (`/poll?guild={guild}`)

- Auth via `Authorization` header on upgrade
- Ping/pong heartbeat (client pings every 60s)
- Messages: `key:value` string format
- Keys: `vc` (video create), `vd` (delete), `vp` (protect), `vu` (unprotect), `vt` (tag), `vm` (chat msg), `vmd` (chat delete), `gu` (usage), `gl` (limit), `mtime`

### Public route (no auth)

- `GET /link/:id` — Serves HTML page with video player for shareable links

## Server stack

- **Runtime**: Node.js + TypeScript + Express
- **Database**: SQLite via better-sqlite3 (single file, zero config)
- **Object storage**: MinIO (S3-compatible, self-hosted) via @aws-sdk/client-s3
- **WebSocket**: ws library
- **Auth**: bcrypt for password hashing, HTTP Basic Auth
- **Deploy**: Docker Compose (3 services)

## Key design decisions

- **MinIO is mandatory** — The client uploads/downloads directly to signed S3 URLs. A proxy approach would bottleneck all video traffic through the Node.js server. MinIO must be reachable from the client machine (exposed on port 9000).
- **`MINIO_PUBLIC_URL`** — Critical env var. Signed URLs use this hostname. Must be the IP/domain clients can reach, not the Docker-internal `minio` hostname.
- **Multipart upload state** — Stored in-memory (Map). Fine for single-instance server.
- **Two S3 clients** — `internalClient` (uses Docker hostname for server-side ops) and `publicClient` (uses `MINIO_PUBLIC_URL` for signing client-facing URLs).

## Running the server

```bash
cd wcr-server
cp .env.example .env          # Set MINIO_PUBLIC_URL to your machine's IP
docker compose up -d
SEED_USER=myuser SEED_PASS=mypass SEED_GUILD=MyGuild npm run seed
```

Then in the WCR app: Settings → Pro → Server URL: `http://<your-ip>:3000`

## What's NOT implemented yet

- **Video Chat** (non-bold feature) — endpoints exist but the video chat overlay feature is Pro-only UI
- **Custom Overlays** — Pro feature, not in scope
- **Access Control UI** — Server supports permissions but there's no admin panel to manage them (use `seed.ts` or direct SQLite)
- **Browser Webapp** — Only the shareable link page exists, not a full browsing webapp
- **HTTPS/TLS** — Server runs plain HTTP. Put behind a reverse proxy (nginx/caddy) for production.
