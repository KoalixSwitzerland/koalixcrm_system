# Configuration

This document catalogs all runtime and deploy-time configuration values in koalixCRM.
These are values that an operator or administrator can change without modifying source code —
primarily through environment variables read at process startup or through the Docker
entrypoint shell. Configuration is consumed by three distinct runtime components:

- The Django application process (`projectsettings/settings/`)
- The Celery / SQS microservice worker (`koalixcrm_microservices/celery_app.py`)
- The shared AWS client utilities (`koalixcrm_utils/aws_clients.py`, `s3_storage.py`)

## Definitions

| Term | Description |
|------|-------------|
| Configuration | A value adjustable at deploy-time or runtime by an operator, without requiring a code change. |
| Environment Variable | A key-value pair injected into the process environment by the container runtime or shell. |
| Default Value | The fallback value used when the env var is absent. |
| Secret | A sensitive configuration value (token, password, key) requiring restricted access. |
| Feature Flag | A boolean toggle that enables or disables a capability at runtime. |

## Summary

| Source Type | Config Count | Scope |
|-------------|-------------|-------|
| Django settings env vars | 19 | Django application process |
| Celery / SQS worker env vars | 8 | `koalixcrm_microservices` worker process |
| AWS client / S3 env vars | 6 | Shared `koalixcrm_utils` library (used by Django and worker) |
| Entrypoint shell vars | 3 | Container startup |
| **Total** | **36** | |

---

## Configuration Inventory

### Source: Environment Variables — Django Application Process

**File:** `projectsettings/settings/base_settings.py` and `projectsettings/settings/development_docker_settings.py`

#### Application Identity

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `KOALIXCRM_VERSION` | ENV | `"vX.Y.Z-develop"` | No | Application version string injected by CI at image build time (from Docker `ARG APP_VERSION`). Displayed for diagnostics; not used in business logic. | Global |
| `KOALIXCRM_LANGUAGE_CODE` | ENV | `'en-us'` | No | Django `LANGUAGE_CODE` setting — the default language for the UI and translated model verbose names. Overridden at the process level. | Global |

#### Security

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `DJANGO_SECRET_KEY` | ENV | `'modify_during_deployment'` | **Yes (prod)** | Django's `SECRET_KEY`. Used to sign sessions, CSRF tokens, and password reset links. The default value is insecure and must be replaced in any non-development deployment. | Global |
| `DJANGO_DEBUG` | ENV | `'True'` | No | Enables Django debug mode when `'true'`, `'1'`, or `'yes'`. Must be `'false'` in production. In debug mode Django returns full stack traces to the browser. | Global |
| `ALLOWED_HOSTS` | Hardcoded `['*']` in dev settings | `['*']` | No | Django's `ALLOWED_HOSTS`. In the development settings file this is hardcoded to `['*']`; a production settings file should restrict this. | Global |

#### Database

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `DB_CHOICE` | ENV | `'postgresql'` | No | Selects the database backend. `'postgresql'` uses PostgreSQL (default); `'sqlite3'` uses SQLite for lightweight local development. | Global |
| `POSTGRES_DB` | ENV | `'koalixcrm'` | No | PostgreSQL database name. | Global (PostgreSQL path only) |
| `POSTGRES_USER` | ENV | `'koalixcrm'` | No | PostgreSQL user. | Global (PostgreSQL path only) |
| `POSTGRES_PASSWORD` | ENV | `'koalixcrm'` | **Yes (prod)** | PostgreSQL password. The default is insecure; must be replaced in production. | Global (PostgreSQL path only) |
| `POSTGRES_HOST` | ENV | `'postgres'` | No | PostgreSQL host. Defaults to `'postgres'`, matching the docker-compose service name. | Global (PostgreSQL path only) |
| `POSTGRES_PORT` | ENV | `'5432'` | No | PostgreSQL port. | Global (PostgreSQL path only) |
| `SQLITE_DB_FILE` | ENV | `<BASE_DIR>/db.sqlite3` | No | Path to the SQLite database file. Used only when `DB_CHOICE=sqlite3`. | Global (SQLite path only) |

