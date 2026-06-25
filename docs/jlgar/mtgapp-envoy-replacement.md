# MtgApp Envoy Replacement Plan

## Platform Summary

MtgApp is a Django-based mortgage workflow platform for residential loan applications, Form 1003 workflows, borrower data, income calculations, credit report extraction, MISMO 3.4 import/export, and broker/originator administration.

The backend runs Django 5.1 on Python 3.11 with PostgreSQL 15 as the system of record and Redis 7 as the Django cache backend. Runtime configuration is loaded from environment variables, with staging and production values sourced from AWS SSM Parameter Store and AWS Secrets Manager.

The application surface includes:

- `/users/login/` and `/users/logout/`
- `/applications/`, `/applications/dashboard/`, and `/applications/system-admin/`
- `/form-1003/`
- `/income-calculation/`
- `/credit-reports/`
- `/api/internal/backup-database/`
- `/api/internal/purge-deleted-loans/`
- `/health/`

## Current Edge Architecture

Development currently runs PostgreSQL, Redis, and nginx in Docker while Django runs directly on the WSL host through `./runserver.sh`. Django binds to `0.0.0.0:8000`, and nginx listens on `127.0.0.1:80` before proxying to Django.

Staging is Docker Compose based on a Debian host. The stack includes `web`, `nginx`, `postgres`, and `redis`. The nginx container exposes `80` and `443`, mounts Cloudflare Origin Certificates from the host, and proxies to the Gunicorn web container.

Production runs on AWS ECS Fargate behind an ALB with ACM TLS certificates. The production image is built from `platform/Dockerfile`, static assets are collected at Docker build time, and Gunicorn logs to stdout/stderr.

## Replacement Goal

The goal is to replace nginx with Envoy across development, staging, and eventually production-adjacent workflows while preserving current application behavior.

Envoy must preserve:

- HTTP listener behavior on port `80`.
- HTTPS/TLS termination behavior in staging where nginx currently uses Cloudflare Origin Certificates.
- Proxying to Gunicorn on `0.0.0.0:8000` or the equivalent Docker service endpoint.
- `/health/` behavior for local checks, staging checks, and production ALB health checks.
- Static and media path behavior for `/static/` and `/media/`.
- Forwarded headers required by Django, including `Host`, `X-Forwarded-For`, and `X-Forwarded-Proto`.
- Secure proxy behavior expected by Django settings.
- Docker stdout/stderr logging behavior.
- CI/CD compatibility and rollback to nginx during the transition.

## Development Envoy Shape

The local development replacement runs Envoy natively in WSL and proxies to Django on `127.0.0.1:8000`.

Current local config:

- Envoy config: `configs/jlgar/local-django-gunicorn.yaml`
- Listener: `0.0.0.0:80`
- Admin: `127.0.0.1:9901`
- Upstream cluster: `django_dev`
- Upstream endpoint: `127.0.0.1:8000`
- Body limit equivalent: 50 MiB through `envoy.filters.http.buffer`

The local validation path is:

- Stop the nginx development container.
- Start Django through `./runserver.sh`.
- Start Envoy with `configs/jlgar/local-django-gunicorn.yaml`.
- Validate the UI through `http://127.0.0.1/`.
- Check Envoy stats through `http://127.0.0.1:9901/stats`.

## Staging Envoy Shape

The staging replacement should be introduced in parallel before removing nginx from the deployment path.

Initial staging target:

- Add an Envoy container to the Docker Compose stack.
- Keep nginx available for rollback until Envoy has passed route, TLS, static/media, and health checks.
- Run Envoy on an alternate host port first if needed.
- Proxy from Envoy to the existing `web` Gunicorn container over the Docker network.
- Preserve Cloudflare Origin Certificate usage or explicitly move TLS termination according to the staging cutover plan.

The target request path is:

```text
Browser -> Cloudflare -> Envoy container -> Gunicorn/Django web container -> PostgreSQL/Redis
```

Rollback path:

```text
Browser -> Cloudflare -> nginx container -> Gunicorn/Django web container -> PostgreSQL/Redis
```

## Production Considerations

Production currently uses ALB as the public edge. Envoy should not be inserted into production without a separate architecture decision.

Production replacement options include:

- ALB -> Envoy sidecar/container -> Gunicorn/Django
- ALB -> Envoy service -> Django service
- Keep ALB as public TLS edge and use Envoy for internal routing, policy, observability, or custom filters

Any production Envoy design must preserve ALB health checks, ACM TLS behavior, ECS logging, Secrets Manager/SSM injection, RDS connectivity, ElastiCache connectivity, and CloudWatch observability.

## Required Staging Inputs

Before writing the staging Envoy config, collect:

- Current staging nginx config.
- Docker Compose service names, networks, and exposed ports.
- Cloudflare Origin Certificate mount paths.
- Hostnames and TLS assumptions for `staging.easymloapp.com`.
- Current `web` service name and Gunicorn port.
- Current static/media behavior in staging.
- CI/CD upload paths and remote compose restart commands.

## Implementation Rules

- Do not change MtgApp secrets handling.
- Do not place staging or production secrets in this repository.
- Keep Envoy configs under `configs/jlgar/` or `distribution/jlgar/`.
- Keep Docker/deployment artifacts under `distribution/jlgar/`.
- Preserve rollback to nginx until Envoy is validated in staging.
- Prefer read-only discovery before modifying staging deployment files.
- Avoid blocking Envoy worker threads in any custom C++ filters.
- Use Proxy-Wasm for dynamically loaded policy experiments when recompiling Envoy is not desirable.
