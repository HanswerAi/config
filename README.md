# HanswerAi config

Externalized Spring configuration served by Spring Cloud Config Server.

## Files

| File                                | Served as                                          |
| ----------------------------------- | -------------------------------------------------- |
| `interviewmate-application-dev.yml` | `interviewmate-application` app, `dev` profile     |
| `interviewmate-application-prod.yml`| `interviewmate-application` app, `prod` profile    |

Secrets are **never** stored here. Every sensitive value uses `${ENV_VAR}` placeholders and is supplied via the per-VM `/opt/deploy/.env`, which is materialized from GitHub Environment secrets at deploy time (see [infra/scripts/migrate-env-to-github-secrets.sh](../infra/scripts/migrate-env-to-github-secrets.sh)).

## Required environment variables (dev & prod)

| Variable                  | Purpose                                                                | Source            |
| ------------------------- | ---------------------------------------------------------------------- | ----------------- |
| `DATABASE_URL`            | JDBC URL incl. host / port / db                                        | Supabase project  |
| `DB_USER` / `DB_PASSWORD` | Postgres user (pooler creds for Supabase)                              | Supabase project  |
| `JWT_SECRET`              | HS256 signing secret (≥256 bits)                                       | Random            |
| `JWT_REFRESH_SECRET`      | Refresh token signing secret                                           | Random            |
| `SUPABASE_URL`            | `https://<ref>.supabase.co`                                            | Supabase project  |
| `SUPABASE_ANON_KEY`       | Browser-safe anon JWT                                                  | Supabase project  |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-only service-role JWT                                         | Supabase project  |
| `SUPABASE_JWT_ISSUER_URI` | `${SUPABASE_URL}/auth/v1`                                              | Supabase project  |
| `REDIS_HOST_PWD`          | Upstash Redis password (host is in `interviewmate-application-*.yml`)   | Upstash console   |
| `GOOGLE_OAUTH_CLIENT_ID`  | OAuth client id                                                        | Google Cloud      |
| `DEEPGRAM_API_KEY`        | STT/TTS key                                                            | Deepgram console  |
| `OPENAI_API_KEY`          | LLM key                                                                | OpenAI dashboard  |
| `STRIPE_SECRET_KEY`       | Stripe billing                                                         | Stripe dashboard  |
| `STRIPE_WEBHOOK_SECRET`   | Stripe webhook signing                                                 | Stripe dashboard  |
| `APP_PUBLIC_WEB_URL`      | Browser-facing app URL (e.g. `https://dev.hanswerai.com`)              | Cloudflare Pages  |
| `CORS_ORIGIN`             | Allowed CORS origin (usually same as web URL)                          | Set explicitly    |

Optional (presence enables the feature):

- `OPENROUTER_API_KEY`, `MINIMAX_API_KEY`, `DEEPSEEK_API_KEY`, `GROQ_API_KEY`
- `POSTHOG_API_KEY` + `POSTHOG_HOST` (must set host explicitly — no region default)
- `LOGZIO_LOGS_TOKEN`, `LOGZIO_METRICS_TOKEN`, `LOGZIO_TRACES_TOKEN`
- `OPENAI_BASE_URL` for Azure OpenAI / OpenAI-compatible proxies
- `DEEPGRAM_API_ENDPOINT`, `DEEPGRAM_WS_ENDPOINT`, `DEEPGRAM_TTS_ENDPOINT` for regional Deepgram routing

## Local validation

```bash
python3 - <<'PY'
import yaml, pathlib
for path in pathlib.Path('.').glob('*.yml'):
    yaml.safe_load_all(path.read_text())
    print('OK', path)
PY
```

The `safe_load_all` form is required because each file uses YAML document separators (`---`) for profile-specific overrides (`posthog`, `logzio`).

## Runtime

Each Oracle VM runs a `config-server` container. The backend imports this repo through:

```yaml
spring:
  config:
    import: optional:configserver:${CONFIG_SERVER_URI:}
```

The Config Server is configured to point at this repo's `main` branch (see `infra/deploy/oracle/*/config-server.env`). Restarting `config-server` + `backend` pulls the latest config from this repo.

## Adding a new external dependency

When the backend introduces a new external dependency (e.g. Anthropic API, Cloudflare R2), the workflow is:

1. Add the property to `application.yml` in the backend repo as `${NEW_VAR:}` (empty default — no hardcoded URL/key).
2. Add the property here in both `dev.yml` and `prod.yml`, with a sensible default for the URL (region-aware where applicable) and `${NEW_VAR}` (no default) for the secret.
3. Add the env var to `infra/scripts/secret-routing.yml` so the migration script knows where to push it.
4. The CI guard `no-hardcoded-urls.yml` will fail if the literal URL leaks back into source.