#### Storage (S3)

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `S3_ENDPOINT_URL` | ENV | `None` | No | Custom S3-compatible endpoint URL. When set, boto3 clients (S3 and the `TemplateFileStorage` backend) point to this URL instead of AWS. Used for MinIO in local development. When absent, standard AWS S3 is used. | Django app + worker |
| `S3_PDF_BUCKET` | ENV | `'koalixcrm-pdf-exports'` | No | S3 bucket name for generated PDF exports and uploaded document template files. | Django app + worker |

#### OIDC — User-facing API Authentication

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `OIDC_ISSUER` | ENV | `None` | No | Issuer URL of the OIDC provider used to validate user Bearer JWTs on the REST API (`OIDCAccessTokenAuthentication`). When absent, OIDC Bearer token authentication is disabled. | Django REST API |
| `OIDC_ACCEPTED_AUDIENCES` | ENV | `''` (empty list) | No | Comma-separated list of accepted `aud`/`client_id`/`azp` values for user access tokens. When empty, audience validation is skipped. | Django REST API |

#### OIDC — Admin Browser Login

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `ADMIN_OIDC_ISSUER` | ENV | `None` | No | Issuer URL of the OIDC provider used for the Django Admin browser login flow. When absent, the admin falls back to the Django username/password form. | Django Admin |
| `ADMIN_OIDC_CLIENT_ID` | ENV | `None` | No | OAuth 2.0 client ID registered with the IdP for the admin login client. | Django Admin |
| `ADMIN_OIDC_CLIENT_SECRET` | ENV | `None` | **Yes (if ADMIN_OIDC_ISSUER set)** | OAuth 2.0 client secret for the admin login client. Required when `token_endpoint_auth_method` is `client_secret_post`. | Django Admin |
| `SITE_URL` | ENV | `''` | No | Base URL of the Django application (e.g., `https://crm.example.com`). Used to construct OAuth callback redirect URIs. When absent, `request.build_absolute_uri()` is used as fallback. | Django Admin / Auth |

#### OIDC — M2M (Machine-to-Machine) Authentication

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `CELERY_WORKER_M2M_OIDC_ISSUER` | ENV | `None` | No | Issuer URL of the OIDC provider trusted for M2M Client Credentials JWTs (Celery worker and PDF export service). When absent, M2M authentication is disabled. | Django REST API |
| `CELERY_WORKER_M2M_CLIENT_ID` | ENV | `None` | No | OAuth 2.0 client ID of the Celery worker / PDF export service M2M client. Used to verify that incoming M2M tokens were issued for this specific client. | Django REST API |
| `CELERY_WORKER_M2M_CLIENT_SECRET` | ENV | `None` | No | OAuth 2.0 client secret for the M2M client. Not used in token validation (the secret stays with the worker); listed for completeness as it is read from the environment by the settings file. | Django settings |
| `CELERY_WORKER_M2M_SCOPE` | ENV | `None` | No | OAuth 2.0 scope requested by the M2M client when obtaining its token. Read by the settings file; the actual token request is made by the worker, not Django. | Django settings |

#### Django Settings Module

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `DJANGO_SETTINGS_MODULE` | ENV | `projectsettings.settings.development_docker_settings` (dev), `projectsettings.settings.production_docker_postgres_settings` (prod) | No | Selects the active Django settings profile. Set by the container entrypoint when not already present in the environment. | Container startup |

---

