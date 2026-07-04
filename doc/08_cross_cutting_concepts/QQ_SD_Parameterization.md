# Parameterization

This document catalogs all developer-controlled values in koalixCRM: constants, hardcoded
defaults, algorithm tuning parameters, and build-time fixtures embedded directly in source
code or settings files. Changing any of these values requires a code edit, a rebuild, or
a redeployment of the affected component.

## Definitions

| Term | Description |
|------|-------------|
| Parameterization | A value set by a developer that controls internal software behavior. Changing it requires a code change or rebuild. |
| Default Value | The value used when no override is provided at runtime. |
| Constant | A named, immutable value defined in source code. |
| Choice Set | A fixed enumeration of allowed values for a model field, serializer, or API. |
| Hardcoded Literal | A value embedded directly in source code without a symbolic name. |

## Summary

| Scope | Parameter Count | Sources |
|-------|----------------|---------|
| Low-Level (source files) | 24 | Constants in `core/const/`, `auth/oidc_utils.py`, `koalixcrm_microservices/`, `koalixcrm_utils/` |
| Mid-Level (apps / modules) | 11 | `apps.py` peer declarations, `base_settings.py` fixed lists |
| High-Level (system / entrypoints) | 6 | Entrypoint shell defaults, Gunicorn defaults, ElasticMQ static config |
| **Total** | **41** | |

---

## Parameterization Inventory

### Scope: LL — `koalixcrm/core/const/country.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `COUNTRIES` tuple | 249 ISO 3166-1 alpha-2/alpha-3/numeric triples | Constant / Choice Set | `koalixcrm/core/const/country.py` | Populates country drop-down fields on postal address and organisation models | Adding or removing entries affects which countries are selectable; changing codes breaks existing data references |

**Source Reference:** `koalixcrm/core/const/country.py`

---

### Scope: LL — `koalixcrm/core/const/party.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `PARTY_ROLE_CUSTOMER` … `PARTY_ROLE_AUTHORITY` | 8 string literals (`'customer'`, `'supplier'`, …) | Constant | `koalixcrm/core/const/party.py` | Named keys for `PARTY_ROLE_CHOICES`; referenced in model fields, serializers, and admin filters | Renaming breaks existing database values stored under the old key |
| `PARTY_ROLE_CHOICES` | 8-entry tuple | Choice Set | `koalixcrm/core/const/party.py` | Drives the party-role drop-down across contacts and contracts | Any addition or removal requires a migration if the field is constrained |
| `IDENTIFICATION_SCHEME_CHOICES` | 7-entry tuple (`internal`, `vat`, `uid`, `gln`, `duns`, `lei`, `iban`) | Choice Set | `koalixcrm/core/const/party.py` | Controls which identifier scheme values are valid for `PartyIdentifier` records | Changing values invalidates existing stored data |
| `ORG_RELATIONSHIP_CHOICES` | 4-entry tuple | Choice Set | `koalixcrm/core/const/party.py` | Organisation-to-organisation relationship types | Adding requires a migration; removing orphans existing records |
| `ASSIGNMENT_PURPOSE_CHOICES` | 6-entry tuple (`primary`, `billing`, `shipping`, `legal`, `visit`, `other`) | Choice Set | `koalixcrm/core/const/party.py` | Address/phone/email assignment purpose across user and contact models | Changing breaks existing stored purpose values |
| `LEGAL_FORM_CHOICES` | 10-entry tuple | Choice Set | `koalixcrm/core/const/party.py` | Legal form drop-down for organisations | Changing code values invalidates existing data |
| `LANGUAGE_CHOICES` | 4-entry tuple (`de`, `fr`, `it`, `en`) | Choice Set | `koalixcrm/core/const/party.py` | Selectable contact/party languages | Removing a code breaks records that stored it |

**Source Reference:** `koalixcrm/core/const/party.py`

---

### Scope: LL — `koalixcrm/core/const/postaladdressprefix.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `POSTALADDRESSPREFIX` | 4-entry tuple (`F` Company, `W` Mrs, `H` Mr, `G` Ms) | Choice Set | `koalixcrm/core/const/postaladdressprefix.py` | Salutation prefix options for postal addresses | Changing single-character codes invalidates existing stored values |

**Source Reference:** `koalixcrm/core/const/postaladdressprefix.py`

---

