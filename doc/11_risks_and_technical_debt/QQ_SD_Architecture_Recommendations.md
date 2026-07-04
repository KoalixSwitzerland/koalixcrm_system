# Architecture Recommendations — koalixcrm

## How to Use This Document

Each recommendation below is a self-contained backlog item ready for import into Jira, Azure
DevOps, GitHub Issues, or a similar tool. Items are grouped by category and ordered by priority
(Critical first within each category). The **Implementation Hints** section provides file-level
guidance so that a developer can begin work immediately. **Acceptance Criteria** define testable
"done" conditions.

---

## Summary

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| Security Gaps | 3 | 4 | 4 | 4 | 15 |
| Access Control and Authorization Gaps | 0 | 1 | 3 | 1 | 5 |
| Testing Gaps | 0 | 3 | 3 | 1 | 7 |
| Technical Debt | 0 | 2 | 3 | 0 | 5 |
| Configuration and Settings Gaps | 0 | 1 | 2 | 0 | 3 |
| Architecture Improvements | 0 | 1 | 2 | 1 | 4 |
| Operational Improvements | 0 | 1 | 1 | 1 | 3 |
| Performance Optimizations | 0 | 0 | 2 | 0 | 2 |
| Documentation Gaps | 0 | 0 | 1 | 2 | 3 |
| **Total** | **3** | **13** | **21** | **9** | **46** |

---

## Recommendations

### Category: Security Gaps

#### REC-001: Remove BasicAuthentication from Production Settings

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Critical |
| **Effort** | XS |
| **Affected Components** | `projectsettings`, `koalixcrm.auth`, all REST API endpoints |

**Description:**
`rest_framework.authentication.BasicAuthentication` is the fourth entry in
`DEFAULT_AUTHENTICATION_CLASSES` in `projectsettings/settings/base_settings.py`. Because
`base_settings.py` is the shared foundation for all environment overlays including production,
Basic Auth is active in production deployments. This permits credential-based brute-force
attacks, password spraying against Django accounts, and inadvertent exposure of admin
credentials over the REST API surface. The finding is confirmed in the security report as
F-01 (High) and repeated in the access control document's improvement table.

**Acceptance Criteria:**

- [ ] `BasicAuthentication` is absent from `DEFAULT_AUTHENTICATION_CLASSES` in
  `base_settings.py` and in any production settings overlay.
- [ ] `BasicAuthentication` is present only in a development-or-test-specific settings file
  (e.g. `development_docker_sqlite_settings.py` or a dedicated `test_settings.py`).
- [ ] All existing REST API tests continue to pass when the test runner uses a settings file
  that still includes `BasicAuthentication`.
- [ ] A deployment-validation smoke test confirms that a request using HTTP Basic credentials
  against a production-configured server receives HTTP 401.

**Implementation Hints:**
Remove the `BasicAuthentication` entry from `DEFAULT_AUTHENTICATION_CLASSES` in
`projectsettings/settings/base_settings.py` (line 141). Add it back in
`development_docker_sqlite_settings.py` so that the SQLite-backed test profile retains it for
tests that rely on the admin user fixture. The production settings module
(`production_docker_postgres_settings`) must not inherit or re-add it.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-01
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Improvement Opportunities table
- [QQ_LL_Doc_Auth.md](../05_building_block_view/koalixcrm/auth/QQ_LL_Doc_Auth.md)

---

#### REC-002: Eliminate Default Fallback for DJANGO_SECRET_KEY

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Critical |
| **Effort** | S |
| **Affected Components** | `projectsettings` |

**Description:**
`development_docker_settings.py` sets
`SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY', 'modify_during_deployment')`. The fallback
string `'modify_during_deployment'` is committed in the public repository. If this settings
overlay is accidentally pointed to by `DJANGO_SETTINGS_MODULE` in production without the env
var being set, Django uses a publicly known secret key, enabling session forgery, CSRF token
bypass, and password-reset link manipulation. Additionally,
`development_docker_sqlite_settings.py` hardcodes a fixed key with no env-var override at
all. This is confirmed in security finding F-02 (High).

**Acceptance Criteria:**

- [ ] `development_docker_settings.py` uses `os.environ['DJANGO_SECRET_KEY']` with no
  fallback default (raises `KeyError` if absent, making the misconfiguration visible at
  startup).
- [ ] `development_docker_sqlite_settings.py` reads `SECRET_KEY` from the environment or
  uses a clearly labeled insecure placeholder (e.g. `'INSECURE-TEST-ONLY-DO-NOT-DEPLOY'`)
  that is documented as test-only in a code comment.
- [ ] The production settings module sets `SECRET_KEY` from a secret manager or environment
  variable with no fallback.
- [ ] The project `README` or deployment runbook documents that `DJANGO_SECRET_KEY` is
  mandatory in all non-SQLite deployments.

**Implementation Hints:**
Edit `projectsettings/settings/development_docker_settings.py` line 8. For the SQLite
settings, add a comment block and rename the fallback string to eliminate ambiguity. The
production module (`production_docker_postgres_settings`) should use
`SECRET_KEY = os.environ['DJANGO_SECRET_KEY']`.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-02
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md)

---

#### REC-003: Require Explicit POSTGRES_PASSWORD — Remove Default Fallback

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Critical |
| **Effort** | XS |
| **Affected Components** | `projectsettings` |

**Description:**
`development_docker_settings.py` uses
`os.environ.get('POSTGRES_PASSWORD', 'koalixcrm')` as the PostgreSQL connection password.
The default value `'koalixcrm'` is committed to the repository. A network-accessible
deployment that omits the env var would expose the database with a publicly known password.
This is security finding F-03 (High).

**Acceptance Criteria:**

- [ ] The `POSTGRES_PASSWORD` lookup in all settings files uses no fallback default
  (`os.environ['POSTGRES_PASSWORD']`).
- [ ] The Docker Compose dev file sets `POSTGRES_PASSWORD` explicitly (e.g. via a
  `.env` file that is gitignored).
- [ ] CI pipeline documentation confirms that the env var is injected from the
  `KOALIXCRM_SECRETS_ENV` GitHub Actions secret for integration tests.

**Implementation Hints:**
Edit `projectsettings/settings/development_docker_settings.py` line 30. Update the
Docker Compose configuration to provide `POSTGRES_PASSWORD` via an `.env` file or
secret injection rather than relying on a hardcoded default.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-03
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md)

---

#### REC-004: Add Null-Guards to Subscription.create_invoice Billing-Cycle Chain

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | S |
| **Affected Components** | `koalixcrm.subscriptions` |

**Description:**
`koalixcrm/subscriptions/models/subscription.py` line 49 performs four chained attribute
accesses (`self.contract.defaultcustomer.defaultCustomerBillingCycle.timeToPaymentDate`)
with no null guards. The `subscription_type` FK is declared `null=True`, and neither
`defaultcustomer` nor `defaultCustomerBillingCycle` are guaranteed non-null at the model
level. An `AttributeError` raised here surfaces as an unhandled server error. If
`DEBUG=True`, Django returns a full stack trace to the caller; if `DEBUG=False` it
produces an opaque HTTP 500 with no useful user message. This is security finding F-05
(Medium) and is also documented in the subscriptions LL documentation.

A secondary issue in the same file: `create_quotation` accesses `contract.defaultcustomer`
while `create_invoice` uses `contract.default_customer`, indicating inconsistent field-name
usage that may reflect a stale attribute reference.

**Acceptance Criteria:**

- [ ] `create_invoice` checks that `contract.defaultcustomer` (or the correct field name) is
  not `None` before accessing it, and raises a descriptive application-level exception (e.g.
  `ValueError`) when it is.
- [ ] `create_invoice` checks that `defaultCustomerBillingCycle` is not `None` before
  accessing `timeToPaymentDate`.
- [ ] The field name used in `create_invoice` matches the field name used in `create_quotation`
  (or both are documented as intentionally different).
- [ ] A unit test covers the null-contract-customer path and asserts a `ValueError` (not an
  `AttributeError` or HTTP 500).

**Implementation Hints:**
Edit `koalixcrm/subscriptions/models/subscription.py`. The `create_invoice` method should
guard the chain as:

```python
customer = self.contract.defaultcustomer
if customer is None:
    raise ValueError("Contract has no default customer; cannot create invoice.")
billing_cycle = customer.defaultCustomerBillingCycle
if billing_cycle is None:
    raise ValueError("Customer has no billing cycle; cannot compute payable_until.")
payable_until = date.today() + timedelta(days=billing_cycle.timeToPaymentDate)
```

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-05
- [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md)

---

#### REC-005: Move MinIO Default Credentials Out of Source Code

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | S |
| **Affected Components** | `koalixcrm_utils` |

**Description:**
`koalixcrm_utils/aws_clients.py` hardcodes MinIO development credentials:
`AWS_ACCESS_KEY_ID` defaults to `'minioadmin'` and `AWS_SECRET_ACCESS_KEY` defaults to
`'minioadmin123'` when `S3_ENDPOINT_URL` is set. These strings are in the public repository
and will appear in any audit. Although they are development-only defaults, their presence in
source code is a security anti-pattern (F-04 Medium).

**Acceptance Criteria:**

