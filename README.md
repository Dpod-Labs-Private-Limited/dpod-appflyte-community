# dpod-appflyte-community

Docker Compose deployment for the DPOD (Appflyte) community stack. A single
`docker compose up` brings up the frontend, auth/authorization, backend +
Celery, the bug tracker, the analytics subsystem, the analytics MCP server, and
all supporting infrastructure (DynamoDB, MongoDB, MinIO, Redis, RabbitMQ) behind
a Caddy reverse proxy on `http://localhost:8080`.

## Architecture

Everything is fronted by **Caddy** on port `8080`. Most services get their own
hostname; the analytics API lives under a **path** on the shared `api.localhost`
host:

| URL                                              | Routes to                    |
| ------------------------------------------------ | ---------------------------- |
| `http://localhost:8080`                          | Frontend SPA (appflyte-web)  |
| `http://auth.localhost:8080`                     | Authentication service       |
| `http://backend.localhost:8080`                  | Backend API                  |
| `http://bugtracker.localhost:8080`               | Bug tracker                  |
| `http://api.localhost:8080/ameya/analytics/v1`   | Analytics API                |
| `http://minio.localhost:8080`                    | MinIO S3 API                 |
| `http://mongoui.localhost:8080`                  | Mongo Express (DB UI)        |
| `http://dynamoadmin.localhost:8080`              | DynamoDB Admin UI            |