### Scope: LL — `koalixcrm/core/const/purpose.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `PURPOSESADDRESSINCONTRACT` | 3-entry tuple (`D` Delivery, `B` Billing, `C` Contact) | Choice Set | `koalixcrm/core/const/purpose.py` | Address-purpose codes in contract documents | Changing code letters breaks existing stored values |
| `PURPOSESADDRESSINCUSTOMER` | 4-entry tuple (`H` Private, `O` Business, `P` Mobile Private, `B` Mobile Business) | Choice Set | `koalixcrm/core/const/purpose.py` | Address-purpose codes for customer contact records | Same as above |
| `PURPOSESTEXTPARAGRAPHINDOCUMENTS` | 10-entry tuple (BS, AS, BT, AT, BW, AW, C1–C4) | Choice Set | `koalixcrm/core/const/purpose.py` | Placement positions for text paragraphs inside generated PDF documents | Changing the two-character codes breaks stored paragraph records |
| `PURPOSECALLINCUSTOMER` | 3-entry tuple (`F` First call, `S` Planned, `A` Assistance) | Choice Set | `koalixcrm/core/const/purpose.py` | Purpose classification for customer call records | Changing codes invalidates existing stored values |
| `PURPOSEVISITINCUSTOMER` | 2-entry tuple (`F` First visit, `S` Installation) | Choice Set | `koalixcrm/core/const/purpose.py` | Purpose classification for customer visit records | Same as above |

**Source Reference:** `koalixcrm/core/const/purpose.py`

---

### Scope: LL — `koalixcrm/core/const/status.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `INVOICESTATUS` | 7-entry tuple (P, C, I, F, R, U, D) | Choice Set | `koalixcrm/core/const/status.py` | Valid status codes for `Invoice` records | Changing single-character codes breaks existing stored statuses |
| `QUOTATIONSTATUS` | 6-entry tuple (S, I, Q, F, R, D) | Choice Set | `koalixcrm/core/const/status.py` | Valid status codes for `Quotation` records | Same as above |
| `PURCHASEORDERSTATUS` | 5-entry tuple (O, D, Y, I, P) | Choice Set | `koalixcrm/core/const/status.py` | Valid status codes for `PurchaseOrder` records | Same as above |
| `DESPATCHADVICESTATUS` | 4-entry tuple (C, S, R, R) | Choice Set | `koalixcrm/core/const/status.py` | Valid status codes for `DespatchAdvice` records | Same as above |
| `CREDITNOTESTATUS` | 4-entry tuple (C, S, B, D) | Choice Set | `koalixcrm/core/const/status.py` | Valid status codes for credit note records | Same as above |
| `CALLSTATUS` | 5-entry tuple (P, D, R, F, S) | Choice Set | `koalixcrm/core/const/status.py` | Valid status codes for customer call records | Same as above |

**Source Reference:** `koalixcrm/core/const/status.py`

---

### Scope: LL — `koalixcrm/auth/oidc_utils.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `JWKS_CACHE_TIMEOUT` | `3600` (seconds, 1 hour) | Constant | `koalixcrm/auth/oidc_utils.py:17` | Duration for which the OIDC provider's JWKS key-set response is cached in Django's cache backend | Decreasing increases IdP round-trips; increasing delays propagation of key rotations |
| `OIDC_DISCOVERY_CACHE_TIMEOUT` | `3600` (seconds, 1 hour) | Constant | `koalixcrm/auth/oidc_utils.py:18` | Duration for which the OIDC `/.well-known/openid-configuration` discovery document is cached | Same trade-off as `JWKS_CACHE_TIMEOUT` |
| JWT algorithm | `"RS256"` | Hardcoded Literal | `koalixcrm/auth/oidc_utils.py:110`, `oidc_token_authentication.py:68` | The only accepted JWT signing algorithm for all OIDC token validation paths | Changing requires IdP configuration alignment; supports only asymmetric keys |
| OAuth scope | `'openid profile email'` | Hardcoded Literal | `koalixcrm/auth/oidc_views.py:30` | Scopes requested during the OIDC Authorization Code flow for admin login | Changing affects which claims the IdP includes in the ID token |
| OAuth PKCE method | `'S256'` | Hardcoded Literal | `koalixcrm/auth/oidc_views.py:31` | Code challenge method for PKCE in the admin OIDC login flow | `S256` is the only recommended method; plain is insecure |
| Token endpoint auth method | `'client_secret_post'` | Hardcoded Literal | `koalixcrm/auth/oidc_views.py:32` | How the client secret is sent to the token endpoint | Must match IdP client configuration |
| M2M placeholder email domain | `'@m2m.local'` | Hardcoded Literal | `koalixcrm/auth/m2m_authentication.py:77` | Domain suffix for auto-provisioned M2M service user email addresses | Changing only affects newly provisioned service users |
| SUPPORTED_PROVIDERS | `['oidc']` | Constant | `koalixcrm/auth/oidc_views.py:18` | List of OAuth provider names accepted by `OAuthLoginView` and `OAuthCallbackView` | Adding a second provider requires both code changes and `oauth.register()` calls |