- [ ] `aws_clients.py` reads `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from the
  environment with no hardcoded fallback when `S3_ENDPOINT_URL` is set.
- [ ] A `.env.example` file (gitignored) documents the required env vars for local MinIO.
- [ ] The Docker Compose dev configuration supplies these values via the `.env` file.
- [ ] The integration test CI job injects them from the `KOALIXCRM_SECRETS_ENV` secret.

**Implementation Hints:**
Edit `koalixcrm_utils/aws_clients.py` lines 33–34. Replace the hardcoded defaults with
`os.environ.get('AWS_ACCESS_KEY_ID')` and `os.environ.get('AWS_SECRET_ACCESS_KEY')`.
Document both variables in a new `.env.example` file at the repository root.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-04
- [QQ_LL_Doc_Utils.md](../05_building_block_view/koalixcrm_utils/QQ_LL_Doc_Utils.md)

---

#### REC-006: Disable m2m_token.env File Persistence in Production

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | XS |
| **Affected Components** | `koalixcrm.shared` |

**Description:**
`TokenCache` in `koalixcrm/shared/token_cache.py` writes a cached M2M access token to
`m2m_token.env` on the container filesystem when `KOALIXCRM_TOKEN_SAVE_TO_ENV=true`. This
file is readable by any process with container filesystem access, including sidecars and
processes that may gain unauthorized access to the container. A compromised container could
extract the token and impersonate the service account. This is security finding F-06
(Medium).

**Acceptance Criteria:**

- [ ] The production deployment configuration sets `KOALIXCRM_TOKEN_SAVE_TO_ENV=false`
  (or leaves it unset, which should default to `false`).
- [ ] The deployment runbook documents that `KOALIXCRM_TOKEN_SAVE_TO_ENV=true` is
  permitted only in development environments.
- [ ] No `m2m_token.env` file is present after a container restart in production.

**Implementation Hints:**
Verify the default value of `KOALIXCRM_TOKEN_SAVE_TO_ENV` in
`koalixcrm/shared/token_cache.py`. If the default is `true`, change it to `false`. Update
environment variable documentation in `QQ_SD_Configuration.md` and the Celery worker service
documentation.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-06
- [QQ_LL_Doc_Shared.md](../05_building_block_view/koalixcrm/shared/QQ_LL_Doc_Shared.md)

---

#### REC-007: Configure TLS and Security Headers in the Production Settings Module

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | S |
| **Affected Components** | `projectsettings` |

**Description:**
No reviewed settings module sets `SECURE_SSL_REDIRECT`, `SESSION_COOKIE_SECURE`,
`CSRF_COOKIE_SECURE`, or `SECURE_HSTS_SECONDS`. `SecurityMiddleware` is present in
`MIDDLEWARE` but operates in no-op mode without these settings. A request reaching the
Django application directly (bypassing the reverse proxy) is served over plain HTTP.
The production settings module (`production_docker_postgres_settings`) is not present
in the reviewed source tree, so its content cannot be verified. This is security finding
F-08 (Medium).

**Acceptance Criteria:**

- [ ] The production settings module sets `SECURE_SSL_REDIRECT = True`.
- [ ] The production settings module sets `SESSION_COOKIE_SECURE = True` and
  `CSRF_COOKIE_SECURE = True`.
- [ ] `SECURE_HSTS_SECONDS` is set to a value of at least 31536000 (one year).
- [ ] `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')` is configured to
  trust the reverse proxy `X-Forwarded-Proto` header.
- [ ] `DEBUG = False` and `ALLOWED_HOSTS` is a closed list in the production module.

**Implementation Hints:**
Create or update `projectsettings/settings/production_docker_postgres_settings.py`. The
file should import from `base_settings` and override only the values that differ in
production. Commit a sanitized version (with secret values as env-var references) to the
repository so the security posture of the production configuration is reviewable.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-08
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md)

---

#### REC-008: Implement API Rate Limiting

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `projectsettings`, all REST API ViewSets |

**Description:**
No DRF throttle classes are configured in any reviewed settings file
(`DEFAULT_THROTTLE_CLASSES` and `DEFAULT_THROTTLE_RATES` are both absent from
`base_settings.py`). All REST endpoints — including authentication-adjacent endpoints such as
the workspace switch view — have no rate limiting. This exposes the API to brute-force and
denial-of-service abuse. This is security finding F-11 (Medium).

**Acceptance Criteria:**

- [ ] `DEFAULT_THROTTLE_CLASSES` in `base_settings.py` includes at least
  `AnonRateThrottle` and `UserRateThrottle`.
- [ ] `DEFAULT_THROTTLE_RATES` defines rates for anonymous and authenticated tiers (e.g.
  `'anon': '100/day'`, `'user': '1000/day'`).
- [ ] The workspace switch endpoint (`/admin/core/workspace/switch/`) applies a per-user
  throttle scope.
- [ ] The OIDC callback endpoint is excluded from throttling (it is an OIDC redirect, not
  a direct user call).
- [ ] Throttle limits are documented in `QQ_SD_Configuration.md`.

**Implementation Hints:**
Add `REST_FRAMEWORK['DEFAULT_THROTTLE_CLASSES']` and `REST_FRAMEWORK['DEFAULT_THROTTLE_RATES']`
to `projectsettings/settings/base_settings.py`. Apply a custom throttle scope to sensitive
views by overriding `throttle_scope` on individual ViewSets.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-11
- [QQ_SD_Interface_REST_Specifications.md](../03_system_scope_and_context/QQ_SD_Interface_REST_Specifications.md)

---

#### REC-009: Restrict OpenAPI Schema Endpoints to Authenticated Users

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | XS |
| **Affected Components** | All apps exposing `drf-spectacular` schema endpoints |

**Description:**
Each app exposes live OpenAPI schema, Swagger UI, and Redoc UI endpoints (e.g.
`/koalixcrm_contacts/api/schema/v1/`). `SERVE_INCLUDE_SCHEMA = False` prevents the
schema from appearing in its own listing, but this does not restrict access to the schema
endpoint itself. Whether these endpoints require authentication depends on the
`SpectacularAPIView` configuration in each app's `urls.py` — the current settings do not
confirm either direction. In a non-public API, unauthenticated schema access discloses
endpoint names, parameter structures, and authentication requirements to unauthenticated
actors. This is security finding F-17 (Low/Informational).

**Acceptance Criteria:**

- [ ] Each `SpectacularAPIView`, `SpectacularSwaggerView`, and `SpectacularRedocView` in
  every app's `urls.py` has `permission_classes = [IsAuthenticated]` (or equivalent).
- [ ] Unauthenticated GET requests to any schema endpoint return HTTP 401.
- [ ] The access control document is updated to reflect the schema endpoint policy.

**Implementation Hints:**
Locate the `urls.py` files for `core`, `contacts`, `contracts`, `products`, `accounting`,
and `reporting` apps. In each `SpectacularAPIView.as_view()` call, pass
`permission_classes=[IsAuthenticated]`. The `drf-spectacular` documentation describes how
to configure per-view permissions on schema endpoints.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-17
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Access Control on Interfaces table

---

#### REC-010: Implement GDPR Data Retention and Anonymisation Pipeline

| Field | Value |
|-------|-------|
| **Type** | Epic |
| **Priority** | Medium |
| **Effort** | L |
| **Affected Components** | `koalixcrm.contacts`, `projectsettings` |

**Description:**
`PartyContact.gdpr_consent_date` records when GDPR consent was given, but no automated
data-retention policy, anonymisation pipeline, or deletion workflow was identified in the
reviewed source or documentation. The right to erasure (GDPR Art. 17) is not implemented
as a dedicated workflow; deleting a `Party` row triggers ORM cascades but is not a
privacy-driven erasure. Downstream processes that send marketing communication are expected
to check `gdpr_consent_date` manually with no enforcement mechanism. This is security
finding F-10 (Medium) and is reflected in the GDPR section of the security report.

**Acceptance Criteria:**

- [ ] A Django management command (e.g. `anonymise_expired_contacts`) is implemented that
  identifies and anonymises or deletes `Party` records beyond a configurable retention period.
- [ ] The command is schedulable via Celery Beat or a cron job.
- [ ] A data subject erasure endpoint or admin action is implemented that orchestrates
  full PII removal for a given `Party` record.
- [ ] The GDPR personal data inventory in `QQ_SD_Security_Report.md` is updated to
  document the lawful basis for each data category.
- [ ] `PartyContact.gdpr_consent_date` is validated at the application layer before
  any marketing send path proceeds.

**Implementation Hints:**
Create a management command under `koalixcrm/contacts/management/commands/`. The command
should iterate `PartyContact` records where `gdpr_consent_date` is null or older than
the configured retention window, blank PII fields (or delete the record if no
legal-hold basis remains), and log each erasure action. Use
`KOALIXCRM_GDPR_RETENTION_DAYS` as a new configuration env var.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-09, F-10, GDPR sections
- [QQ_LL_Doc_Contacts_Models.md](../05_building_block_view/koalixcrm/contacts/QQ_LL_Doc_Contacts_Models.md)

---

#### REC-011: Enforce Phone Number E.164 Format at the Model Level

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | XS |
| **Affected Components** | `koalixcrm.contacts` |

**Description:**
`PhoneNumber.phone_e164` is declared as a plain `CharField(max_length=32)` with no model-level
validator enforcing E.164 format. Validation is expected at the form and serializer layer
but is not uniformly enforced. A record inserted via the ORM or a bulk import could contain
malformed phone numbers without rejection. This is security finding F-12 (Low).

**Acceptance Criteria:**

- [ ] A Django model validator (e.g. using a `RegexValidator` matching `^\+[1-9]\d{1,14}$`)
  is added to `PhoneNumber.phone_e164`.
- [ ] The validator is applied in `PhoneNumber.clean()` or via the `validators` kwarg on the
  field definition.
- [ ] A migration is not required because validators are not stored in the database schema.
- [ ] Existing tests confirm that invalid phone numbers raise `ValidationError`.

**Implementation Hints:**
Edit the `PhoneNumber` model in `koalixcrm/contacts/models/`. Add
`validators=[RegexValidator(r'^\+[1-9]\d{1,14}$', 'Phone must be in E.164 format')]`
to the `phone_e164` field definition.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-12
- [QQ_LL_Doc_Contacts_Models.md](../05_building_block_view/koalixcrm/contacts/QQ_LL_Doc_Contacts_Models.md)

