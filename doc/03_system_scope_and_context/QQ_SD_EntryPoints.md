# Service Entry Points

This document lists all entry points of the koalixcrm cloud application, including the WSGI HTTP server, the Celery worker, the SQS poller, and all Django management commands.

## Deployment Overview

The application ships as two independently deployable containers, both backed by the same codebase:

| Container | Image label | Startup command (prod) |
|-----------|-------------|------------------------|
| Django / HTTP | `koalixcrm-django` | `gunicorn projectsettings.wsgi:application --bind 0.0.0.0:8000` |
| Celery worker | `koalixcrm-celery` | `celery -A koalixcrm_microservices.celery_app worker -l info -B` |

The SQS poller is not a separate process: it starts automatically as a daemon thread inside the Celery worker container at `worker_ready` signal time (controlled by `ENABLE_SQS_POLLER` env var).

## Entry Points Table

| Entry Point Type | Function / Handler Name | File Path |
|------------------|------------------------|-----------|
| Django CLI bootstrap | `execute_from_command_line` | `manage.py` |
| WSGI Application | `application` (`get_wsgi_application`) | `projectsettings/wsgi.py` |
| HTTP Server (prod) | `gunicorn projectsettings.wsgi:application` | `docker/prod/entrypoint.sh` |
| HTTP Server (dev) | `manage.py runserver 0.0.0.0:8000` (under debugpy) | `docker/dev/entrypoint.sh` |
| Celery Worker + Beat | `app = Celery('koalixcrm', …)` | `koalixcrm_microservices/celery_app.py` |
| SQS Poller (thread) | `start_poller` (daemon thread, started by `_on_worker_ready`) | `koalixcrm_microservices/sqs_poller.py` |
| Management Command | `koalixcrm_install_defaulttemplates` — seeds default XSL templates and workspace | `koalixcrm/core/management/commands/koalixcrm_install_defaulttemplates.py` |
| Management Command | `sync_split_migrations` — reconciles legacy migration history at startup | `koalixcrm/core/management/commands/sync_split_migrations.py` |
| Management Command | `contacts_backfill_dryrun` — prints planned Party backfill row counts (read-only) | `koalixcrm/contacts/management/commands/contacts_backfill_dryrun.py` |
| Management Command | `contacts_backfill_reconcile` — verifies Party data migration is safe before v2.0.0 cutover | `koalixcrm/contacts/management/commands/contacts_backfill_reconcile.py` |

## Notes on the HTTP Entry Point

The WSGI callable (`projectsettings/wsgi.py`) is the authoritative HTTP entry point for both production (Gunicorn) and development (`manage.py runserver`). There is no ASGI application — the project uses synchronous WSGI only.

URL routing is defined in `projectsettings/urls.py` and follows the workspace-scoped REST API shape mandated by CR-002 (see [`QQ_IMPORT_docs-api-routing.md`](QQ_IMPORT_docs-api-routing.md)):

```text
/<koalixcrm_app>/api/v1/<int:workspace_id>/<resource>/
```

The six app-level URL confs included are:

| App prefix | URL conf |
|------------|----------|
| `koalixcrm_accounting` | `koalixcrm/accounting/urls.py` |
| `koalixcrm_contacts` | `koalixcrm/contacts/urls.py` |
| `koalixcrm_products` | `koalixcrm/products/urls.py` |
| `koalixcrm_core` | `koalixcrm/core/urls.py` |
| `koalixcrm_contracts` | `koalixcrm/contracts/urls.py` |
| `koalixcrm_reporting` | `koalixcrm/reporting/api_urls.py` |

Each app exposes its own OpenAPI schema (`/api/schema/v1/`), Swagger UI (`/api/swagger/v1/`), and Redoc UI (`/api/redoc/v1/`) triplet via `drf-spectacular`.

## Notes on the Celery / SQS Entry Point

The Celery application is defined in `koalixcrm_microservices/celery_app.py`. It uses AWS SQS as its broker (configurable to a local ElasticMQ via `SQS_ENDPOINT_URL`). The beat schedule is intentionally empty; PDF export was moved to a separate Java service that polls its own SQS queue.

At worker startup the `worker_ready` signal handler (`_on_worker_ready`) launches the `start_poller` function from `koalixcrm_microservices/sqs_poller.py` as a background daemon thread. The poller reads from the queue named by `KOALIXCRM_MICROSERVICE_SQS`, dispatches `CommandEnvelope` messages to Celery tasks, and deletes handled messages. The `TASK_ROUTES` dispatch table is currently empty (all Python-side tasks were removed when the PDF worker moved to Java).