**Source Reference:** `koalixcrm/auth/oidc_utils.py`, `koalixcrm/auth/oidc_views.py`, `koalixcrm/auth/oidc_token_authentication.py`, `koalixcrm/auth/m2m_authentication.py`

---

### Scope: LL — `koalixcrm_microservices/celery_app.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| Celery timezone | `'UTC'` | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:14` | Internal Celery beat/task scheduler timezone | Changing to a non-UTC zone can cause beat schedule drift and timestamp inconsistencies |
| `task_reject_on_worker_lost` | `False` | Constant | `koalixcrm_microservices/celery_app.py:17` | Controls whether tasks are re-queued when a Celery worker process dies unexpectedly | Setting to `True` can cause duplicate execution for non-idempotent tasks |
| `task_acks_late` | `False` | Constant | `koalixcrm_microservices/celery_app.py:18` | Controls whether tasks are acknowledged after execution (True) or before (False) | Setting to `True` increases risk of re-execution on failure |
| Dev/local SQS polling interval | `1` (second) | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:30` | Celery broker transport polling interval when `SQS_ENDPOINT_URL` is set (local dev) | Lower values increase CPU usage; not used in AWS production path |
| Dev/local SQS visibility timeout | `3600` (seconds) | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:31` | SQS message visibility timeout for the dev/local path | Should exceed maximum task execution time |
| Dev/local SQS wait time | `2` (seconds) | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:32` | SQS long-poll wait time for the dev/local path | 0 disables long-poll; max is 20 |
| Prod SQS polling interval | `1` (second) | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:51` | Celery broker transport polling interval for the AWS production path | Lower values increase API call cost |
| Prod SQS visibility timeout | `3600` (seconds) | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:52` | SQS message visibility timeout for the production path | Same trade-off as dev path |
| Prod SQS wait time | `20` (seconds) | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:53` | SQS long-poll wait time for the production path | Maximum allowed by AWS SQS |
| Fallback AWS account ID | `'000000000000'` | Hardcoded Literal | `koalixcrm_microservices/celery_app.py:27` | Placeholder AWS account ID used when `SQS_ENDPOINT_URL` is set (ElasticMQ local) | Only relevant for local dev; has no effect when connecting to real AWS |
| SQS poller `MaxNumberOfMessages` | `5` | Hardcoded Literal | `koalixcrm_microservices/sqs_poller.py:73` | Maximum messages retrieved per SQS receive call in the microservice poller | SQS maximum is 10; increasing reduces API calls but increases per-batch latency |
| SQS poller `WaitTimeSeconds` | `2` | Hardcoded Literal | `koalixcrm_microservices/sqs_poller.py:74` | Long-poll duration per receive call in the microservice poller | 0 disables long-poll; max is 20 |
| SQS poller `VisibilityTimeout` | `60` (seconds) | Hardcoded Literal | `koalixcrm_microservices/sqs_poller.py:75` | Duration a polled message is hidden from other consumers while being processed | Must exceed maximum dispatch time; too low causes duplicate delivery |

**Source Reference:** `koalixcrm_microservices/celery_app.py`, `koalixcrm_microservices/sqs_poller.py`

---

### Scope: LL — `koalixcrm_utils/s3_storage.py` and `koalixcrm_utils/presigned_urls.py`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `TemplateFileStorage.location` | `"templates"` | Hardcoded Literal | `koalixcrm_utils/s3_storage.py:21` | S3 object key prefix for all uploaded document template files | Changing orphans existing files stored under the old prefix |
| `TemplateFileStorage.file_overwrite` | `False` | Constant | `koalixcrm_utils/s3_storage.py:22` | Prevents overwriting existing template files on re-upload | Setting to `True` allows silent replacement |
| `DEFAULT_EXPIRES_IN` | `300` (seconds, 5 minutes) | Default Value | `koalixcrm_utils/presigned_urls.py:12` | Default lifetime of presigned S3 URLs for template asset downloads | Shorter values reduce the exposure window but may cause link expiry in slow networks |