---

#### REC-012: Add Database Unique Constraints for PartyIdentification and PartyRole

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | S |
| **Affected Components** | `koalixcrm.contacts` |

**Description:**
`PartyIdentification` and `PartyRole` lack database-level unique constraints. Duplicate IBAN
records per party (`(party, scheme)`) and duplicate role assignments per party
(`(party, role_type)`) are possible via the ORM, creating data-consistency risks that
affect downstream business logic. This is security finding F-13 (Low).

**Acceptance Criteria:**

- [ ] `PartyIdentification.Meta.unique_together` (or `Meta.constraints`) enforces
  uniqueness on `(party, scheme)`.
- [ ] `PartyRole.Meta.unique_together` (or `Meta.constraints`) enforces uniqueness
  on `(party, role_type)`.
- [ ] A Django migration is generated and reviewed for impact on existing data.
- [ ] Tests assert that duplicate entries raise `IntegrityError` at the database level.

**Implementation Hints:**
Edit the model `Meta` classes in `koalixcrm/contacts/models/`. Prefer
`Meta.constraints = [UniqueConstraint(fields=['party', 'scheme'], name='...')]`
over `unique_together` for Django 4+ style. Generate the migration with
`python manage.py makemigrations contacts`.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-13
- [QQ_LL_Doc_Contacts_Models.md](../05_building_block_view/koalixcrm/contacts/QQ_LL_Doc_Contacts_Models.md)

---

#### REC-013: Delete legacy_data.json After Migration Import

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | XS |
| **Affected Components** | `koalixcrm_utils` |

**Description:**
`pre_migrate_cleanup.py` writes a full database dump to `legacy_data.json`. This file
contains all application data including PII. If not deleted after a successful import, it
persists on the filesystem and is accessible to any process with container filesystem access.
This is security finding F-15 (Low).

**Acceptance Criteria:**

- [ ] `pre_migrate_cleanup.py`'s `import` sub-command deletes `legacy_data.json` after a
  successful import operation.
- [ ] The migration runbook documents the deletion step.
- [ ] If deletion fails (e.g. permission error), a warning is logged and the migration does
  not silently continue.

**Implementation Hints:**
Edit `koalixcrm_utils/pre_migrate_cleanup.py`. After the import sub-command successfully
completes, call `os.remove('legacy_data.json')` wrapped in a `try/except` that logs a
warning on failure.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-15
- [QQ_LL_Doc_Utils.md](../05_building_block_view/koalixcrm_utils/QQ_LL_Doc_Utils.md)

---

#### REC-014: Confirm and Document CORS Policy

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | S |
| **Affected Components** | `projectsettings`, reverse proxy configuration |

**Description:**
No CORS policy (`django-cors-headers` or equivalent) was identified in the reviewed
settings or middleware. The `XFrameOptionsMiddleware` is present (providing
`X-Frame-Options: SAMEORIGIN`), but cross-origin request policies for the REST API are
not documented. It is unknown whether CORS is handled at the reverse proxy layer. Without
a CORS policy, browser-based REST clients from other origins will fail or receive overly
permissive defaults depending on the reverse proxy configuration. This is security finding
F-16 (Low/Informational).

**Acceptance Criteria:**

- [ ] The CORS policy is confirmed as one of: (a) handled at the reverse proxy with
  documented rules, (b) implemented via `django-cors-headers` with an explicit allowed-origin
  list, or (c) not applicable because no browser clients other than the embedded admin UI
  exist.
- [ ] The decision and its rationale are documented in `QQ_SD_Security_Report.md`.

**Implementation Hints:**
If CORS is handled at the reverse proxy, document the nginx/caddy configuration in the
deployment view. If application-layer CORS is needed, install `django-cors-headers`,
add it to `INSTALLED_APPS` and `MIDDLEWARE` in `base_settings.py`, and configure
`CORS_ALLOWED_ORIGINS` in the production settings module.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-16

---

### Category: Access Control and Authorization Gaps

#### REC-015: Wire `permissions_for_role()` into the DRF Permission Pipeline

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | High |
| **Effort** | M |
| **Affected Components** | `koalixcrm.core`, `koalixcrm.shared` |

**Description:**
`permissions_for_role()` in `koalixcrm/core/access.py` maps workspace-level roles to Django
model permission actions, but this mapping is not automatically applied on each request. The
current architecture relies on Django group permissions being pre-assigned in the database.
As a result, effective workspace roles (computed dynamically by `effective_roles(user, obj)`)
are not automatically reflected in the DRF permission check. A user who gains or loses a
workspace role via `RoleInWorkspace` must also have their Django group permissions manually
updated. This is identified as an improvement opportunity in `QQ_SD_AccessControl.md`.

**Acceptance Criteria:**

- [ ] A custom DRF permission class (e.g. `WorkspaceRolePermission`) is implemented in
  `koalixcrm/shared/permissions.py` that calls `effective_roles(request.user, obj)` and
  `permissions_for_role(role)` to resolve whether the current request's HTTP method is
  permitted.
- [ ] `BaseModelViewSet` uses `WorkspaceRolePermission` in addition to or instead of
  `ModelPermissionsWithListView`.
- [ ] `effective_roles` is covered by tests that verify role changes take effect without
  a Django group permission update.
- [ ] Existing REST API tests continue to pass.

**Implementation Hints:**
Implement `WorkspaceRolePermission` in `koalixcrm/shared/permissions.py`. In
`has_permission`, call `effective_roles(request.user, None)` (workspace-level check) and
map the union of roles to a permission set. In `has_object_permission`, call
`effective_roles(request.user, obj)` for object-level checks. Register the class in
`BaseModelViewSet.permission_classes`.

**References:**
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Improvement Opportunities table
- [QQ_LL_Doc_Core_Infrastructure.md](../05_building_block_view/koalixcrm/core/QQ_LL_Doc_Core_Infrastructure.md)
- [QQ_LL_Doc_Shared.md](../05_building_block_view/koalixcrm/shared/QQ_LL_Doc_Shared.md)

---