### Source: Environment Variables — Celery / SQS Worker (`koalixcrm_microservices/`)

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `CELERY_BROKER_URL` | ENV | `None` | **Yes** | Celery broker connection URL. For SQS: `sqs://aws_access_key_id:aws_secret_access_key@`. Required for the worker to start. | Celery worker |
| `CELERY_RESULT_BACKEND` | ENV | `None` | No | Celery result backend URL. Optional; results are not currently stored for production SQS-backed tasks. | Celery worker |
| `CELERY_SQS` | ENV | `None` | **Yes** | SQS queue name used as the default task queue (`task_default_queue`). Also used to construct the predefined queue URL for the broker transport. | Celery worker |
| `KOALIXCRM_MICROSERVICE_SQS` | ENV | `'koalixcrm-microservice-sqs'` | No | SQS queue name polled by the microservice SQS poller thread running inside the Celery worker. Also used as a default by `koalixcrm_utils.aws_clients.get_sqs_queue()`. | Celery worker + Django |
| `ENABLE_SQS_POLLER` | ENV | `'true'` | No | Feature flag that controls whether the SQS poller background thread starts when the Celery worker is ready. Set to `'false'` to disable polling (e.g., for Celery Beat-only instances). | Celery worker |
| `POLL_SLEEP_SECONDS` | ENV | `'2'` | No | Sleep duration (seconds) between SQS receive-message iterations in the microservice poller. | Celery worker |
| `LOG_LEVEL` | ENV | `'INFO'` | No | Python logging level for the Celery worker process. Accepted values: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. | Celery worker |

---

### Source: Environment Variables — AWS Clients (`koalixcrm_utils/`)

These environment variables are consumed by the shared `koalixcrm_utils.aws_clients` module
and are therefore available to both the Django application process and the Celery worker.

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `SQS_ENDPOINT_URL` | ENV | `None` | No | Custom SQS-compatible endpoint URL. When set, SQS boto3 clients point to this URL (ElasticMQ for local dev) instead of AWS. Must match `SQS_ENDPOINT_URL` used in `celery_app.py`. | Django + worker |
| `AWS_REGION` | ENV | `'eu-west-3'` | No | AWS region for S3 and SQS clients. Default is `eu-west-3` (Paris). | Django + worker |
| `AWS_ACCESS_KEY_ID` | ENV | `'minioadmin'` (S3 local), `'dummy'` (SQS local) | **Yes (prod)** | AWS access key ID for boto3 authentication. The defaults are placeholder values for local MinIO/ElasticMQ; real AWS credentials are required in production. | Django + worker |
| `AWS_SECRET_ACCESS_KEY` | ENV | `'minioadmin123'` (S3 local), `'dummy'` (SQS local) | **Yes (prod)** | AWS secret access key. Default values are insecure placeholders for local development. | Django + worker |
| `AWS_PROFILE` | ENV | `None` | No | Named AWS CLI profile for boto3 credential resolution. Used in the Celery worker when `SQS_ENDPOINT_URL` is absent (production path) to call STS and resolve the account ID. | Celery worker |
| `PRESIGNED_URL_EXPIRES_IN` | ENV | `'300'` | No | Lifetime in seconds of presigned S3 GET URLs generated for document template assets. The integer value parsed at module import time. | Django app |

---

### Source: Container Entrypoints

#### Production Entrypoint (`docker/prod/entrypoint.sh`)

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `DJANGO_SETTINGS_MODULE` | ENV | `projectsettings.settings.production_docker_postgres_settings` | No | Overrides the Django settings module for the production container. Set by the entrypoint if not already present. | Container startup |
| `GUNICORN_WORKERS` | ENV | `3` | No | Number of Gunicorn worker processes. The entrypoint uses `${GUNICORN_WORKERS:-3}`; operators set this env var to override. | Container startup |
| `GUNICORN_TIMEOUT` | ENV | `120` | No | Gunicorn worker request timeout in seconds. The entrypoint uses `${GUNICORN_TIMEOUT:-120}`; operators set this env var to override. | Container startup |

#### Development Entrypoint (`docker/dev/entrypoint.sh`)