**Source Reference:** `koalixcrm_utils/s3_storage.py`, `koalixcrm_utils/presigned_urls.py`

---

### Scope: ML — `koalixcrm/*/apps.py` (App Peer Dependencies)

The `required_peers` and `optional_peers` class attributes on each `AppConfig` subclass
define the module dependency graph checked at startup via `register_peer_check`. These
are developer-controlled constants embedded in the app registration.

| App | `required_peers` | `optional_peers` | Location |
|-----|-----------------|-----------------|----------|
| `koalixcrm.core` | `()` | `('koalixcrm.accounting',)` | `koalixcrm/core/apps.py:11-12` |
| `koalixcrm.contacts` | `('koalixcrm.core',)` | `()` | `koalixcrm/contacts/apps.py:11-12` |
| `koalixcrm.products` | `('koalixcrm.core',)` | `('koalixcrm.accounting',)` | `koalixcrm/products/apps.py:11-12` |
| `koalixcrm.contracts` | `('koalixcrm.core', 'koalixcrm.contacts')` | `('koalixcrm.products', 'koalixcrm.djangoUserExtension')` | `koalixcrm/contracts/apps.py:9-13` |
| `koalixcrm.djangoUserExtension` | `('koalixcrm.core', 'koalixcrm.contacts')` | `('koalixcrm.reporting',)` | `koalixcrm/djangoUserExtension/apps.py:10-11` |
| `koalixcrm.accounting` | `('koalixcrm.core', 'koalixcrm.djangoUserExtension')` | `('koalixcrm.products',)` | `koalixcrm/accounting/apps.py:8-9` |
| `koalixcrm.reporting` | n/a (no peers declared) | n/a | `koalixcrm/reporting/apps.py` |
| `koalixcrm.subscriptions` | n/a (no peers declared) | n/a | `koalixcrm/subscriptions/apps.py` |

**Notes.** The `default_auto_field = 'django.db.models.BigAutoField'` class attribute is
set on all apps, fixing the primary-key type to 64-bit integer for all models in each
app. This is a parameterization decision that would require a migration if changed.

**Source Reference:** All `koalixcrm/*/apps.py` files.

---

### Scope: ML — `projectsettings/settings/base_settings.py` (Fixed Application Lists)

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `PREREQUISITE_APPS` | 9-entry list (`contenttypes`, `grappelli.dashboard`, `grappelli`, `filebrowser`, `admin`, `auth`, `sessions`, `messages`, `staticfiles`, `rest_framework`, `django_filters`, `drf_spectacular`, `storages`) | Constant | `base_settings.py:16-30` | Fixed set of third-party and Django framework apps required by all settings profiles | Removing any entry breaks dependent functionality; additions require package installation |
| `PROJECT_APPS` | 8-entry list of `koalixcrm.*` apps | Constant | `base_settings.py:32-41` | All koalixCRM application modules loaded in every deployment | Removing disables the corresponding business domain |
| `KOALIXCRM_PLUGINS` | `('koalixcrm.subscriptions',)` | Constant | `base_settings.py:45-47` | Apps treated as optional plugins by the plugin loading system | Adding/removing entries changes which apps are managed as plugins |
| `MIDDLEWARE` | 7-entry ordered list | Constant | `base_settings.py:49-59` | Fixed Django middleware stack including `WorkspaceContextMiddleware` and `TimezoneMiddleware` | Order matters; inserting or removing a middleware can break authentication, session handling, or workspace scoping |
| `AUTH_PASSWORD_VALIDATORS` | 4-entry list of Django built-in validators | Constant | `base_settings.py:90-103` | Password strength rules enforced during user creation and password change | Removing validators weakens password security |
| `AUTHENTICATION_BACKENDS` | 2-entry list (`OIDCAuthenticationBackend`, `ModelBackend`) | Constant | `base_settings.py:164-167` | Django authentication backend chain; determines how credentials are resolved | Order matters: OIDC is tried first; removing `ModelBackend` disables username/password login |

**Source Reference:** `projectsettings/settings/base_settings.py`

---