#### REC-016: Add Audit Log for RoleInWorkspace Changes

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.core` |

**Description:**
`WorkspaceSwitchEvent` provides an audit trail for workspace switches, but there is no audit
log for `RoleInWorkspace` creation, modification, or deletion. Because role changes take
effect immediately on the next request and control access to all workspace data, privilege
escalation events (e.g. granting `ADMIN` to an account) are not traceable. This is
identified as an improvement opportunity in `QQ_SD_AccessControl.md`.

**Acceptance Criteria:**

- [ ] Every `RoleInWorkspace` create, update, and delete operation is recorded in an audit
  trail (Django Admin log, a dedicated audit model, or a signal-based log).
- [ ] Each log entry records: actor (`request.user`), action type, affected workspace,
  affected group, old role (for updates/deletes), new role (for creates/updates), timestamp.
- [ ] The audit log is queryable from the Django Admin.
- [ ] The access control documentation is updated to describe the audit mechanism.

**Implementation Hints:**
Use a `post_save` and `post_delete` signal on `RoleInWorkspace` in
`koalixcrm/core/models/access.py`. Alternatively, override `save_model` and `delete_model`
in the `RoleInWorkspace` admin class to call `LogEntry.objects.log_action`. The Django
Admin already provides `LogEntry` in `django.contrib.admin.models`.

**References:**
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Improvement Opportunities table
- [QQ_LL_Doc_Core_Models.md](../05_building_block_view/koalixcrm/core/QQ_LL_Doc_Core_Models.md)

---

#### REC-017: Scope Accounting Data to Workspaces or Remove Workspace Path Segment

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | Medium |
| **Effort** | M |
| **Affected Components** | `koalixcrm.accounting`, `koalixcrm.core` |

**Description:**
The accounting domain (`Account`, `AccountingPeriod`, `Booking`) is the only business-domain
app whose models are not `WorkspaceScopedModel` instances. These models are global records
shared across all workspaces. The REST API URL carries a `<workspace_id>` path segment for
routing consistency, but that segment does not filter the returned data. Consequently, all
authenticated users can read all accounting records regardless of their workspace membership.
Write access is not restricted beyond authentication. This mismatch is documented in
`QQ_SD_AccessControl.md` (Improvement Opportunities) and in the Accounting REST API section
of the access control interface table.

**Acceptance Criteria:**

- [ ] Either: accounting models (`Account`, `AccountingPeriod`, `Booking`,
  `ProductCategory`) are migrated to inherit `WorkspaceScopedModel` and receive full
  workspace scoping; OR
- [ ] The REST API URL for accounting is explicitly documented as workspace-agnostic
  (e.g. the `<workspace_id>` segment is removed or replaced with a path-neutral prefix),
  and access is gated by `is_staff=True` to match the admin-only intent.
- [ ] A decision record (ADR) documents the chosen direction and its rationale.

**Implementation Hints:**
Option A (full scoping): Add `WorkspaceScopedModel` as a base, generate migrations, and
update all accounting ViewSet querysets to filter by workspace. Option B (explicit global):
Remove `<workspace_id>` from the accounting URL prefix and add `IsAdminUser` to
`AccountViewSet`, `AccountingPeriodViewSet`, `BookingViewSet`, and `ProductCategoryViewSet`
permission classes.

**References:**
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Accounting domain, Improvement Opportunities
- [QQ_LL_Doc_Accounting.md](../05_building_block_view/koalixcrm/accounting/QQ_LL_Doc_Accounting.md)

---

#### REC-018: Implement Object-Level Role Grants (CR-10)

| Field | Value |
|-------|-------|
| **Type** | Epic |
| **Priority** | Medium |
| **Effort** | XL |
| **Affected Components** | `koalixcrm.core`, `koalixcrm.shared` |

**Description:**
The current RBAC model grants roles at the workspace level only (`RoleInWorkspace`). Fine-grained,
per-object role grants (`RoleOnObject`, referred to as CR-10 in the access control document) are
explicitly deferred. The `effective_roles()` function is already structured to support this
extension (the parameter `obj` is accepted but not yet used for object-level lookups). For use cases
that require different access levels per Contract, Project, or Contact, a workspace-level role is too
coarse. This is identified as an improvement opportunity in `QQ_SD_AccessControl.md`.

**Acceptance Criteria:**

- [ ] A `RoleOnObject` join model is defined in `koalixcrm/core/models/access.py` linking a Django
  Group to a specific `ContentType` and object PK at a `Role`.
- [ ] `effective_roles(user, obj)` is extended to merge workspace-level roles and object-level roles.
- [ ] The `permissions_for_role()` pipeline correctly applies the merged role set.
- [ ] Existing workspace-level access control behavior is unchanged (backward compatible).
- [ ] Tests cover the object-level grant path for at least one model type.

**Implementation Hints:**
Add `RoleOnObject(group, content_type, object_id, role)` to
`koalixcrm/core/models/access.py`. Update `effective_roles()` to additionally query
`RoleOnObject` filtered by `content_type` and `object_id` derived from the passed `obj`.
Use Django's `GenericForeignKey` / `ContentType` framework for the polymorphic object
reference.

**References:**
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Improvement Opportunities table (CR-10)
- [QQ_LL_Doc_Core_Infrastructure.md](../05_building_block_view/koalixcrm/core/QQ_LL_Doc_Core_Infrastructure.md)

---

#### REC-019: Reduce JWKS Cache TTL with Fallback Re-Fetch on Key Rotation

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | S |
| **Affected Components** | `koalixcrm.auth` |

**Description:**
JWKS key sets and OIDC discovery documents are cached for 3600 seconds (1 hour) in
`koalixcrm/auth/oidc_utils.py`. When the OIDC provider rotates signing keys, tokens signed
with the new key cause authentication failures for up to one hour. This is documented in
security finding F-14 (Low/Informational) and in the access control document's token
caching table.

**Acceptance Criteria:**

- [ ] When JWT decoding raises a key-not-found error (unknown `kid`), the JWKS cache is
  invalidated for that issuer and the JWKS endpoint is fetched again immediately before
  returning an authentication failure.
- [ ] The retry is performed at most once per request to prevent loops.
- [ ] The base cache TTL remains 3600 seconds for normal (key-found) operation.
- [ ] Documentation in `QQ_LL_Doc_Auth.md` reflects the retry behavior.

**Implementation Hints:**
In `oidc_utils.validate_jwt`, catch the `KeyError` or `InvalidKeyError` raised when the
`kid` is not found in the cached JWKS. On that exception, delete the Django cache key for
that issuer's JWKS and call `get_jwks()` again. Re-attempt `jwt.decode` once with the
fresh JWKS.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-14
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — Token and JWKS Caching table
- [QQ_LL_Doc_Auth.md](../05_building_block_view/koalixcrm/auth/QQ_LL_Doc_Auth.md)

---

### Category: Testing Gaps

#### REC-020: Add Unit Tests for the Auth Package (OIDC and M2M Authentication)

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | High |
| **Effort** | M |
| **Affected Components** | `koalixcrm.auth` |

**Description:**
The OIDC Bearer JWT (`OIDCAccessTokenAuthentication`) and M2M (`CeleryWorkerM2MAuthentication`)
authentication classes have no dedicated unit tests. The auth code paths are exercised indirectly
by API tests that use `BasicAuthentication`, but the OIDC and M2M code paths — JWT signature
validation, JWKS fetching, user auto-provisioning, `at_hash` binding verification, and the
token-confusion pre-check — are entirely untested. This is identified as a key gap in
`QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] Unit tests cover `OIDCAccessTokenAuthentication.authenticate()` with a valid JWT
  (mock JWKS), an expired token, a wrong-issuer token, and a missing `aud` claim.
- [ ] Unit tests cover `CeleryWorkerM2MAuthentication.authenticate()` including the
  pre-check bypass (wrong `iss`/`azp`) and the workspace-fixup path.
- [ ] Tests mock the JWKS endpoint using `unittest.mock` to avoid network calls.
- [ ] At least 80% of lines in `koalixcrm/auth/oidc_token_authentication.py` and
  `koalixcrm/auth/m2m_authentication.py` are covered by the new tests.

**Implementation Hints:**
Create `tests/auth/` mirroring the pattern in `tests/core_api_py/`. Use
`unittest.mock.patch('koalixcrm.auth.oidc_utils.get_jwks', return_value=mock_jwks)` to
inject a test JWKS. Use `PyJWT` to generate test tokens signed with a local RSA key pair
generated for the test suite.

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Key Gaps section
- [QQ_LL_Doc_Auth.md](../05_building_block_view/koalixcrm/auth/QQ_LL_Doc_Auth.md)

---

#### REC-021: Add Unit Tests for the Subscriptions App

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | High |
| **Effort** | M |
| **Affected Components** | `koalixcrm.subscriptions` |

**Description:**
The `koalixcrm.subscriptions` app has no test files at all. The key factory methods
`create_invoice()` and `create_quotation()`, the admin bulk actions, and the null-guard
gap in `create_invoice` (REC-004) are entirely untested. Additionally, the subscriptions
LL documentation identifies a copy-paste defect in the admin: `create_quotation` bulk
action calls `create_invoice` instead of `create_quotation`, and a `create_subscription_pdf`
action is listed in `actions` but not implemented on the class. This is identified as a
key gap in `QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] `create_invoice` is tested with a fully populated contract (happy path) and with a
  null `defaultcustomer` (expects the guard exception from REC-004).
- [ ] `create_quotation` is tested to confirm it calls `create_quotation()` on the
  `Subscription` model (not `create_invoice()`).
- [ ] The copy-paste defect in `create_quotation` admin action is fixed (calls
  `obj.create_quotation()`) and the fix is verified by the test.
- [ ] The `create_subscription_pdf` action is either implemented or removed from the
  `actions` list, and this is verified by a test.
- [ ] A factory for `Subscription` and `SubscriptionType` is added under
  `tests/factories/subscriptions/`.

**Implementation Hints:**
Create `tests/subscriptions/` following the pattern of `tests/contacts/`. Use
`factory_boy` to create `Contract`, `Subscription`, and related objects. The copy-paste
defect is in `koalixcrm/subscriptions/admin/subscription_admin.py` in the
`create_quotation` static method.

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Key Gaps section
- [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md)

---

#### REC-022: Add Tests for REST API Write Validation and Permission Denial Paths

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | High |
| **Effort** | M |
| **Affected Components** | All REST API apps |

**Description:**
The `*_api_py` test directories exercise the CRUD happy path (list, read, create with
valid data, partial update). Validation error responses (missing required fields, wrong
workspace, permission denial for non-superusers, wrong data types) are not covered. The
`WorkspaceScopedViewSetMixin` cross-workspace rejection, the missing `workspace_id` path
case, and the 403 path for insufficient-role users are all untested. This is identified as
a key gap in `QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] At least one test per REST API app asserts HTTP 400 when a required field is missing
  in a POST request.
- [ ] At least one test asserts HTTP 403 when a non-superuser with `VIEWER` role attempts
  a `DELETE` or `POST` operation.
- [ ] A test asserts HTTP 400 or 403 when a create request carries a `workspace` FK
  pointing to a different workspace than the URL `workspace_id`.
- [ ] Tests use the DRF `APIClient` with a non-superuser user created via the factory.

**Implementation Hints:**
Create a shared test fixture for a `viewer_user` (factory-based, assigned `VIEWER` role in
the test workspace) in the root `conftest.py`. Extend `tests/core_api_py/` first as a
model; then apply the pattern to `contracts_api_py`, `contacts_api_py`, etc.

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Key Gaps section
- [QQ_LL_Doc_Shared.md](../05_building_block_view/koalixcrm/shared/QQ_LL_Doc_Shared.md)

---

#### REC-023: Add Direct Tests for the djangoUserExtension App

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.djangoUserExtension` |

**Description:**
The `djangoUserExtension` app has no direct test file. `DocumentTemplate`, `TemplateSet`,
and `UserExtension` are used only indirectly via factories in other tests. The
`TemplateSetJSONSerializer` (which serializes a `TemplateSet` and its ten MTI template
subtypes) and the missing-extension redirect views are untested. These serializers and views
are critical to the PDF export flow. This is identified as a key gap in
`QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] `TemplateSetJSONSerializer` is tested with a fully populated `TemplateSet` (all ten
  template subtypes) and with a minimal `TemplateSet` (some subtypes null).
- [ ] `UserExtension.get_user_extension()` is tested for the user-found and user-not-found
  paths.
- [ ] The missing-extension redirect view returns the expected redirect and error message.
- [ ] A factory for `UserExtension` is added or verified in `tests/factories/djangoUserExtension/`.

**Implementation Hints:**
Create `tests/djangoUserExtension/` with a file `test_user_extension.py`. Use the existing
`DocumentTemplate` and `TemplateSet` factories already present in
`tests/factories/djangoUserExtension/`. Test `TemplateSetJSONSerializer` directly by
instantiating it with a factory-built object and calling `.data`.

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Key Gaps section
- [QQ_LL_Doc_DjangoUserExtension.md](../05_building_block_view/koalixcrm/djangoUserExtension/QQ_LL_Doc_DjangoUserExtension.md)

---