| Config Key | Source | Default Value | Required | Description | Scope |
|------------|--------|---------------|----------|-------------|-------|
| `DJANGO_SETTINGS_MODULE` | ENV | `projectsettings.settings.development_docker_settings` | No | Overrides the Django settings module for the development container. | Container startup |

---

## Configuration File Catalog

| File | Format | Purpose | Loaded By | Environment-Specific |
|------|--------|---------|-----------|---------------------|
| `projectsettings/settings/base_settings.py` | Python | Base Django settings; hardcoded constants and env var reads shared by all profiles | All Django processes | No (base only) |
| `projectsettings/settings/development_docker_settings.py` | Python | Development profile: PostgreSQL or SQLite, DEBUG=True, OIDC optional | Dev containers | Yes (development) |
| `projectsettings/settings/development_docker_sqlite_settings.py` | Python | Lightweight development profile: SQLite only, DEBUG=True | Dev containers without PostgreSQL | Yes (development) |
| `elasticmq.conf` | Typesafe (HOCON) | Static ElasticMQ broker configuration for local development; defines queues and network binding | ElasticMQ container at startup | Yes (local dev only) |

**Note.** The production settings file `projectsettings.settings.production_docker_postgres_settings`
is referenced in `docker/prod/entrypoint.sh` but is not present in the repository. This is an
identified gap (see Improvement Opportunities below).

---

## Command-Line Interface

The Django management interface exposes the following runtime CLI commands relevant to
configuration and deployment. These are invoked by the entrypoint scripts; they are not
parameterized by CLI flags beyond the standard `manage.py` options.

| Command | Invoked By | Description |
|---------|-----------|-------------|
| `manage.py sync_split_migrations` | Production and dev entrypoints | Reconciles migration history after app-splitting; run once on upgrade. Accepts no custom flags. |
| `manage.py migrate --noinput` | Both entrypoints | Applies pending database migrations non-interactively. |
| `manage.py collectstatic --noinput` | Both entrypoints | Copies static files to `STATIC_ROOT` non-interactively. |
| `manage.py koalixcrm_install_defaulttemplates` | Manual operator step | Seeds default reference data (workspace, currency, templates). Idempotent. |

---

## Cross-Reference: Configuration Influencing Use Cases

| Configuration Key | Use Case(s) Affected | Effect |
|-------------------|---------------------|--------|
| `ADMIN_OIDC_ISSUER` | UC-WA-01 Login via OIDC, UC-WA-02 Logout | When set, the admin login redirects to the IdP; when absent, the Django form fallback is used. Logout redirects to `end_session_endpoint` only when this is set. |
| `ADMIN_OIDC_CLIENT_ID`, `ADMIN_OIDC_CLIENT_SECRET` | UC-WA-01 Login via OIDC | Required for the Authorization Code token exchange. |
| `SITE_URL` | UC-WA-01, UC-WA-02 | Used to construct the OAuth redirect URI and `post_logout_redirect_uri`. When absent, the URI is derived from the request, which may produce incorrect URLs behind a reverse proxy. |
| `OIDC_ISSUER` | UC-WA-08 REST API Authentication | Enables OIDC Bearer JWT validation for API clients. Absent means all OIDC Bearer tokens are rejected. |
| `OIDC_ACCEPTED_AUDIENCES` | UC-WA-08 REST API Authentication | Restricts which OAuth clients' tokens are accepted. An empty list disables audience validation. |
| `CELERY_WORKER_M2M_OIDC_ISSUER`, `CELERY_WORKER_M2M_CLIENT_ID` | UC-WA-08 REST API Authentication | Enables M2M authentication for the Celery worker and PDF export service. |
| `DJANGO_SECRET_KEY` | UC-WA-01, UC-WA-03, all session-based use cases | Signs session cookies; a weak or default key compromises session security. |
| `DJANGO_DEBUG` | All use cases | In debug mode, full stack traces are returned to the browser on errors, revealing internal details. |
| `POSTGRES_*` | All use cases (data persistence) | Determines which database instance the application connects to. Misconfiguration prevents startup. |
| `DB_CHOICE` | All use cases (data persistence) | Switches between PostgreSQL and SQLite backends. |
| `S3_ENDPOINT_URL`, `S3_PDF_BUCKET` | UC-REP PDF export, UC-UE Upload Document Template | Determines whether PDF exports and templates are stored in AWS S3 or a local MinIO instance. |
| `CELERY_BROKER_URL`, `CELERY_SQS` | UC-REP async PDF generation | The worker cannot start without these; PDF export via SQS is unavailable. |
| `ENABLE_SQS_POLLER` | UC-REP async PDF notification | Setting to `false` disables the background SQS polling thread (e.g., for Beat-only worker instances). |
| `KOALIXCRM_MICROSERVICE_SQS` | Microservice command dispatch | Determines which SQS queue the poller consumes for inter-service commands. |
| `GUNICORN_WORKERS` | All HTTP use cases | Too few workers causes request queuing under load; too many exhausts memory. |
| `GUNICORN_TIMEOUT` | UC-REP PDF export (synchronous path), long-running admin actions | Requests that exceed this timeout are killed by Gunicorn. |
| `KOALIXCRM_LANGUAGE_CODE` | All UI use cases | Sets the default UI language for translated admin labels and messages. |