The browser must resolve `*.localhost` to `127.0.0.1`. Chrome and Firefox do
this automatically; **Safari does not** — see [Host resolution](#host-resolution).

### Services

- **Application** — `appflyte-web-service` (frontend), `appflyte-authentication-service`,
  `appflyte-authorization-service` (gRPC), `appflyte-backend-service`,
  `appflyte-celery-worker`, `appflyte-bug-tracker`.
- **Analytics** — `analytics-api`, `analytics-celery-beat`,
  `analytics-celery-worker`, `analytics-mongo-server`, `analytics-mongo-consumer`,
  `analytics-rabbitmq`.
- **MCP** — `mcp-server`, `mcp-consumer` (analytics MCP for the chatbot).
- **Infrastructure** — `dynamodb` (+ `dynamodb-admin`, `dynamodb-init`),
  `mongodb` (+ `mongo-express`), `minio`, `appflyte-redis`, `appflyte-rabbitmq`,
  `caddy`.

## Prerequisites

- **Docker** and **Docker Compose v2** (`docker compose`, not `docker-compose`).
- LLM API key(s):
  - `OPENAI_API_KEY` — the analytics subsystem uses OpenAI embeddings
    (`text-embedding-3-small`), so this is required for analytics.
  - `GEMINI_API_KEY` — used by the analytics chatbot / MCP LLM features.
- License credentials for DPOD (`LICENSE_TOKEN`, `LICENSE_PUBLIC_KEY`,
  `ROOT_USER_TOKEN`).

> The `dpod-labs-private-limited` images are published on a **public** GHCR
> registry — no `docker login` is needed to pull them.

## Deployment

### 1. Configure environment

Copy the example env file and fill in real values:

```bash
cp .env.example .env
```

Set the following in `.env`:

| Variable             | Description                                        |
| -------------------- | -------------------------------------------------- |
| `LICENSE_TOKEN`      | DPOD license token                                 |
| `LICENSE_PUBLIC_KEY` | DPOD license public key                            |
| `ROOT_USER_TOKEN`    | Root/admin user bootstrap token                    |
| `GEMINI_API_KEY`     | Google Gemini API key (analytics chatbot LLM)      |
| `OPENAI_API_KEY`     | OpenAI API key (analytics embeddings)              |

> Infrastructure credentials (MinIO, DynamoDB, Mongo, RabbitMQ) are baked into
> `docker-compose.yaml` with local-development defaults — change them before any
> non-local use.

### 2. Create the external network

The stack attaches to an **external** Docker network named
`appflyte_community_network`. Compose will not create it for you — create it once:

```bash
docker network create appflyte_community_network
```

### 3. Start the stack

```bash
docker compose up -d
```

On first run this pulls all images, then `dynamodb-init` runs once to create
the DynamoDB table and the MinIO `dpod-aws-s3` bucket. Services start in
dependency order and wait on healthchecks — note the backend and analytics API
have long `start_period`s (up to ~8 min), so the full stack can take several
minutes to report healthy.

Watch progress with:

```bash
docker compose ps
docker compose logs -f
```

### 4. Verify

Open **http://localhost:8080** in Chrome or Firefox. If the SPA loads and can
reach the API, the deployment is up.

## Host resolution

Chrome and Firefox resolve `*.localhost` → `127.0.0.1` automatically (RFC 6761).
**Safari and some other clients do not.** Add these entries to `/etc/hosts`:

```
127.0.0.1  auth.localhost backend.localhost api.localhost bugtracker.localhost
127.0.0.1  minio.localhost mongoui.localhost dynamoadmin.localhost
```

## Common operations

```bash
# View logs for one service
docker compose logs -f appflyte-backend-service

# Restart a single service
docker compose restart analytics-api

# Pull updated images and recreate
docker compose pull && docker compose up -d

# Stop the stack (keeps volumes/data)
docker compose down

# Stop AND delete all data volumes (full reset)
docker compose down -v
```

## Configuration notes

- **`api.localhost` path routing** — The analytics API is reached via
  `api.localhost` split by path in the `Caddyfile`: `/ameya/analytics/v1/*` →
  `analytics-api`. Paths are preserved (no stripping); the upstream mounts under
  its own prefix.
- **MinIO / presigned URLs** — The backend presigns S3 URLs against
  `http://minio.localhost:8080` (`MINIO_PUBLIC_ENDPOINT`). S3 SigV4 signs the
  Host header, so this host must match exactly between the browser and the
  backend; do not change it without updating both the endpoint config and the
  Caddy route. Internally, services reach MinIO directly at `http://minio:9000`.
- **CORS** — The SPA calls the API hosts cross-origin. Caddy adds uniform CORS
  headers on the API hosts (see `Caddyfile`); no per-service CORS config needed.
- **Persistent data** — All stateful data lives in named volumes
  (`mongodb_data`, `dynamodb-data`, `minio-data`, `redis-data`,
  `rabbitmq-data`, etc.). `docker compose down` preserves them; `-v` deletes them.
- **Changing the `8080` port** — Port `8080` is embedded in the browser-facing
  URLs throughout the env anchors. To run on another host port, remap only the
  host side of the Caddy `ports:` (e.g. `"9090:8080"`) and change the
  `*.localhost:8080` / `localhost:8080` URLs to the new port. Caddy matches
  sites by hostname regardless of port, so the `Caddyfile` itself needs no change.

## Troubleshooting

| Symptom                                    | Likely cause / fix                                                             |
| ------------------------------------------ | ------------------------------------------------------------------------------ |
| `network appflyte_community_network not found`| Run step 2: `docker network create appflyte_community_network`.             |
| `no matching manifest for linux/arm64`     | Images are amd64-only. Run under emulation on Apple Silicon: `DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose up -d`. |
| SPA loads but API calls fail in Safari     | Add the `*.localhost` entries to `/etc/hosts` (see Host resolution).           |
| Analytics endpoints error out              | Missing/invalid `OPENAI_API_KEY`, or `analytics-api` still warming up (long start_period). |
| Chatbot replies *"I encountered an issue while processing your request. Please try again."* | `GEMINI_API_KEY` is invalid or its usage/quota limit is exhausted. Verify the key and check your Gemini quota; then `docker compose restart` the affected service. |
| A service is stuck `unhealthy`             | Inspect its logs; dependents wait on healthchecks and won't start.             |
