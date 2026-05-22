# HanswerAi config

Externalized Spring configuration served by Spring Cloud Config Server.

## Files

| File | Served as |
|---|---|
| `hanswer-application-dev.yml` | `hanswer-application` app, `dev` profile |
| `hanswer-application-prod.yml` | `hanswer-application` app, `prod` profile |

Secrets are **not** stored here. Use `${ENV_VAR}` placeholders and provide values through GitHub environment secrets and the VM `/opt/deploy/.env` file.

## Local validation

```bash
python3 - <<'PY'
import yaml, pathlib
for path in pathlib.Path('.').glob('*.yml'):
    yaml.safe_load(path.read_text())
    print('OK', path)
PY
```

## Runtime

Each Oracle VM runs a `config-server` container. The backend imports this repo through:

```yaml
spring:
  config:
    import: optional:configserver:${CONFIG_SERVER_URI:}
```

Restarting `config-server` + `backend` pulls the latest config from this repo.