---

## Security Considerations

| Configuration Key | Contains Secret | Storage Method | Rotation Policy | Notes |
|-------------------|----------------|----------------|-----------------|-------|
| `DJANGO_SECRET_KEY` | Yes | ENV | Rotate on compromise or periodically; rotation invalidates all existing sessions | Must differ per deployment environment |
| `POSTGRES_PASSWORD` | Yes | ENV | Rotate via database user management | Default value `'koalixcrm'` must never be used in production |
| `ADMIN_OIDC_CLIENT_SECRET` | Yes | ENV | Rotate via IdP client configuration | Required for Authorization Code token exchange |
| `CELERY_WORKER_M2M_CLIENT_SECRET` | Yes | ENV | Rotate via IdP client configuration | Held by the worker, not Django; the settings file reads it to pass through |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Yes | ENV | Rotate per AWS IAM best practice | Default dev values (`minioadmin`, `minioadmin123`, `dummy`) must not reach production |
| `CELERY_BROKER_URL` | Yes (if credentials embedded) | ENV | Rotate with SQS IAM credentials | The SQS URL format may embed access key and secret |

---

## Improvement Opportunities

| Finding | Current State | Recommendation | Priority |
|---------|--------------|----------------|----------|
| Missing production settings file | `docker/prod/entrypoint.sh` references `projectsettings.settings.production_docker_postgres_settings` which does not exist in the repository | Create the production settings file with hardened defaults (`DEBUG=False`, restricted `ALLOWED_HOSTS`, secure session cookie settings) | Critical |
| Insecure default `DJANGO_SECRET_KEY` | `development_docker_settings.py` falls back to `'modify_during_deployment'` | Enforce a missing-key error in the production settings file; never provide a non-random default | High |
| Insecure default `POSTGRES_PASSWORD` | Defaults to `'koalixcrm'` | Enforce a required env var with no default in the production settings; add a startup assertion | High |
| Insecure default AWS credentials | `AWS_ACCESS_KEY_ID` defaults to `'minioadmin'`/`'dummy'` | Production path should use IAM instance roles rather than static credentials; remove static fallbacks from production code paths | High |
| `ALLOWED_HOSTS = ['*']` in dev settings | Hardcoded open; not suitable for production | Production settings file must restrict to the actual hostname | High |
| No `.env.example` template | No example file documents required env vars for new deployments | Create a `.env.example` listing all env vars, their defaults, and which are required | Medium |
| `PRESIGNED_URL_EXPIRES_IN` | Already readable from env; the constant name and its env var key are not prominently documented | Document in `.env.example`; the implementation is already correct | Low |