#### REC-024: Add Tests for koalixcrm_utils AWS Client Utilities

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm_utils` |

**Description:**
`koalixcrm_utils` contains the AWS S3 and SQS client factories, S3 template storage, and
presigned URL generation. These utilities are critical to the PDF export path and the SQS
command bus. They are mocked in the PDF endpoint tests but have no isolated unit tests.
This means regressions in the factory logic or presigned URL expiry configuration would go
undetected until integration tests run. This is identified as a key gap in
`QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] `get_s3_client()` and `get_sqs_client()` are tested with `S3_ENDPOINT_URL` set (local
  MinIO path) and unset (AWS path).
- [ ] The presigned URL generation function is tested to confirm the expiry is set to the
  expected value (5 minutes default).
- [ ] Tests mock `boto3.client` to avoid requiring live AWS or MinIO.

**Implementation Hints:**
Create `tests/koalixcrm_utils/test_aws_clients.py`. Use `unittest.mock.patch('boto3.client')`
to intercept client creation and assert the correct arguments (endpoint URL, credentials).

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Key Gaps section
- [QQ_LL_Doc_Utils.md](../05_building_block_view/koalixcrm_utils/QQ_LL_Doc_Utils.md)

---

#### REC-025: Add Admin Bulk-Action Tests for Contracts, Reporting, and Products

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | M |
| **Affected Components** | `koalixcrm.contracts`, `koalixcrm.reporting`, `koalixcrm.products` |

**Description:**
Accounting admin PDF-export actions are tested (`tests/accounting/`). Equivalent admin
bulk-action tests for `contracts` (create invoice from contract, create quotation from
contract, create PDF), `reporting` (create PDF report), and `products` are not identified
in the test suite. The `CreateNewDocumentView` utility, which handles `TemplateSetMissingInContract`
and `TemplateMissingInTemplateSet` error paths, is also untested. This is identified as a
gap in `QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] A test exercises the `create_invoice` admin action for `Contract` and asserts the
  redirect points to the correct invoice admin URL.
- [ ] A test exercises the `TemplateSetMissingInContract` error path in
  `CreateNewDocumentView` and asserts the user receives an appropriate error message.
- [ ] A test exercises the `create PDF` admin action in reporting and asserts a
  `PDFExportProcess` record is created.

**Implementation Hints:**
Use `django.test.TestCase` with the `admin_user` fixture. Call the admin action
via the Django test client POST to `/admin/<app>/<model>/`. Follow the pattern in
`tests/accounting/test_admin_pdf_export.py`.

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Key Gaps section
- [QQ_LL_Doc_Contracts_ViewsSerializers.md](../05_building_block_view/koalixcrm/contracts/QQ_LL_Doc_Contracts_ViewsSerializers.md)

---

#### REC-026: Retire the Legacy django.yml CI Workflow

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | XS |
| **Affected Components** | `.github/workflows/` |

**Description:**
`.github/workflows/django.yml` is an older CI definition targeting deprecated branch names
(`master`/`development`) and running coverage differently from the current `test.yml`. It is
superseded by `test.yml` but remains in the repository, creating confusion about the
authoritative CI pipeline. This is documented in `QQ_SD_Unit_Test_Coverage.md`.

**Acceptance Criteria:**

- [ ] `.github/workflows/django.yml` is deleted from the repository.
- [ ] CI documentation in `QQ_SD_Unit_Test_Coverage.md` reflects only the current `test.yml`
  pipeline.

**Implementation Hints:**
Delete the file and open a pull request. Verify that no branch protection rule or external
integration references `django.yml` by name.

**References:**
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — CI/CD Integration section

---

### Category: Technical Debt

#### REC-027: Fix Copy-Paste Defect in Subscriptions create_quotation Admin Action

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | XS |
| **Affected Components** | `koalixcrm.subscriptions` |

**Description:**
The `create_quotation` static method in
`koalixcrm/subscriptions/admin/subscription_admin.py` calls `obj.create_invoice()` instead
of `obj.create_quotation()`. This is a copy-paste error documented in the subscriptions
LL documentation. When an admin user selects subscriptions and triggers the "Create
Quotation" action, an invoice is created instead, which will generate incorrect
document types and potentially affect financial records.

Additionally, the `actions` list on `OptionSubscription` includes `create_subscription_pdf`
but this method is not implemented on the class; calling it raises an `AttributeError`.

**Acceptance Criteria:**

- [ ] `create_quotation` calls `obj.create_quotation()` on each subscription instance.
- [ ] `create_subscription_pdf` is either implemented with the correct PDF export logic
  (triggering `PDFExportProcess`) or removed from the `actions` list.
- [ ] A unit test (from REC-021) verifies the corrected behavior.

**Implementation Hints:**
Edit `koalixcrm/subscriptions/admin/subscription_admin.py`. In the `create_quotation`
static method, change `obj.create_invoice()` to `obj.create_quotation()`. Decide on
`create_subscription_pdf`: implement it by creating a `PDFExportProcess` record, or remove
the entry from `actions = [...]`.

**References:**
- [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md)

---

#### REC-028: Fix Incorrect Redirect URL in create_subscription Admin Action

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | XS |
| **Affected Components** | `koalixcrm.subscriptions` |

**Description:**
The module-level `create_subscription` function in
`koalixcrm/subscriptions/admin/subscription_admin.py` redirects to
`/admin/subscriptions/{subscription.id}`, which is missing the model name segment in the URL
path. The correct Django admin change URL format is
`/admin/{app_label}/{model_name}/{id}/change/`. The current URL will result in a 404 response
after a subscription is created, leaving the admin user unable to navigate to the new
subscription record. This is documented in the subscriptions LL documentation.

**Acceptance Criteria:**

- [ ] The redirect in `create_subscription` uses Django's `reverse('admin:subscriptions_subscription_change', args=[subscription.id])` to generate the correct admin URL.
- [ ] After creating a subscription from a contract admin page, the user is redirected to
  the new subscription's change page with an HTTP 302.
- [ ] A test verifies the redirect URL format.

**Implementation Hints:**
Edit `koalixcrm/subscriptions/admin/subscription_admin.py`. Replace the manual URL
construction with `django.urls.reverse('admin:subscriptions_subscription_change', args=[subscription.id])`.

**References:**
- [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md)

---

#### REC-029: Eliminate Duplicate DB Queries in Nested Document Serializer

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.contracts` |

**Description:**
`_BaseCommercialDocumentNestedSerializer._positions()` is called twice per nested request —
once from `get_items()` to build the line-item list, and once from `get_tax_summary()` to
compute the tax buckets. Each call issues a separate database query for
`CommercialDocumentPosition` rows. The nested endpoint also issues at minimum 9 queries per
request (positions ×2, party MTI check ×2, contact assignment tables ×3, user extension ×1).
No `select_related` or `prefetch_related` is applied at the ViewSet level. This is documented
in the contracts LL documentation as an "information not available" item and is a confirmed
N+1 pattern.

**Acceptance Criteria:**

- [ ] `_positions()` is called once per serializer instance and the result is cached on the
  instance (e.g. `self._cached_positions`).
- [ ] `get_items()` and `get_tax_summary()` both use the cached result.
- [ ] The nested ViewSet applies `select_related` on party, staff, currency, and template-set
  fields to reduce joins.
- [ ] A query-count assertion test verifies the number of DB queries for a nested request
  with N line items does not grow as O(N).

**Implementation Hints:**
In `_BaseCommercialDocumentNestedSerializer`, add a `@cached_property` or manual
`_positions_cache` pattern in `_positions()`. In the document ViewSets that use
`NestedDetailMixin`, override `get_queryset()` to chain
`.select_related('buyer_party', 'staff', 'default_currency', 'default_template_set')`.

**References:**
- [QQ_LL_Doc_Contracts_ViewsSerializers.md](../05_building_block_view/koalixcrm/contracts/QQ_LL_Doc_Contracts_ViewsSerializers.md) — Access to External Interfaces table

---

#### REC-030: Resolve Inconsistent Field Name Usage in Subscriptions Model

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | XS |
| **Affected Components** | `koalixcrm.subscriptions` |

**Description:**
`Subscription.create_invoice()` accesses `contract.default_customer` while
`Subscription.create_quotation()` accesses `contract.defaultcustomer`. This inconsistency
suggests that one of these references uses the correct current field name on the `Contract`
model and the other is stale. Depending on which one is correct, one of the two methods
may fail silently with an `AttributeError` on `create_quotation()` or raise an exception
on `create_invoice()`. This is documented in the subscriptions LL documentation.

**Acceptance Criteria:**

- [ ] Both methods use the same, correct field name for the default customer on `Contract`.
- [ ] The correct field name is confirmed by inspecting `koalixcrm/contracts/models/`.
- [ ] A unit test confirms that both `create_invoice` and `create_quotation` succeed with
  a fully populated contract.

**Implementation Hints:**
Inspect `koalixcrm/contracts/models/contract_object_management.py` (or equivalent) to
confirm the canonical field name. Update the incorrect reference in either
`subscription.py` or the admin file accordingly.

**References:**
- [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md)

---