### Scope: HL — Docker Entrypoints (`docker/prod/entrypoint.sh`)

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| Gunicorn bind address | `0.0.0.0:8000` | Hardcoded Literal | `docker/prod/entrypoint.sh:15` | The network interface and port Gunicorn listens on inside the container | Changing requires corresponding container port mapping adjustments |
| Default `GUNICORN_WORKERS` | `3` | Default Value | `docker/prod/entrypoint.sh:17` | Number of Gunicorn worker processes when `GUNICORN_WORKERS` env var is absent | More workers increases memory use; fewer may limit throughput |
| Default `GUNICORN_TIMEOUT` | `120` (seconds) | Default Value | `docker/prod/entrypoint.sh:18` | Gunicorn worker request timeout when `GUNICORN_TIMEOUT` env var is absent | Too low kills in-flight PDF export requests |

**Source Reference:** `docker/prod/entrypoint.sh`

---

### Scope: HL — `docker/dev/entrypoint.sh`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| Debugpy listen address | `0.0.0.0:5678` | Hardcoded Literal | `docker/dev/entrypoint.sh:21` | The interface and port the debugpy adapter listens on for IDE attachment | Changing requires corresponding IDE debug configuration update |
| Django dev server address | `0.0.0.0:8000` | Hardcoded Literal | `docker/dev/entrypoint.sh:21` | The network interface and port the development server listens on | Requires port mapping change in docker-compose |

**Source Reference:** `docker/dev/entrypoint.sh`

---

### Scope: HL — `elasticmq.conf`

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| ElasticMQ protocol | `http` | Hardcoded Literal | `elasticmq.conf:5` | Transport protocol for the local ElasticMQ SQS emulator | Changing to `https` requires TLS certificate setup |
| ElasticMQ bind host | `"*"` (all interfaces) | Hardcoded Literal | `elasticmq.conf:6` | Network interfaces ElasticMQ listens on | Restricting to `127.0.0.1` would break docker-compose inter-container connectivity |
| ElasticMQ port | `9324` | Hardcoded Literal | `elasticmq.conf:7-11` | Port for the SQS-compatible REST API endpoint | Must align with `SQS_ENDPOINT_URL` env var used by the Django and Celery services |
| `sqs-limits` | `strict` | Hardcoded Literal | `elasticmq.conf:14` | Whether ElasticMQ enforces SQS message attribute limits | `strict` reveals potential limit violations early in development |
| Static queue names | `koalixcrm-celery-sqs`, `koalixcrm-microservice-sqs` | Constant | `elasticmq.conf:19-22` | Pre-created queues in the local emulator | Must match `CELERY_SQS` and `KOALIXCRM_MICROSERVICE_SQS` env var values |

**Source Reference:** `elasticmq.conf`

---

### Scope: HL — `projectsettings/settings/base_settings.py` (Fixed Framework Settings)

| Parameter | Value | Type | Location | Purpose | Impact of Change |
|-----------|-------|------|----------|---------|-----------------|
| `USE_I18N` | `True` | Constant | `base_settings.py:107` | Enables Django's internationalization framework | Disabling removes translation support |
| `USE_L10N` | `True` | Constant | `base_settings.py:108` | Enables locale-aware formatting of dates and numbers | Disabling produces unlocalized output |
| `USE_TZ` | `True` | Constant | `base_settings.py:109` | Enables timezone-aware datetime storage | Disabling removes DST/timezone safety and breaks `TimezoneMiddleware` |
| `STATIC_URL` | `'/static/'` | Constant | `base_settings.py:111` | URL prefix for static file serving | Must match web server static file serving configuration |
| `MEDIA_URL` | `"/media/"` | Constant | `base_settings.py:114` | URL prefix for media file serving | Must match web server media serving configuration |
| `LOGIN_URL` | `"/auth/login/"` | Constant | `base_settings.py:133` | URL to which unauthenticated users are redirected | Changing requires corresponding URL routing update |
| `FILEBROWSER_DIRECTORY` | `'uploads/'` | Constant | `base_settings.py:123` | Subdirectory within `MEDIA_ROOT` used by the Grappelli filebrowser | Changing moves the expected location of uploaded files |
| `FILEBROWSER_EXTENSIONS` | Dict of `XML`, `XSL`, `JPG`, `PNG`, `GIF`, `TTF` | Constant | `base_settings.py:124-131` | File types the Grappelli filebrowser accepts for upload | Adding or removing extensions expands or restricts what can be uploaded |
| FOP executable path | `"/usr/bin/fop-2.9/fop/fop"` | Hardcoded Literal | `development_docker_settings.py:36`, `development_docker_sqlite_settings.py:22` | Filesystem path to the Apache FOP binary used for PDF generation in dev containers | The path is baked into the dev settings file; changing FOP version requires updating this path |

