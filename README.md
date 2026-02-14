> [!IMPORTANT]
> If you just want the TLDR take a look at [stoat-compose.yml](./stoat-compose.yml)

If you're here looking for quick answers and don't care about reading, take a look at `stoat-configs`.

If you want an ansible example check out `ansible-example-role`.

# Self hosting [stoat](https://stoat.chat) with voice and video support

- [Issues and PRs to keep track of voice and video progress](#issues-and-prs-to-keep-track-of-voice-and-video-progress)
- [Quick Reference: Credentials to Generate](#quick-reference-credentials-to-generate)
  * [Generating LiveKit Keys](#generating-livekit-keys)
- [Architecture Overview](#architecture-overview)
  * [The Three Layers](#the-three-layers)
  * [Configuration Split: Revolt.toml vs Environment Variables](#configuration-split-revolttoml-vs-environment-variables)
- [Docker Networking](#docker-networking)
- [Voice/Video - The Non-Obvious Stuff](#voicevideo---the-non-obvious-stuff)
- [LiveKit](#livekit)
  * [The Patched Web Client](#the-patched-web-client)
  * [Node Configuration: The lat/lon Requirement](#node-configuration-the-latlon-requirement)
  * [Node Naming Must Match](#node-naming-must-match)
  * [Port Requirements](#port-requirements)
- [MinIO S3 Storage - Virtual Host Addressing](#minio-s3-storage---virtual-host-addressing)
- [Service Dependencies](#service-dependencies)
- [Common Gotchas](#common-gotchas)
  * [EMPTY CACHE AND HARD RELOAD](#empty-cache-and-hard-reload)
  * ["missing field `lat`" Panic](#missing-field-lat-panic)
  * [Voice/Video Buttons Missing](#voicevideo-buttons-missing)
  * [Files Not Uploading](#files-not-uploading)
  * [WebSocket Connection Failures](#websocket-connection-failures)
  * [Services Can't Find Each Other](#services-cant-find-each-other)
  * [API Starts Then Crashes](#api-starts-then-crashes)
- [Internal Service Ports](#internal-service-ports)
- [The Caddy Configuration](#the-caddy-configuration)
- [Using Your Own Reverse Proxy](#using-your-own-reverse-proxy)
  * [Critical: WebSocket Support](#critical-websocket-support)
  * [Route Summary](#route-summary)

<!-- tocstop -->

Stoat is currently in a transitionary period, therefore the
documentation for self hosting with voice and video support
is non-existent, and community patches to the front end have
been made to enable the features.

There is also no official docker image for the web front end yet.

We'll be using the image graciously provided by [baptisterajaut](https://github.com/baptisterajaut) which uses
the fork by [LordGuenni](https://github.com/LordGuenni/for-web)

## Issues and PRs to keep track of voice and video progress

- [#176](https://github.com/stoatchat/self-hosted/issues/176) - Seems to be the main discussion thread right now.
- [#313](https://github.com/stoatchat/stoatchat/issues/313) - Progress tracker for voice/video.
- [PR for dockerizing web front end](https://github.com/Flash1232/for-web/pull/3)
- [PR to the official self hosted guide for setting up livekit](https://github.com/Flash1232/self-hosted/pull/2)

## Quick Reference: Credentials to Generate

Before deploying, generate these credentials. All values marked `CHANGE_ME_*` in config files must be replaced.

| Credential | Used In | How to Generate |
|------------|---------|-----------------|
| **LiveKit API Key & Secret** | Revolt.toml, livekit.yaml, compose | `docker run --rm --tmpfs /output livekit/generate --local` |
| **MinIO User & Password** | Revolt.toml (`access_key_id`/`secret_access_key`), compose | `openssl rand -hex 24` |
| **RabbitMQ Password** | Revolt.toml, compose | `openssl rand -hex 24` |
| **File Encryption Key** | Revolt.toml (`files.encryption_key`) | `openssl rand -base64 32` |
| **VAPID Keys** | Revolt.toml (`pushd.vapid`) | `KEY_PEM=$(openssl ecparam -genkey -name prime256v1 -noout) && for OPTION in "" -pubout; do openssl ec -in <(echo "${KEY_PEM}") ${OPTION}; done` |

### Generating LiveKit Keys

```bash
docker run --rm livekit/generate --local
```

## Architecture Overview

Stoat runs as 14 interconnected services. Understanding how they communicate saves debugging time.

### The Three Layers

**Infrastructure Layer** - Standard backing services:
- **MongoDB** (`stoat-database`) - Primary data store
- **Redis/KeyDB** (`stoat-redis`) - Caching, sessions, and pub/sub for LiveKit
- **RabbitMQ** (`stoat-rabbit`) - Async message queue for notifications and events
- **MinIO** (`stoat-minio`) - S3-compatible storage for file uploads
- **LiveKit** (`stoat-livekit-server`) - WebRTC server for voice/video

**Application Layer** - The actual Stoat services:
- **API** (`stoat-api`) - REST API, the main backend
- **Events** (`stoat-events`) - WebSocket server for real-time updates
- **Autumn** (`stoat-autumn`) - File upload/download service
- **January** (`stoat-january`) - URL metadata extraction and image proxying
- **Gifbox** (`stoat-gifbox`) - Tenor GIF proxy
- **Crond** (`stoat-crond`) - Scheduled tasks
- **Pushd** (`stoat-pushd`) - Push notification delivery
- **Voice Ingress** (`stoat-voice-ingress`) - Voice call handling

**Frontend Layer**:
- **Web** (`stoat-web`) - The web client
- **Caddy** (`stoat-caddy`) - Reverse proxy that routes everything

### Configuration Split: Revolt.toml vs Environment Variables

This is important to understand:

**Revolt.toml** is read by all the Stoat backend services (API, Events, Autumn, January, Gifbox, Crond, Pushd, Voice-Ingress). It contains:
- Database connection strings
- Service URLs (how services find each other)
- Feature flags and limits
- S3/MinIO credentials
- LiveKit node configuration
- Push notification settings

**Environment variables** are used by:
- Infrastructure services (MongoDB, Redis, RabbitMQ, MinIO, LiveKit)
- The web client (it doesn't read Revolt.toml at all)

The web client needs its own set of environment variables because it runs in the browser - it can't read server-side config files:
```
REVOLT_PUBLIC_URL  - Where to find the API
VITE_WS_URL        - WebSocket endpoint
VITE_MEDIA_URL     - File server (Autumn)
VITE_PROXY_URL     - Metadata proxy (January)
```
## Docker Networking

All Stoat containers **must** be on the same Docker network. This is how services find each other.

Docker provides internal DNS resolution - when containers are on the same network, they can reach each other by container name. This is why the configs use hostnames like `stoat-database`, `stoat-redis`, `stoat-api`, etc. Docker resolves these names to the container's internal IP.

For example, in `Revolt.toml`:
```toml
[database]
mongodb = "mongodb://stoat-database"  # Resolves via Docker DNS
redis = "redis://stoat-redis/"
```

If services can't find each other, the first thing to check is network membership:
```bash
docker network inspect stoat_network
```

All 14 containers should be listed. If something's missing, it won't be able to resolve other container names.


## Voice/Video - The Non-Obvious Stuff

Getting voice and video working required the most debugging. Here's what you need to know:

## LiveKit

Stoat moved to using [LiveKit](https://livekit.io/) to handle voice and video streams. We just use the official livekit image in our setup.

### The Patched Web Client

The standard Stoat/Revolt web client **does not include voice/video support**. The LiveKit integration exists in the backend, but the official web client doesn't have the UI for it (yet).

`baptisterajaut/stoatchat-web:dev` is a community-patched image that adds the voice/video UI. Without this, you'll have a working LiveKit server that nothing can connect to.

### Node Configuration: The lat/lon Requirement

The API will panic with `missing field 'lat'` if you don't include geographic coordinates in your LiveKit node config:

```toml
[api.livekit.nodes.worldwide]
url = "http://stoat-livekit-server:7880"
lat = 0.0    # Required!
lon = 0.0    # Required!
key = "..."
secret = "..."
```

### Node Naming Must Match

This one's subtle. The node name in `[hosts.livekit]` must match the section name in `[api.livekit.nodes.*]`:

```toml
[hosts.livekit]
worldwide = "wss://your-domain/livekit"  # "worldwide" is the node name

[api.livekit.nodes.worldwide]  # Must match "worldwide"
url = "http://stoat-livekit-server:7880"
# ...
```

If these don't match, voice/video calls will fail silently or with confusing errors.

### Port Requirements

LiveKit needs three types of network access:

- **7880/tcp** (internal only) - HTTP API, used by Stoat services
- **7881/tcp** (external) - WebRTC signaling over TCP
- **50000-50100/udp** (external) - Actual media streams

The UDP range is where voice/video data flows. If these ports are blocked, calls will either fail or fall back to TCP (higher latency). Make sure your firewall allows this UDP range inbound.

## MinIO S3 Storage - Virtual Host Addressing

MinIO needs a bunch of network aliases:
```yaml
aliases:
  - minio
  - revolt-uploads.minio
  - attachments.minio
  - avatars.minio
  - backgrounds.minio
  - icons.minio
  - banners.minio
  - emojis.minio
```

Why? MinIO supports virtual-host style bucket addressing where `bucketname.minio` routes to the `bucketname` bucket. Stoat's file server (Autumn) uses this pattern to access different buckets for different file types.

Without these aliases, file uploads will fail with connection errors because `avatars.minio` won't resolve to anything.

The actual bucket created is just `revolt-uploads` - the aliases are for routing, not for creating separate buckets.

## Service Dependencies

Understanding startup order helps with debugging:

1. **MongoDB, Redis, RabbitMQ, MinIO** - No dependencies, start first
2. **MinIO bucket creation** - Runs once after MinIO is healthy
3. **LiveKit** - Needs Redis for coordination
4. **All Stoat services** - Need MongoDB, RabbitMQ, and Revolt.toml mounted
5. **Caddy** - Needs all backend services ready to proxy to

Health checks matter here. MongoDB and RabbitMQ have health checks that other services wait on. If your deployment is flaky on startup, check that health checks are passing.

## Common Gotchas

### EMPTY CACHE AND HARD RELOAD

Many times during debugging I chased red herrings, when all I had to do was empty cache and hard reload on my browser.

### "missing field `lat`" Panic
Add `lat = 0.0` and `lon = 0.0` to your `[api.livekit.nodes.*]` config. See LiveKit section above.

### Voice/Video Buttons Missing
You're using the wrong web client image. Use `baptisterajaut/stoatchat-web:dev` instead of the official image.

### Files Not Uploading
1. Check MinIO is running and the `revolt-uploads` bucket exists
2. Verify MinIO has all the network aliases configured
3. Check Revolt.toml S3 credentials match MinIO's MINIO_ROOT_USER/PASSWORD

### WebSocket Connection Failures
The `/ws` path needs proper WebSocket upgrade handling. Caddy handles this by default, but if you're using a different reverse proxy, make sure it's configured for WebSocket.

### Services Can't Find Each Other
All services must be on the same Docker network. Check `docker network ls` and ensure everything's connected to your stoat network.

### API Starts Then Crashes
Usually a Revolt.toml syntax error or missing required field. Check container logs - TOML parse errors are usually descriptive.

## Internal Service Ports

For reference, these are the ports services listen on inside the Docker network:

| Service | Port |
|---------|------|
| API | 14702 |
| Events (WebSocket) | 14703 |
| Autumn (files) | 14704 |
| January (proxy) | 14705 |
| Gifbox | 14706 |
| Web | 5000 |
| LiveKit | 7880 |
| MongoDB | 27017 |
| Redis | 6379 |
| RabbitMQ | 5672 |
| MinIO | 9000 |

You shouldn't need to expose these externally - Caddy proxies everything through a single port.

## The Caddy Configuration

Caddy acts as the single entry point, routing requests to the correct backend service based on URL path. Here's what each route does:

You can refer to the [Example Caddyfile](./stoat-configs/Caddyfile.example)

```
/api/*     → stoat-api:14702      (REST API)
/livekit/* → stoat-livekit:7880   (WebRTC signaling for voice/video)
/ws        → stoat-events:14703   (WebSocket for real-time updates)
/autumn/*  → stoat-autumn:14704   (File uploads and downloads)
/january/* → stoat-january:14705  (URL metadata and image proxy)
/gifbox/*  → stoat-gifbox:14706   (Tenor GIF proxy)
/*         → stoat-web:5000       (Web frontend - default/fallback)
```

The `uri strip_prefix` directive is important - it removes the path prefix before forwarding. So a request to `/api/users` becomes `/users` when it hits the API server. The backends expect paths without the prefix.

## Using Your Own Reverse Proxy

If you already have nginx, Traefik, or another reverse proxy, you can skip Caddy entirely. You just need to replicate the routing.

OR

You can just bind the caddy container on a different port on your host and forward request from your main proxy to it.

### Critical: WebSocket Support

Two paths require WebSocket upgrades:
- `/ws` - Real-time events
- `/livekit/*` - Voice/video signaling

Make sure your proxy can handle this.

### Route Summary

Whatever proxy you use, configure these routes:

| Path | Backend | Notes |
|------|---------|-------|
| `/api/*` | stoat-api:14702 | Strip `/api` prefix |
| `/ws` | stoat-events:14703 | WebSocket upgrade required |
| `/livekit/*` | stoat-livekit-server:7880 | WebSocket upgrade required, strip `/livekit` prefix |
| `/autumn/*` | stoat-autumn:14704 | Strip `/autumn` prefix |
| `/january/*` | stoat-january:14705 | Strip `/january` prefix |
| `/gifbox/*` | stoat-gifbox:14706 | Strip `/gifbox` prefix |
| `/*` | stoat-web:5000 | Default fallback |