#### REC-031: Define Task Routes for the Celery Worker or Document Its Placeholder Status

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm_microservices` |

**Description:**
The `koalixcrm-celery` container's `TASK_ROUTES` dispatch table in `sqs_poller.py` is
intentionally empty. All `CommandEnvelope` messages received are logged and deleted with no
Celery task dispatch. The Celery beat schedule is also empty. The container therefore runs
as infrastructure-only with no operational purpose at this time. This is documented in the
Celery service documentation. Resources are consumed (SQS polling, container runtime,
STS identity resolution at startup) without producing any observable side effect.

**Acceptance Criteria:**

- [ ] Either: at least one command type is registered in `TASK_ROUTES` and the corresponding
  Celery task is implemented and tested; OR
- [ ] A clear decision record documents the current placeholder status, the expected timeline
  for first use, and whether the container should be disabled (`ENABLE_SQS_POLLER=false`)
  until a task is needed.
- [ ] If the container is to remain as a placeholder, the `ENABLE_SQS_POLLER` default is
  changed to `false` to avoid unnecessary SQS polling costs.

**Implementation Hints:**
To register a task route, add an entry to `TASK_ROUTES` in
`koalixcrm_microservices/sqs_poller.py` of the form
`{'command_type_name': ['celery_app.task_name']}`, and create the corresponding Celery
task function decorated with `@celery_app.task`.

**References:**
- [QQ_SD_ServiceDocumentation_CeleryWorker.md](../05_building_block_view/QQ_SD_ServiceDocumentation_CeleryWorker.md) — Current Operational State
- [QQ_LL_Doc_Microservices.md](../05_building_block_view/koalixcrm_microservices/QQ_LL_Doc_Microservices.md)

---

### Category: Configuration and Settings Gaps

#### REC-032: Add .env.example File Documenting All Required Environment Variables

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | S |
| **Affected Components** | `projectsettings`, `koalixcrm_microservices`, `koalixcrm_utils` |

**Description:**
The configuration document (`QQ_SD_Configuration.md`) catalogues 36 configuration
keys across the Django application, Celery worker, and AWS utilities. No `.env.example`
file was identified in the repository. Developers setting up the project for the first
time must consult the documentation (or read multiple settings files) to determine which
environment variables are required. Missing required variables like `DJANGO_SECRET_KEY`
and `POSTGRES_PASSWORD` have insecure defaults (addressed in REC-002 and REC-003) and
their absence must be made explicit at startup.

**Acceptance Criteria:**

- [ ] A `.env.example` file is present at the repository root listing all 36 configuration
  keys with placeholder values and inline comments describing each.
- [ ] The file is committed to the repository (it contains no actual secrets).
- [ ] A `.env` file is listed in `.gitignore`.
- [ ] The project README references `.env.example` in its setup instructions.

**Implementation Hints:**
Create `.env.example` from the inventory in
`doc/08_cross_cutting_concepts/QQ_SD_Configuration.md`. Mark required variables with a
comment `# REQUIRED` and optional variables with their default. Mark secret variables with
`# SECRET - do not commit actual values`.

**References:**
- [QQ_SD_Configuration.md](../08_cross_cutting_concepts/QQ_SD_Configuration.md)
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md)

---

#### REC-033: Persist Timezone Setting Across Sessions

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.core`, `koalixcrm.djangoUserExtension` |

**Description:**
The `django_timezone` user setting is session-scoped and resets to the server's `TIME_ZONE`
setting on logout or session expiry. Users must re-select their timezone on every login. This
is documented in `QQ_SD_Settings.md`. For a CRM used across time zones, resetting the display
timezone on every login creates friction and may cause date/time interpretation errors.

**Acceptance Criteria:**

- [ ] The timezone preference is stored in the `UserExtension` model (a new `preferred_timezone`
  field) or in a dedicated per-user preference model.
- [ ] On login (OIDC callback or session creation), `TimezoneMiddleware` reads the persisted
  preference and activates it.
- [ ] The session key `django_timezone` is populated from the persisted preference at session
  start.
- [ ] Users can still override the active timezone per-session via the existing set_timezone
  form.
- [ ] `QQ_SD_Settings.md` is updated to reflect the new storage scope.

**Implementation Hints:**
Add `preferred_timezone = models.CharField(max_length=50, blank=True, default='')` to
`UserExtension` in `koalixcrm/djangoUserExtension/models/user_extension.py`. Generate the
migration. In `koalixcrm/core/middleware/timezoneMiddleware.py`, after checking
`request.session['django_timezone']`, fall back to `user.userextension.preferred_timezone`.
Update the set_timezone view to save the value to `UserExtension` when a user is
authenticated.

**References:**
- [QQ_SD_Settings.md](../08_cross_cutting_concepts/QQ_SD_Settings.md) — Timezone Preference entry
- [QQ_LL_Doc_DjangoUserExtension.md](../05_building_block_view/koalixcrm/djangoUserExtension/QQ_LL_Doc_DjangoUserExtension.md)

---

#### REC-034: Add Startup Validation for Required Configuration Variables

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `projectsettings` |

**Description:**
Required configuration variables (e.g. `DJANGO_SECRET_KEY`, `POSTGRES_PASSWORD`,
`OIDC_ISSUER`) currently have fallback defaults or no validation at startup. A
misconfigured deployment that omits a required variable starts successfully but behaves
incorrectly or insecurely. Django's system-check framework (`AppConfig.ready()`) can
be used to emit `CRITICAL` errors for missing required configuration before the first
request is served.

**Acceptance Criteria:**

- [ ] A Django system check (registered in `CoreConfig.ready()` or a dedicated
  `checks.py`) emits a `checks.Critical` error if any of the following are unset or
  empty: `DJANGO_SECRET_KEY`, `OIDC_ISSUER`, `ADMIN_OIDC_ISSUER`.
- [ ] The check emits a `checks.Warning` if `DEBUG=True` in a non-SQLite settings context
  (i.e. `DB_CHOICE` is `postgresql`).
- [ ] The check runs as part of `python manage.py check` and in CI.

**Implementation Hints:**
Register a system check in `koalixcrm/core/app_checks.py` following the pattern of the
existing peer-dependency checks (`register_peer_check`). Use
`@register(Tags.configuration)` decorator on a check function that reads `settings.OIDC_ISSUER`
and emits `checks.Critical` if absent.

**References:**
- [QQ_SD_Configuration.md](../08_cross_cutting_concepts/QQ_SD_Configuration.md)
- [QQ_LL_Doc_Core_Infrastructure.md](../05_building_block_view/koalixcrm/core/QQ_LL_Doc_Core_Infrastructure.md)

---

### Category: Architecture Improvements

#### REC-035: Implement REST API for the Subscriptions App

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | High |
| **Effort** | M |
| **Affected Components** | `koalixcrm.subscriptions` |

**Description:**
The `subscriptions` app has no REST API. `subscriptions_api.py` is a stub with no endpoints,
`views.py` is empty, and the `serializers/` package contains no serializers. All subscription
management is Admin-only. This limits programmatic access to subscription data and prevents
downstream integrations (e.g. the WFS fork or external billing systems) from managing
subscriptions via the standard REST surface. This is documented in the subscriptions LL
documentation and in the HL overview table.

**Acceptance Criteria:**

- [ ] `SubscriptionViewSet` and `SubscriptionTypeViewSet` are implemented, inheriting from
  `BaseModelViewSet`.
- [ ] Serializers for `Subscription` and `SubscriptionType` follow the flat `*JSONSerializer`
  pattern used in other apps.
- [ ] The endpoints are mounted under
  `/koalixcrm_subscriptions/api/v1/<workspace_id>/subscriptions/` and
  `/koalixcrm_subscriptions/api/v1/<workspace_id>/subscription-types/`.
- [ ] An OpenAPI schema endpoint and Swagger UI are registered for the app.
- [ ] REST API tests for `Subscription` and `SubscriptionType` (list, read, write, modify)
  are added under `tests/subscriptions_api_py/`.
- [ ] The REST interface specification (`QQ_SD_Interface_REST_Specifications.md`) is updated.

**Implementation Hints:**
Follow the pattern of `koalixcrm/products/` as a reference: `serializers/`, `views/`,
and registration in `urls.py` and `api.py`. Mount the app URL under `subscriptions_api.py`.
Because `Subscription` is a `WorkspaceScopedModel` (via its `Contract` FK), the standard
`WorkspaceScopedViewSetMixin` should apply.

**References:**
- [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md)
- [QQ_HL_Doc_KoalixCRM.md](../05_building_block_view/QQ_HL_Doc_KoalixCRM.md) — API Specifications table
- [QQ_SD_Interface_REST_Specifications.md](../03_system_scope_and_context/QQ_SD_Interface_REST_Specifications.md)

---

#### REC-036: Evaluate and Formalize Optional-App Dependency Validation

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.core`, all optional apps |

**Description:**
The fork-isolation invariant (CR-5) is enforced by `test_fork_isolation.py` using AST-level
and regex scanning of module-level imports. Lazy imports inside function bodies are
intentionally excluded from the check, meaning an optional-peer dependency accessed lazily
is not detected by the test. The `apps.is_installed()` runtime branches that gate optional
features are not tested for correctness (i.e. there is no test that verifies correct
graceful degradation when `accounting`, `reporting`, or `subscriptions` is absent from
`INSTALLED_APPS`). This is a structural robustness gap in the modular boundary enforcement.

**Acceptance Criteria:**

- [ ] A test configuration (separate pytest profile or test class) defines `INSTALLED_APPS`
  with only the five public-surface apps and verifies that Django starts without error.
- [ ] The test profile runs at least one request against each of the five public-surface apps
  and asserts HTTP 200 or 404 (not 500).
- [ ] The `test_fork_isolation.py` documentation is updated to state that lazy imports are
  excluded from the check.

**Implementation Hints:**
Create a `conftest_minimal_apps.py` or a `pytest-minimal` profile in `pytest.ini`. Use
`DJANGO_SETTINGS_MODULE` pointing to a settings file that installs only the five core apps.
Run a smoke-test request using the `django.test.Client` against the contacts REST API.

**References:**
- [QQ_HL_Doc_KoalixCRM.md](../05_building_block_view/QQ_HL_Doc_KoalixCRM.md) — Optional-App Fork Isolation section
- [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) — Fork isolation invariant section
- [QQ_IMPORT_docs-architecture-optional-apps.md](../05_building_block_view/QQ_IMPORT_docs-architecture-optional-apps.md)

---