**Source Reference:** `projectsettings/settings/base_settings.py`, `projectsettings/settings/development_docker_settings.py`

---

## Cross-Reference: Parameters Influencing Use Cases

| Parameter | Use Case(s) Affected | Effect |
|-----------|---------------------|--------|
| `JWKS_CACHE_TIMEOUT` (3600 s) | UC-WA-01 Login via OIDC, UC-WA-08 REST API Authentication | Determines how long signing keys are cached; a longer timeout delays response to IdP key rotation |
| `OIDC_DISCOVERY_CACHE_TIMEOUT` (3600 s) | UC-WA-01, UC-WA-02, UC-WA-08 | Controls how often the OIDC discovery document is refreshed; affects `end_session_endpoint` discovery in logout |
| JWT algorithm `RS256` | UC-WA-01, UC-WA-08 | Enforces asymmetric signature validation; tokens signed with other algorithms are rejected |
| `AUTHENTICATION_BACKENDS` order | UC-WA-01, UC-WA-08 | OIDC backend is tried before `ModelBackend`; determines which authentication path resolves first |
| `task_reject_on_worker_lost` / `task_acks_late` | UC-REP (PDF export async path) | Controls retry behavior for background tasks; `False`/`False` means tasks are not automatically retried on failure |
| Prod SQS `wait_time_seconds` (20 s) | UC-REP (PDF export queue consumption) | Long-poll reduces empty-receive API calls; maximum AWS SQS value |
| SQS poller `VisibilityTimeout` (60 s) | All async PDF export use cases | Must exceed dispatch processing time to prevent duplicate delivery |
| `DEFAULT_EXPIRES_IN` (300 s) | UC-REP PDF export (presigned URL download) | Limits the window in which the Java PDF worker can download template assets |
| `MIDDLEWARE` list / `TimezoneMiddleware` | UC-WA-07 Set Display Timezone | `TimezoneMiddleware` must be present and correctly positioned; its absence causes timezone selection to have no effect |
| `MIDDLEWARE` list / `WorkspaceContextMiddleware` | UC-WA-03, all workspace-scoped use cases | Must be positioned after `AuthenticationMiddleware` to resolve the authenticated user before workspace lookup |
| `FILEBROWSER_EXTENSIONS` | UC-UE (Upload Document Template) | Restricts the file types that can be uploaded through the Grappelli filebrowser |
| App `required_peers` / `optional_peers` | All use cases per app | Startup check fails if a required peer is missing from `INSTALLED_APPS`; optional peers enable conditional features |

---

## Improvement Opportunities

| Parameter | Current Scope | Recommended Promotion | Rationale |
|-----------|--------------|----------------------|-----------|
| `JWKS_CACHE_TIMEOUT` | Hardcoded constant (1 hour) | Configuration | IdP key rotation intervals vary by deployment; operators should be able to tune the cache without a code change |
| `OIDC_DISCOVERY_CACHE_TIMEOUT` | Hardcoded constant (1 hour) | Configuration | Same rationale as `JWKS_CACHE_TIMEOUT` |
| `DEFAULT_EXPIRES_IN` (presigned URL TTL) | Default Value read from env with fallback 300 s | Already promotable via `PRESIGNED_URL_EXPIRES_IN` env var — no code change needed | The env var path exists; document it as Configuration (see `QQ_SD_Configuration.md`) |
| FOP executable path (`/usr/bin/fop-2.9/fop/fop`) | Hardcoded in dev settings files | Configuration (`FOP_EXECUTABLE` env var) | The production settings file referenced in `entrypoint.sh` does not exist in the repository; the FOP path should be configurable via an env var to support different FOP versions and installation locations |
| Gunicorn bind port (`8000`) | Hardcoded in entrypoint | Configuration | Allows running the application on a non-default port without rebuilding the image |
| Debugpy listen port (`5678`) | Hardcoded in dev entrypoint | Configuration | Allows parallel dev containers to use different debug ports |
| SQS `VisibilityTimeout` values | Hardcoded literals | Configuration | PDF export task duration may vary; operators should tune this per-environment |
| SQS poller `MaxNumberOfMessages` | Hardcoded literal (`5`) | Configuration | Throughput tuning parameter that should be adjustable without code change |
