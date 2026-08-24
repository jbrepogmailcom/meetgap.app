# Meetgap deployment

Meetgap runs as three Docker Compose services: the Vue frontend build, the Go
server, and MongoDB. Production and staging use separate Compose project names,
ports, databases, and volumes.

## Local development

The repository includes an ignored `server/.env` with safe local-only secrets.
Calendar sign-in remains disabled until OAuth client IDs are added.

```bash
docker compose up -d --build
docker compose ps
curl --fail http://localhost:3002/api/health
```

Open <http://localhost:3002>. Stop the stack with `docker compose down`. Do not
use `docker compose down -v` unless the local database may be deleted.

## Staging architecture

- Git branch: `staging`
- Public URL: `https://staging.meetgap.app`
- Host port: `127.0.0.1:3003`
- Compose project: `meetgap-staging`
- Deployment: automatic GitHub Actions run after each push to `staging`
- Releases: stored under `<deploy path>/releases/<git SHA>`

The deployment workflow is `.github/workflows/deploy-staging.yml`. It uploads a
release archive, builds it on the server with Docker Compose, and verifies
`/api/health`. GitHub shows the public staging URL on each successful deployment.

## One-time server setup

The SSH user needs Docker access and write permission to a dedicated absolute
directory ending in `/meetgap-staging`, for example:

```bash
sudo mkdir -p /opt/meetgap-staging
sudo chown "$USER":"$USER" /opt/meetgap-staging
docker version
docker compose version
```

Add the `staging.meetgap.app` DNS record pointing to the server, then add the
staging block from `Caddyfile.example` to Caddy and reload it. Caddy terminates
HTTPS and proxies only to the loopback-bound port 3003.

## GitHub staging secrets

Create a GitHub environment named `staging`, then add these environment secrets:

| Secret | Value |
| --- | --- |
| `STAGING_SSH_HOST` | Server hostname or IP |
| `STAGING_SSH_USER` | Deployment SSH user |
| `STAGING_SSH_PRIVATE_KEY` | Private key used only for deployment |
| `STAGING_SSH_KNOWN_HOSTS` | Pinned server host-key line from `ssh-keyscan` |
| `STAGING_DEPLOY_PATH` | Absolute path ending in `/meetgap-staging` |
| `STAGING_ENV_FILE` | Complete multiline server environment shown below |

Minimum `STAGING_ENV_FILE`:

```dotenv
CLIENT_ID=
CLIENT_SECRET=
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
ENCRYPTION_KEY=replace-with-exactly-16-24-or-32-characters
SESSION_SECRET=replace-with-at-least-32-random-characters
CORS_ORIGINS=https://staging.meetgap.app
BASE_URL=https://staging.meetgap.app
LISTMONK_ENABLED=false
```

Generate secrets with `openssl rand -hex 16` for `ENCRYPTION_KEY` and
`openssl rand -hex 32` for `SESSION_SECRET`. OAuth values can remain empty while
testing the core scheduling UI, but calendar integrations require provider-side
callback URLs for the staging domain.

## Production

Production uses the same `compose.yaml` with its own environment and defaults to
port 3002 and project name `meetgap`. It is intentionally not deployed on every
push. A production workflow should be enabled only after the staging setup and
OAuth callbacks have been verified.