#### REC-037: Decouple M2M Workspace Fixup from Authentication Class

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.auth`, `koalixcrm.core` |

**Description:**
`CeleryWorkerM2MAuthentication` performs workspace fixup by assigning
`user_workspaces(user).order_by('pk').first()` to `request.active_workspace` after
authenticating the service user. This conflates authentication (verifying identity) with
workspace resolution (a middleware concern). If `WorkspaceContextMiddleware` is refactored
or M2M callers need to specify a workspace, the workspace-fixup logic must be changed in
the authentication class. This is documented in `QQ_SD_AccessControl.md` (M2M
Authentication section).

**Acceptance Criteria:**

- [ ] Workspace resolution for M2M requests is moved to `WorkspaceContextMiddleware` or a
  dedicated post-authentication hook, not performed inside the authentication class.
- [ ] The authentication class sets only `request.user` (not `request.active_workspace`).
- [ ] M2M requests continue to have a valid `active_workspace` set before the ViewSet
  queryset is evaluated.
- [ ] Existing M2M integration tests continue to pass.

**Implementation Hints:**
In `WorkspaceContextMiddleware`, after resolving `request.user`, check whether a workspace
was set by the authentication class and if not, apply the fallback
`user_workspaces(user).order_by('pk').first()`. This keeps the middleware as the single
point of workspace activation.

**References:**
- [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) — M2M Authentication section
- [QQ_LL_Doc_Auth.md](../05_building_block_view/koalixcrm/auth/QQ_LL_Doc_Auth.md)

---

#### REC-038: Add Structured Logging for Business Events

| Field | Value |
|-------|-------|
| **Type** | Story |
| **Priority** | Low |
| **Effort** | M |
| **Affected Components** | All business-domain apps |

**Description:**
No structured logging (e.g. JSON-formatted log entries with event type, workspace ID, user
ID, and affected entity) was identified in the reviewed source. The only documented logging
is the Celery worker's SQS poll activity (line-level INFO to stdout). Business events such
as invoice creation, contract status changes, subscription lifecycle events, and workspace
switches generate no structured audit trail beyond the Django Admin `LogEntry`. Structured
logs would enable operational monitoring, debugging of multi-tenant issues, and compliance
tracing without database access.

**Acceptance Criteria:**

- [ ] A shared logging utility (e.g. `koalixcrm/shared/audit_log.py`) provides a
  `log_business_event(event_type, workspace, user, entity_type, entity_id, payload)` function.
- [ ] Key business events are logged: invoice creation, contract state change, subscription
  invoice creation, workspace switch, `PDFExportProcess` state transitions.
- [ ] Log output is JSON-formatted (compatible with log aggregation systems such as
  Datadog, Elasticsearch, or CloudWatch).
- [ ] The log format is documented in `QQ_SD_Configuration.md` (LOG_LEVEL variable entry).

**Implementation Hints:**
Use Python's `logging` module with `pythonjsonlogger.jsonlogger.JsonFormatter`. Configure
the handler in `base_settings.py` under `LOGGING`. Emit log events from signal handlers
and admin actions using `logging.getLogger('koalixcrm.audit')`.

**References:**
- [QQ_HL_Doc_KoalixCRM.md](../05_building_block_view/QQ_HL_Doc_KoalixCRM.md) — Security section
- [QQ_SD_Configuration.md](../08_cross_cutting_concepts/QQ_SD_Configuration.md) — LOG_LEVEL entry

---

### Category: Operational Improvements

#### REC-039: Add Health Check Endpoint to the Django Container

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | High |
| **Effort** | S |
| **Affected Components** | `koalixcrm-django`, `projectsettings` |

**Description:**
No HTTP health check endpoint was identified in the reviewed source or configuration. The
integration test smoke check polls `/admin/login/` with retries, but this is a CI-only
workaround rather than a production health endpoint. Container orchestration platforms (ECS,
Kubernetes, Docker Compose) rely on health checks to determine container readiness and to
route traffic. Without a dedicated health endpoint, failing database connections or OIDC
provider failures may go undetected until a user request fails.

**Acceptance Criteria:**

- [ ] A `GET /health/` endpoint returns HTTP 200 with a JSON body
  `{"status": "ok", "database": "ok"}` when the database is reachable.
- [ ] The endpoint returns HTTP 503 when the database connection fails.
- [ ] The endpoint is unauthenticated (no auth class applied) to allow load balancer probes.
- [ ] The Docker Compose `healthcheck` directive is updated to use `/health/`.
- [ ] The endpoint URL is documented in `QQ_SD_EntryPoints.md`.

**Implementation Hints:**
Use `django-health-check` or implement a minimal view in `koalixcrm/core/views/` that
calls `django.db.connection.ensure_connection()` inside a `try/except` and returns the
appropriate JSON response. Mount it in the root `urls.py` before authentication middleware.

**References:**
- [QQ_SD_EntryPoints.md](../03_system_scope_and_context/QQ_SD_EntryPoints.md)
- [QQ_SD_ServiceDocumentation_DjangoApp.md](../05_building_block_view/QQ_SD_ServiceDocumentation_DjangoApp.md)

---

#### REC-040: Commit a Sanitized Production Settings Module to the Repository

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `projectsettings` |

**Description:**
`projectsettings/wsgi.py` references
`koalixcrm.projectsettings.settings.production_docker_postgres_settings` as the default
settings module, but this file is absent from the reviewed source tree. Its security-relevant
settings (`ALLOWED_HOSTS`, `DEBUG`, `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`,
`SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS`, `SECRET_KEY`) cannot be verified or reviewed.
This is security finding F-07 (Medium).

**Acceptance Criteria:**

- [ ] A `production_docker_postgres_settings.py` file is present in the repository with all
  secret values replaced by env-var references (no hardcoded secrets).
- [ ] The file imports from `base_settings` and overrides only environment-specific values.
- [ ] The security settings from REC-007 are present in this file.
- [ ] A comment at the top of the file notes that secrets must be injected via environment
  variables and must not be committed.

**Implementation Hints:**
Create `projectsettings/settings/production_docker_postgres_settings.py`. Import
`from .base_settings import *`. Override `DEBUG = False`, `ALLOWED_HOSTS`,
`SECURE_SSL_REDIRECT = True`, `SESSION_COOKIE_SECURE = True`, `CSRF_COOKIE_SECURE = True`,
`SECURE_HSTS_SECONDS = 31536000`, and `SECURE_PROXY_SSL_HEADER`.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-07
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md)

---

#### REC-041: Add Celery Worker Liveness and Readiness Monitoring

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | S |
| **Affected Components** | `koalixcrm-celery`, `koalixcrm_microservices` |

**Description:**
The `koalixcrm-celery` container has no documented health check. The SQS poller runs as a
daemon thread and its liveness is not exposed externally. If the daemon thread crashes
(e.g. due to an unhandled exception in `start_poller`), the Celery worker process continues
running without SQS polling, and no alert is generated. Container orchestration cannot
detect the thread failure.

**Acceptance Criteria:**

- [ ] The SQS poller thread sets a `threading.Event` on each successful poll cycle; the
  main thread or a watchdog checks this event periodically and exits if it is stale.
- [ ] A Docker health check command (e.g. `celery -A koalixcrm_microservices.celery_app
  inspect ping`) is added to the Celery container definition.
- [ ] Thread-level exceptions in `start_poller` are re-raised or logged as CRITICAL to
  trigger alert pipeline pickup.

**Implementation Hints:**
Add a `last_poll_timestamp` module-level variable in `sqs_poller.py` updated on each
successful poll. Add a `threading.Thread` watchdog that checks
`time.time() - last_poll_timestamp > 60` and calls `os._exit(1)` to force a container
restart when the poller is stale. Add the Docker `HEALTHCHECK` instruction to
`docker/prod/Dockerfile.celery`.

**References:**
- [QQ_SD_ServiceDocumentation_CeleryWorker.md](../05_building_block_view/QQ_SD_ServiceDocumentation_CeleryWorker.md)
- [QQ_LL_Doc_Microservices.md](../05_building_block_view/koalixcrm_microservices/QQ_LL_Doc_Microservices.md)

---

### Category: Performance Optimizations

#### REC-042: Apply select_related and prefetch_related to Reporting QuerySets

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.reporting` |

**Description:**
The reporting app models — `Task`, `Project`, `Work`, `ReportingPeriod` — have methods that
aggregate cost and effort by traversing multiple FK relationships (task → project → contract,
work → task → resource → agreement). These methods are exercised by the test suite and are
confirmed to produce correct results, but there is no evidence that the underlying
`QuerySet.filter()` calls or the serializers in `reporting_api_py` use `select_related` or
`prefetch_related`. With typical project sizes (tens of tasks, hundreds of work entries),
this creates N+1 query patterns on the list and nested endpoints.

**Acceptance Criteria:**

- [ ] The `Task` and `Work` ViewSet `get_queryset()` methods apply `select_related` on
  `project`, `responsible`, `task_status`, and `reporting_period`.
- [ ] The `Project` ViewSet applies `prefetch_related('tasks')`.
- [ ] A query-count test (using `django.test.utils.CaptureQueriesContext`) asserts that
  fetching a list of 10 tasks issues fewer than 5 queries total.

**Implementation Hints:**
Edit the `get_queryset()` overrides in `koalixcrm/reporting/views/`. Add
`.select_related('project', 'responsible', 'task_status', 'reporting_period')` to the
`Task` ViewSet. Add `.prefetch_related('tasks')` to the `Project` ViewSet.

**References:**
- [QQ_ML_Doc_Reporting.md](../05_building_block_view/koalixcrm/reporting/QQ_ML_Doc_Reporting.md)
- [QQ_LL_Doc_Reporting_ProjectTaskModels.md](../05_building_block_view/koalixcrm/reporting/QQ_LL_Doc_Reporting_ProjectTaskModels.md)

---

#### REC-043: Cache Queue URL Resolution in the SQS Poller

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | XS |
| **Affected Components** | `koalixcrm_microservices` |

