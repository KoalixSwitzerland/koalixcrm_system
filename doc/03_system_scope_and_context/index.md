# System Scope and Context

This chapter describes the business and technical context of koalixcrm: what the system does,
which actors and external systems it interacts with, and where its boundary lies.

## Business Context

koalixcrm is a modular Django monolith providing CRM, commercial-document management,
double-entry accounting, project-time reporting, and async PDF document generation for small
businesses. Browser-based operators and REST API clients interact with the Django application
container. Asynchronous PDF rendering is offloaded to an external Java service via an SQS
queue, keeping synchronous response times short.

The OIDC identity provider (Keycloak-compatible) sits outside the system boundary; it
validates browser sessions and machine-to-machine JWT credentials without being part of the
koalixcrm codebase.

## Technical Context

The technical entry points are:

- **Gunicorn / WSGI** — the primary HTTP server exposing the Django Admin UI and the REST
  API on port 8000.
- **Celery worker** — an async worker container consuming `CommandEnvelope` messages from
  the microservice SQS queue and relaying them to registered task handlers.
- **Management commands** — a set of Django management commands for maintenance tasks such
  as `sync_split_migrations` and `koalixcrm_install_defaulttemplates`.

External systems outside the koalixcrm boundary:

| External System | Direction | Protocol |
|---|---|---|
| OIDC Provider (Keycloak-compatible) | Inbound token validation | HTTPS / JWKS |
| AWS SQS / ElasticMQ (PDF export queue) | Outbound publish, inbound consume | boto3 / SQS API |
| AWS SQS / ElasticMQ (microservice queue) | Outbound publish, inbound consume | kombu[sqs] |
| AWS S3 / MinIO | Read/Write objects | django-storages / boto3 |
| PDF Export Service (external Java / Apache FOP) | Inbound REST callback + S3 write | HTTP REST / S3 |
| User browser | Inbound HTTP/HTTPS | Port 8000 |

## Documents in this Chapter

| Document | Description |
|---|---|
| [QQ_SD_EntryPoints.md](QQ_SD_EntryPoints.md) | All application entry points: the WSGI server, Celery worker, and Django management commands |
| [QQ_SD_Interface_REST_Specifications.md](QQ_SD_Interface_REST_Specifications.md) | Outward-facing REST API specifications, endpoint catalog, and authentication requirements |
| [QQ_SD_Interface_Async_Specifications.md](QQ_SD_Interface_Async_Specifications.md) | Asynchronous interface specifications: SQS message schemas for PDF export and microservice commands |

## Imported Source Documents

| Document | Description |
|---|---|
| [QQ_IMPORT_docs-api-routing.md](QQ_IMPORT_docs-api-routing.md) | Human-authored API routing reference covering per-app URL prefix conventions and DRF router registration (converted from the project's `docs/` tree) |
