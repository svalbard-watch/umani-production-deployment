# umani-production-deployment

Production deployment of [Umami](https://umami.is) web analytics (Umami + PostgreSQL) as a single Docker Swarm stack on Dokploy. Same layout and conventions as `swi-production-deployment`, minus the staging and pre-prod branches: **`main` is production**.

## The stack

| Service | Image | Notes |
|---|---|---|
| `umami` | `ghcr.io/umami-software/umami` | Routed by Traefik at `UMAMI_HOST`, port 3000, healthcheck on `/api/heartbeat` |
| `db` | `postgres:17-alpine` | Internal network only, data in the `umami_db_data` volume |

Image tags are pinned exactly in [compose.yml](compose.yml). Both containers drop all capabilities and run with a read-only root filesystem.

## Configuration (environment variables)

Set these in Dokploy (Environment tab). [.env.example](.env.example) is the template for local runs.

| Variable | Required | Meaning |
|---|---|---|
| `UMAMI_HOST` | yes | Public hostname for the Traefik router |
| `UMAMI_APP_SECRET` | yes | Random secret for signing sessions (`openssl rand -base64 32`) |
| `UMAMI_TWO_FACTOR_KEY` | yes | Encrypts stored 2FA secrets, exactly 64 hex characters (`openssl rand -hex 32`). Keep it stable |
| `UMAMI_DB_PWD` | yes | Postgres password, URL-safe characters only (`openssl rand -hex 24`) |
| `TRAEFIK_CERT_RESOLVER` | no | Defaults to `letsencrypt`; empty locally |

Real values live only in Dokploy, never in the repo. A missing required variable fails the deploy.

## Deploying on Dokploy

1. Create a **Compose** project from this repository, branch `main`, compose path `./compose.yml`, type **Docker Compose** with Swarm stack mode as for the SWI stack.
2. Add the variables above and point the DNS record of `UMAMI_HOST` at the server.
3. Enable auto-deploy on push, so merging to `main` redeploys.
4. First login: Umami creates `admin` / `umami`. **Change the password immediately.**

## Automatic updates

[update-versions.yml](.github/workflows/update-versions.yml) runs daily at 06:00 UTC (and on demand, with a dry-run option). It reads the real tag list from each registry and, for every image with a newer tag, opens or refreshes one PR into `main` from a `deps/<image>` branch, with the upstream release notes. The PR is never merged automatically.

- The [deployment smoke test](.github/workflows/deploy-smoke-test.yml) gates the PR: it boots the stack behind Traefik and requests Umami.
- Postgres only moves within its pinned major (see [.github/version-updater.yml](.github/version-updater.yml)); a major upgrade is a manual dump/restore.
- Actions are updated by Dependabot ([.github/dependabot.yml](.github/dependabot.yml)).

Repository settings this relies on: *Actions → Allow GitHub Actions to create and approve pull requests*, and a `main` branch rule requiring the smoke test.

## Security scanning

Same as the other SWI repos:

- [security.yml](.github/workflows/security.yml): shared workflow from `swi-shared-workflows` (secret scan, dependency audit, workflow lint), push, PR and weekly. Report-only for now.
- [scan-deployed-images.yml](.github/workflows/scan-deployed-images.yml): daily Trivy scan of every image in `compose.yml`, report-only (`ENFORCE: "false"`). Time-limited exceptions go in [.github/image-scan-exceptions.json](.github/image-scan-exceptions.json).

## Run it locally

```bash
docker network create dokploy-network
mkdir -p traefik-testing/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout traefik-testing/certs/local.key -out traefik-testing/certs/local.crt -subj "/CN=*.localhost"
docker compose -f traefik-testing/compose.yml up -d
cp .env.example .env     # then set real-looking secrets
docker compose -f compose.yml up -d
```

Umami is then at https://umami.localhost (self-signed certificate).

## Tests

```bash
pip install pyyaml pytest
python -m pytest tests
```