**Description:**
The SQS poller daemon thread calls `sqs.get_queue_url` on every poll iteration (every 2
seconds) to resolve the queue URL for `KOALIXCRM_MICROSERVICE_SQS`. This is documented in
the Celery service documentation. The queue URL for a given queue name is stable between
deployments; resolving it on every iteration is unnecessary and adds latency to each poll
cycle as well as an extra API call billable by AWS.

**Acceptance Criteria:**

- [ ] The queue URL is resolved once at poller startup (before the loop begins) and cached
  in a local variable.
- [ ] If `get_queue_url` fails at startup, the error is logged and the poller retries with
  exponential back-off before entering the poll loop.
- [ ] If the queue is deleted and re-created (URL changes), the poller catches the SQS
  `QueueDoesNotExist` error and re-resolves the URL.

**Implementation Hints:**
Move the `sqs.get_queue_url` call in `start_poller()` from inside the `while True:` loop
to before the loop starts. Store the result in `queue_url`. Inside the loop, catch
`sqs.exceptions.QueueDoesNotExist` and refresh `queue_url` on that exception only.

**References:**
- [QQ_SD_ServiceDocumentation_CeleryWorker.md](../05_building_block_view/QQ_SD_ServiceDocumentation_CeleryWorker.md) — SQS Poller Daemon Thread section
- [QQ_LL_Doc_Microservices.md](../05_building_block_view/koalixcrm_microservices/QQ_LL_Doc_Microservices.md)

---

### Category: Documentation Gaps

#### REC-044: Add SalesOrder to Nested Endpoint Coverage or Document the Gap

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Medium |
| **Effort** | S |
| **Affected Components** | `koalixcrm.contracts` |

**Description:**
`SalesOrderViewSet` does not mix in `NestedDetailMixin` and therefore exposes no
`GET /{id}/nested/` endpoint for the PDF worker. This is documented in the contracts LL
documentation: "SalesOrderViewSet follows the same pattern but does not mix in
`NestedDetailMixin` — no nested endpoint is exposed for sales orders." It is unclear
whether this is intentional (no PDF generation for sales orders) or an omission. The
difference is not explained in the interface specification or the service documentation.

**Acceptance Criteria:**

- [ ] Either: `SalesOrderViewSet` mixes in `NestedDetailMixin` with a `SalesOrderNestedSerializer`,
  and the interface specification is updated to document the endpoint; OR
- [ ] A code comment in `SalesOrderViewSet` explicitly states that PDF generation for sales
  orders is not supported and references the decision that led to this choice.
- [ ] The interface specification (`QQ_SD_Interface_REST_Specifications.md`) reflects the
  correct state.

**Implementation Hints:**
Check whether `CommercialDocument` subclasses other than `SalesOrder` have PDF export
admin actions. If sales orders do have a PDF action, implement the nested serializer
following the pattern of `InvoiceNestedSerializer`. If not, add a comment to the ViewSet.

**References:**
- [QQ_LL_Doc_Contracts_ViewsSerializers.md](../05_building_block_view/koalixcrm/contracts/QQ_LL_Doc_Contracts_ViewsSerializers.md)
- [QQ_SD_Interface_REST_Specifications.md](../03_system_scope_and_context/QQ_SD_Interface_REST_Specifications.md)

---

#### REC-045: Document the SQS Message Authentication Boundary and IAM Policy Requirements

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | S |
| **Affected Components** | `koalixcrm.core`, `koalixcrm_utils`, deployment configuration |

**Description:**
`PDFExportCommand` messages sent to the SQS PDF export queue are JSON payloads with no
message-level signature or authentication. The security boundary relies entirely on IAM
queue access controls. The async interface specification documents the message format but
does not describe the IAM policy required to prevent unauthorized principals from publishing.
This is security finding F-18 (Low/Informational).

**Acceptance Criteria:**

- [ ] `QQ_SD_Interface_Async_Specifications.md` documents the required IAM policy for the
  PDF export SQS queue (which principals may send, which may receive).
- [ ] The deployment runbook documents how to verify that the queue policy is correctly
  configured before go-live.
- [ ] The security report is updated to reflect whether the IAM policy is in place.

**Implementation Hints:**
Add an IAM policy example to `QQ_SD_Interface_Async_Specifications.md` specifying
`sqs:SendMessage` for the Django container's IAM role and `sqs:ReceiveMessage` /
`sqs:DeleteMessage` for the PDF export service role. Reference the AWS SQS resource-based
policy documentation.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Finding F-18
- [QQ_SD_Interface_Async_Specifications.md](../03_system_scope_and_context/QQ_SD_Interface_Async_Specifications.md)

---

#### REC-046: Document Secret Management Strategy for Production

| Field | Value |
|-------|-------|
| **Type** | Task |
| **Priority** | Low |
| **Effort** | S |
| **Affected Components** | `projectsettings`, deployment infrastructure |

**Description:**
The security report's Secret Management Summary notes that all production secrets are
expected to be delivered via environment variables, but it is unknown whether a dedicated
secret manager (AWS Secrets Manager, HashiCorp Vault, Kubernetes Secrets) is used, and
whether secrets are rotated automatically. The deployment view documentation does not
describe secret injection for production. This gap means that deployment reviewers cannot
confirm that production secrets are handled securely.

**Acceptance Criteria:**

- [ ] The deployment view documentation describes the secret injection mechanism used in
  production (e.g. ECS task definition secrets from AWS Secrets Manager, Kubernetes Secrets,
  or Docker Secrets).
- [ ] The documentation states whether automatic secret rotation is implemented and with
  what frequency.
- [ ] A runbook step documents how to rotate `DJANGO_SECRET_KEY` and `CELERY_WORKER_M2M_CLIENT_SECRET`
  without downtime.

**Implementation Hints:**
Update `doc/07_deployment_view/` with a secrets management section. If no secret manager is
in use, document the expectation that secrets are injected via the container runtime's
environment and reference the orchestration platform documentation.

**References:**
- [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) — Secret Management Summary
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md)

---

## Cross-References Between Related Recommendations

| Recommendation | Related To |
|----------------|-----------|
| REC-001 (Remove BasicAuthentication) | REC-007 (TLS settings), REC-015 (DRF permission pipeline) |
| REC-002 (SECRET_KEY) | REC-003 (POSTGRES_PASSWORD), REC-032 (.env.example), REC-040 (production settings) |
| REC-004 (null guard) | REC-021 (subscriptions tests), REC-027 (copy-paste defect), REC-028 (redirect defect) |
| REC-007 (TLS settings) | REC-040 (production settings module) |
| REC-010 (GDPR pipeline) | REC-009 (schema endpoint auth) |
| REC-015 (DRF permission pipeline) | REC-018 (CR-10 object grants) |
| REC-016 (audit log) | REC-038 (structured logging) |
| REC-017 (accounting scope) | REC-015 (permission pipeline) |
| REC-020 (auth tests) | REC-022 (REST validation tests) |
| REC-021 (subscriptions tests) | REC-027 (copy-paste), REC-028 (redirect), REC-030 (field name) |
| REC-027 (copy-paste) | REC-028 (redirect), REC-030 (field name), REC-021 (tests) |
| REC-029 (nested serializer N+1) | REC-042 (reporting querysets) |
| REC-031 (TASK_ROUTES) | REC-041 (Celery health check) |
| REC-032 (.env.example) | REC-034 (startup validation) |
| REC-035 (subscriptions REST API) | REC-021 (subscriptions tests) |

---

## References

| Document | Path |
|----------|------|
| Security Report | [QQ_SD_Security_Report.md](../08_cross_cutting_concepts/QQ_SD_Security_Report.md) |
| Access Control | [QQ_SD_AccessControl.md](../08_cross_cutting_concepts/QQ_SD_AccessControl.md) |
| Unit Test Coverage | [QQ_SD_Unit_Test_Coverage.md](../08_cross_cutting_concepts/QQ_SD_Unit_Test_Coverage.md) |
| Configuration | [QQ_SD_Configuration.md](../08_cross_cutting_concepts/QQ_SD_Configuration.md) |
| Settings | [QQ_SD_Settings.md](../08_cross_cutting_concepts/QQ_SD_Settings.md) |
| High-Level Documentation | [QQ_HL_Doc_KoalixCRM.md](../05_building_block_view/QQ_HL_Doc_KoalixCRM.md) |
| Service Architecture | [QQ_SD_ServiceArchitecture.md](../05_building_block_view/QQ_SD_ServiceArchitecture.md) |
| Component Architecture | [QQ_SD_ComponentArchitecture.md](../05_building_block_view/QQ_SD_ComponentArchitecture.md) |
| Celery Worker Service | [QQ_SD_ServiceDocumentation_CeleryWorker.md](../05_building_block_view/QQ_SD_ServiceDocumentation_CeleryWorker.md) |
| Subscriptions LL | [QQ_LL_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md) |
| Contracts Views/Serializers LL | [QQ_LL_Doc_Contracts_ViewsSerializers.md](../05_building_block_view/koalixcrm/contracts/QQ_LL_Doc_Contracts_ViewsSerializers.md) |
| Auth LL | [QQ_LL_Doc_Auth.md](../05_building_block_view/koalixcrm/auth/QQ_LL_Doc_Auth.md) |
| Project Settings LL | [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md) |
| REST Interface Specifications | [QQ_SD_Interface_REST_Specifications.md](../03_system_scope_and_context/QQ_SD_Interface_REST_Specifications.md) |
| Async Interface Specifications | [QQ_SD_Interface_Async_Specifications.md](../03_system_scope_and_context/QQ_SD_Interface_Async_Specifications.md) |
| Allocation Matrix | [QQ_SD_AllocationMatrix.md](../06_runtime_view/QQ_SD_AllocationMatrix.md) |
