# Deployment Guide

## Pipeline Overview

```
PR opened
  └─► CI (tests, lint, format, clippy) ──► must pass to merge

Merge to main
  └─► Build & Push Docker image to GHCR
        └─► Deploy to Staging (automatic)
              └─► Deploy to Production (manual approval required)
```

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | Every PR + push to main | Tests, lint, format, clippy for all components |
| `security.yml` | PRs, push to main, weekly | cargo-audit, npm audit, Semgrep SAST, Trivy, Gitleaks |
| `build.yml` | Push to main / version tags | Build Docker image, push to GHCR |
| `deploy-staging.yml` | After successful build | Auto-deploy to staging |
| `deploy-production.yml` | Manual dispatch only | Deploy to production with approval gate |
| `rollback.yml` | Manual dispatch only | Roll back staging or production to any previous image |

## Required GitHub Secrets

### Staging
| Secret | Description |
|---|---|
| `STAGING_HOST` | SSH host for staging server |
| `STAGING_USER` | SSH username |
| `STAGING_SSH_KEY` | Private SSH key |
| `STAGING_DATABASE_URL` | PostgreSQL connection string |
| `STAGING_REDIS_URL` | Redis connection string |
| `STAGING_JWT_SECRET` | JWT signing secret |
| `STAGING_ANCHOR_WEBHOOK_SECRET` | Anchor webhook HMAC secret |
| `STAGING_SENTRY_DSN` | Sentry DSN for error tracking |

### Production
| Secret | Description |
|---|---|
| `PRODUCTION_HOST` | SSH host for production server |
| `PRODUCTION_USER` | SSH username |
| `PRODUCTION_SSH_KEY` | Private SSH key |
| `PRODUCTION_DATABASE_URL` | PostgreSQL connection string |
| `PRODUCTION_REDIS_URL` | Redis connection string |
| `PRODUCTION_JWT_SECRET` | JWT signing secret |
| `PRODUCTION_ANCHOR_WEBHOOK_SECRET` | Anchor webhook HMAC secret |
| `PRODUCTION_SANCTIONS_API_KEY` | Compliance sanctions API key |
| `PRODUCTION_SENTRY_DSN` | Sentry DSN for error tracking |

### Shared
| Secret | Description |
|---|---|
| `SLACK_WEBHOOK_URL` | Slack incoming webhook for deployment notifications |
| `SEMGREP_APP_TOKEN` | Semgrep Cloud token (optional, for dashboard) |

## Required GitHub Variables (non-secret)

| Variable | Example |
|---|---|
| `STAGING_URL` | `https://staging.blinks.app` |
| `PRODUCTION_URL` | `https://api.blinks.app` |

## GitHub Environment Setup

1. Go to **Settings → Environments** in your repository
2. Create `staging` environment — no protection rules needed
3. Create `production` environment:
   - Enable **Required reviewers** (add at least one approver)
   - Enable **Wait timer** (optional, e.g. 5 minutes)
   - Restrict to `main`/`master` branch only

## Deploying to Production

1. Confirm the staging deployment looks healthy
2. Note the image tag from the staging deploy (e.g. `sha-abc1234` or `v1.2.3`)
3. Go to **Actions → Deploy to Production → Run workflow**
4. Enter the image tag and type `deploy` in the confirm field
5. A required reviewer must approve the deployment in the GitHub UI
6. Monitor the health check step — auto-rollback fires if it fails

## Rolling Back

1. Go to **Actions → Rollback → Run workflow**
2. Select the environment (`staging` or `production`)
3. Enter the image tag to roll back to (check GHCR for available tags)
4. Provide a reason (logged in Slack notification)
5. For production rollbacks, the environment approval gate still applies

## Docker Images

Images are published to GitHub Container Registry:

```
ghcr.io/<org>/blinks-backend:<tag>
```

Tags produced on each push to main:
- `latest`
- `main`
- `sha-<short-sha>`

Tags produced on version tags (`v*.*.*`):
- `v1.2.3`
- `v1.2`
- `sha-<short-sha>`

## Database Migrations

Migrations run automatically before each deployment via `sqlx migrate run`.
The migration binary is baked into the Docker image at `/app/migrations/`.

To run migrations manually:
```bash
docker run --rm \
  -e BLINKS_DATABASE__URL="postgres://..." \
  ghcr.io/<org>/blinks-backend:latest \
  /app/blinks-backend migrate
```

## Monitoring

After deployment, verify:
- `GET /health` returns `200`
- `GET /ready` returns `200` (confirms DB connectivity)
- `GET /metrics` returns Prometheus metrics
- Grafana dashboard shows traffic and error rates normalizing
