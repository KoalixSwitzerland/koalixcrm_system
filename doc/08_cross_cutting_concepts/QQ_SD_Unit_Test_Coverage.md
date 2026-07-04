# Unit Test Coverage and Strategy

## Introduction

This document describes the unit test coverage, methodology, and strategy for the koalixcrm
project. It is based on direct inspection of the `tests/` directory tree, the pytest
configuration files (`pytest.ini`, `conftest.py`), the GitHub Actions workflow files under
`.github/workflows/`, and the relevant sections of the mid-level and high-level documentation
files in `doc/05_building_block_view/`.

---

## Test Infrastructure

### Frameworks and Tools

| Tool | Role |
|------|------|
| **pytest** | Primary test runner; configured via `pytest.ini` |
| **pytest-django** | Django database access; `@pytest.mark.django_db` and the `db` fixture |
| **factory_boy** (`factory.django.DjangoModelFactory`) | Model instance creation in all Django-backed test suites |
| **Django `TestCase` / `LiveServerTestCase`** | Base class for model and REST API tests that require database fixtures and a live HTTP server |
| **Django `StaticLiveServerTestCase`** | Base class for end-to-end Selenium tests |
| **Selenium / Chrome headless** | Browser automation for E2E tests in `tests/e2e/` |
| **DRF `APIClient`** | REST API access in `core_api_py` and accounting admin action tests |
| **`unittest.mock`** | Mocking S3/presigned-URL calls in PDF endpoint tests and admin action tests |
| **boto3** | AWS SDK used in integration smoke tests (S3 and SQS round-trips) |
| **coverage / Codacy** | Line and branch coverage reporting; XML report uploaded to Codacy |

### pytest Configuration

The `pytest.ini` file at the repository root sets the Django settings module to
`projectsettings.settings.development_docker_sqlite_settings` (an in-memory SQLite database),
which makes the full test suite runnable without a running PostgreSQL instance. Four custom
markers are declared:

| Marker | Purpose |
|--------|---------|
| `back_end_tests` | Model and business-logic tests |
| `front_end_tests` | Selenium-driven E2E tests |
| `version_increase` | Tests that verify the version-bump workflow |
| `integration` | Tests that require a live Docker Compose stack (MinIO, ElasticMQ, Celery) |

### Root `conftest.py`

The root-level `conftest.py` provides two shared fixtures available to all test modules:

- `admin_user` (`db` scope) — creates a Django superuser with `username="admin"` and
  `password="adminpassword"`. Django is imported lazily so the conftest remains importable
  under the `unit-celery` pytest profile, which runs without a configured
  `DJANGO_SETTINGS_MODULE`.
- `use_m2m_auth` — returns `True` when the `CELERY_WORKER_M2M_OIDC_ISSUER` environment
  variable is set, enabling integration tests to switch from BasicAuth to M2M OIDC token
  authentication.

---

## Test Directory Structure

```text
tests/
├── factories/          # factory_boy factories for all domain models
│   ├── accounting/
│   ├── contacts/
│   ├── contracts/
│   ├── core/
│   ├── djangoUserExtension/
│   ├── products/
│   └── reporting/
├── unit/               # Architectural invariant tests (no Django DB required)
├── integration/        # Infrastructure smoke tests (require Docker Compose)
├── e2e/                # Selenium browser tests
├── legacy_crm/         # Model-level business logic tests (reporting + contracts)
├── accounting/         # Accounting model and admin-action tests
├── accounting_api_py/  # REST API client tests for accounting
├── contacts/           # Contacts model and admin-action tests
├── contacts_api_py/    # REST API client tests for contacts
├── contracts/          # Shared Selenium helper functions (no test cases)
├── contracts_api_py/   # REST API client tests for contracts
├── core_api_py/        # REST API, access-control, and workspace tests for core
├── products_api_py/    # REST API client tests for products
└── reporting_api_py/   # REST API client tests for reporting
```

---

## Test Strategy and Philosophy

### Three Execution Profiles

The GitHub Actions workflow (`.github/workflows/test.yml`) defines three independent
CI jobs, each using a dedicated Docker Compose profile:

| CI Job | Docker Compose Profile | What Runs |
|--------|------------------------|-----------|
| Unit Tests (Django) | `unit-django` | All tests except those marked `integration` and the Celery-only subset; runs against SQLite |
| Unit Tests (Celery) | `unit-celery` | Tests scoped to `koalixcrm_microservices/` with `-p no:django` (no Django settings configured) |
| Integration Tests | `integration` | Tests marked `@pytest.mark.integration`; requires live MinIO, ElasticMQ, and Keycloak |

Coverage XML reports are produced by each job: Django coverage is uploaded to Codacy via
`codacy/codacy-coverage-reporter-action@v1` and also archived as a GitHub Actions artifact
(`test_report/coverage.xml`). The Celery coverage is archived as `test_report/coverage-celery.xml`.

### Factory-Based Fixture Construction

All Django model tests rely on `factory_boy` factories rather than raw `Model.objects.create`
calls or Django fixtures. Factories are located in `tests/factories/` and are organised to
mirror the app structure (`accounting/`, `contacts/`, `contracts/`, `core/`,
`djangoUserExtension/`, `products/`, `reporting/`). This approach produces self-contained,
composable test data without fixture files, and allows individual test cases to override
specific factory fields via keyword arguments.

### Architectural Invariant Tests

Two tests in `tests/unit/` enforce structural constraints that cannot be verified by
running the application:

**`test_mq_commands_is_django_free.py` (CR-3)**
Launches a subprocess with no `DJANGO_SETTINGS_MODULE` configured and attempts to import
`koalixcrm_mq_commands`. Any leaked `django.*` module in `sys.modules` causes the test to fail.
This guarantees that the dataclass-only MQ command package can be copied verbatim to other
services (such as the WFS fork) without dragging the Django stack along.

**`test_fork_isolation.py` (CR-5)**
Walks every `*.py` source file under the five public fork-surface apps (`core`, `contacts`,
`contracts`, `djangoUserExtension`, `products`) using Python's `ast` module and a regular
expression. It asserts that no module-level `import` or `from ... import` statement, and no
string-based `ForeignKey` target, references the three optional apps (`reporting`, `accounting`,
`subscriptions`). The test is parameterised over the five apps, producing ten separate
pytest nodes (two checks per app). Lazy imports inside function bodies are intentionally
excluded from the check because they do not execute at import time.

### Model Business-Logic Tests (`legacy_crm/`)

Seventeen test files in `tests/legacy_crm/` exercise the cost and effort computation methods
on `Task` and `Project` in the `reporting` app, as well as document price calculations in
`contracts`. Tests use `django.test.TestCase` with factory-constructed data and call Django
model methods directly, comparing their return values using `assertEqual`. These cover:

- Task planned/effective duration, effort, and costs (with and without a resource price agreement)
- Project planned and effective costs
- Document price calculation under varied date ranges, customer groups, currency transforms, unit
  transforms, discounts, and currency rounding
- Incomplete sales document position handling
- Work-entry deletion cascade
- Version-number increase workflow
- Task status-update timestamp propagation

### REST API Tests (`*_api_py/` directories)

Each domain app that exposes a REST API has a corresponding `tests/<app>_api_py/` directory.
Tests in these directories instantiate `django.test.LiveServerTestCase` and use the
generated Python API clients (`KoalixCRM*APIClient`) to exercise the full HTTP round-trip
against a real Django dev server backed by SQLite. Each test file covers one resource type
and verifies the standard CRUD surface: list, read, write (create), and modify (partial update).

### Integration Tests (`tests/integration/`)

`test_infra_smoke.py` is decorated with `@pytest.mark.integration` and is excluded from the
unit-django CI job. It exercises three cross-component paths:

- MinIO S3 round-trip: PUT, GET, and DELETE of a test object on the PDF bucket
- ElasticMQ SQS round-trip: create an ephemeral queue, send a `CommandEnvelope`, receive and
  verify it, then delete the message
- Django backend reachability: polls `http://backend:8000/admin/login/` with up to thirty
  retries, accepting HTTP 200 or 302

These tests require environment variables for `S3_ENDPOINT_URL` and `SQS_ENDPOINT_URL` and
are only executed under the `integration` Docker Compose profile.

### End-to-End Tests (`tests/e2e/`)

Seven Selenium test cases in `tests/e2e/` automate browser interactions against the Django
admin. They are marked `@pytest.mark.front_end_tests`. All extend `UITests` (in
`tests/e2e/UITests.py`), which sets up Chrome headless, captures screenshots and page source
on failure, and quits the driver in `tearDown`. Test cases cover:

- Creating quotations, invoices, and purchase orders from a contract
- Creating sales documents from an existing invoice and from an existing quote
- Deleting empty time-tracking rows
- The project admin view
- Adding a time-tracking row
- The user-not-a-human-resource error path
- Viewing the work entry form

---

## Coverage Table by Module

| Module | Test Directory / Files | Test Functions | Coverage Description |
|--------|------------------------|----------------|----------------------|
| `koalixcrm.core` (workspace, RBAC, access helpers, PDF export endpoints, workspace-aware manager, middleware) | `tests/core_api_py/` (6 files) | 84 | Workspace model fields, `Role` enum, `RoleInWorkspace` uniqueness, `effective_roles()` edge cases, `user_workspaces()` filtering, `permissions_for_role()` for all seven roles, `WorkspaceSwitchView` happy path and 403 path, `WorkspaceAwareManager` scoping, `raise_on_missing_context` branch, `workspace_context()` context manager, `WorkspaceContextMiddleware` single/multiple/none workspace paths, `WorkspaceScopedModelAdmin.save_model`, `PDFExportProcess` workspace scoping and `visible_to()`, PDF-service endpoints with mocked S3 presigned URLs |
| `koalixcrm.contacts` (Party conversion admin actions) | `tests/contacts/` (2 files) | 20 | `convert_organizations_to_contacts` and `convert_contacts_to_organizations` bulk admin actions: address and role preservation, single-word name split, membership removal, already-converted party skip guard, full round-trip; party model fields |
| `koalixcrm.contracts` (document price calculations, incomplete positions) | `tests/legacy_crm/` (partial) | 11 | `Calculations.calculate_document_price` under 10 combinations: no customer group, date-boundary variants, currency rounding on/off, document-level discount, customer-group transform, currency transform, unit transform; incomplete document position handling |
| `koalixcrm.accounting` (bookkeeping model, accounting period aggregates, admin PDF actions) | `tests/accounting/` (2 files) | 10 | `Account.sum_of_all_bookings`, `sum_of_all_bookings_before_accounting_period`, `sum_of_all_bookings_within_accounting_period`; `AccountingPeriod.overall_liabilities`, `overall_assets`, `overall_earnings`, `overall_spendings`; accounting period admin actions that queue `PDFExportProcess` records (balance sheet and P&L export) |
| `koalixcrm.reporting` (task cost, effort, duration; project costs) | `tests/legacy_crm/` (partial) | 16 | `Task.planned_duration`, `planned_effort`, `planned_costs`, `effective_effort`, `effective_costs` (with and without a resource price agreement), `effective_duration`, `last_status_update`; `Project.planned_costs`, `effective_costs`; `Work` delete cascade; task constructor |
| `koalixcrm.accounting` REST API | `tests/accounting_api_py/` (5 files) | 21 | List, read, write, and modify for `Account`, `AccountingPeriod`, `Booking`, `BookingSums`, and `ProductCategory` via the REST API client |
| `koalixcrm.contacts` REST API | `tests/contacts_api_py/` (1 file) | 4 | List, read, write, and modify for `CustomerBillingCycle` via the REST API client |
| `koalixcrm.contracts` REST API | `tests/contracts_api_py/` (3 files) | 12 | List, read, write, and modify for `Contract`, `Invoice`, and `Quotation` via the REST API client |
| `koalixcrm.products` REST API | `tests/products_api_py/` (1 file) | 4 | List, read, write, and modify for `ProductType` via the REST API client |
| `koalixcrm.reporting` REST API | `tests/reporting_api_py/` (13 files) | 52 | List, read, write, and modify for `Agreement`, `AgreementStatus`, `AgreementType`, `Estimation`, `EstimationStatus`, `HumanResource`, `Project`, `ProjectStatus`, `ReportingPeriod`, `ReportingPeriodStatus`, `Task`, `TaskStatus`, and `Work` via the REST API client |
| `koalixcrm_mq_commands` (Django-free invariant) | `tests/unit/test_mq_commands_is_django_free.py` | 1 | Subprocess import check — verifies no `django.*` module appears in `sys.modules` after importing `koalixcrm_mq_commands` |
| Fork isolation invariant | `tests/unit/test_fork_isolation.py` | 2 (10 nodes) | AST-level and regex-level scan of all `*.py` files in the five public apps for top-level imports of and string FK references to optional apps (`reporting`, `accounting`, `subscriptions`) |
| Integration (MinIO, ElasticMQ, Django backend) | `tests/integration/test_infra_smoke.py` | 3 | S3 object round-trip, SQS `CommandEnvelope` round-trip, Django admin endpoint reachability (conditional on `integration` marker) |
| E2E (Django Admin / Selenium) | `tests/e2e/` (7 files) | 8 | Admin browser flows: document creation from contracts/invoices/quotes, time-tracking row operations, project admin view, work entry form, user-not-human-resource guard |
| `koalixcrm.subscriptions` | None | 0 | No test files identified |
| `koalixcrm.djangoUserExtension` | Indirect only | 0 direct | Factories for `DocumentTemplate` and `TemplateSet` are used in contract and accounting tests, but no dedicated test file exercises `djangoUserExtension` models or views directly |
| `koalixcrm.auth` | None | 0 | No test files identified |
| `koalixcrm.shared` | None | 0 | No test files identified |
| `koalixcrm_microservices` | None (Celery profile) | 0 identified | The `unit-celery` CI profile targets `koalixcrm_microservices/`; no test source files are present in the `tests/` tree for this package based on the reviewed directory listing |
| `koalixcrm_utils` | None | 0 | No test files identified |

---

## Key Gaps

The following modules or concerns have no identified direct test coverage:

- **`koalixcrm.subscriptions`** — the module is not tested. The absence is noted in
  [QQ_ML_Doc_Subscriptions.md](../05_building_block_view/koalixcrm/subscriptions/QQ_ML_Doc_Subscriptions.md).
- **`koalixcrm.auth`** — the OIDC, Bearer JWT, and M2M authentication classes have no
  dedicated unit tests. The authentication chain is exercised indirectly by API tests that
  use BasicAuth, but the OIDC and M2M code paths are not covered in the reviewed test suite.
- **`koalixcrm.shared`** — `BaseModelViewSet`, `WorkspaceScopedViewSetMixin`, and
  `BaseAPIClient` are exercised indirectly through all API tests but have no dedicated unit
  tests verifying edge cases (e.g., workspace mismatch rejection at the ViewSet level, missing
  `workspace_id` in the URL).
- **`koalixcrm_utils`** — AWS client factories, S3 template storage, and presigned URL
  generation are mocked in the PDF endpoint tests but have no isolated unit tests.
- **`koalixcrm.djangoUserExtension`** — factories for `DocumentTemplate` and `TemplateSet`
  are used by other tests, but the extension's own models (UserExtension, serializers) are not
  tested directly.
- **REST API write-validation paths** — the `*_api_py` tests exercise the happy CRUD path.
  Validation error responses (missing required fields, wrong workspace, permission denial for
  non-superusers) are not covered in the identified tests.
- **Admin views beyond accounting** — accounting admin PDF actions are tested; equivalent
  admin bulk-action tests for `contracts`, `reporting`, and `products` admin views are not
  identified.

---

## CI/CD Integration

Two GitHub Actions workflow files are relevant:

**`.github/workflows/test.yml`** — The current active test pipeline with three parallel jobs:

- `unit-django-tests`: pulls the `koalixcrm_system_config` Docker image from GHCR, brings up
  the `unit-django` Docker Compose profile, and runs the Django unit test suite. Coverage is
  reported to Codacy and archived as `django-coverage-xml`.
- `unit-celery-tests`: same infrastructure, `unit-celery` profile, archives
  `celery-coverage-xml`.
- `integration-tests`: same infrastructure, `integration` profile, requires the
  `KOALIXCRM_SECRETS_ENV` GitHub Actions secret which contains all runtime credentials
  (Keycloak, MinIO, ElasticMQ).

The pipeline is triggered on push and pull-request events targeting the `main` and `develop`
branches.

**`.github/workflows/django.yml`** — An older CI definition (targets `master`/`development`
branches) that ran `coverage run -m pytest` directly inside a Docker Compose service and
piped results to Codacy. This file is superseded by `test.yml` but remains in the repository.

---

## References

- `pytest.ini` — Pytest configuration: settings module and custom markers
- `conftest.py` — Shared fixtures: `admin_user`, `use_m2m_auth`
- `tests/unit/test_fork_isolation.py` — Fork isolation invariant (CR-5)
- `tests/unit/test_mq_commands_is_django_free.py` — Django-free MQ commands invariant (CR-3)
- `tests/integration/test_infra_smoke.py` — Infrastructure smoke tests
- `.github/workflows/test.yml` — Active CI pipeline
- [QQ_HL_Doc_KoalixCRM.md](../05_building_block_view/QQ_HL_Doc_KoalixCRM.md) — High-level documentation; Testing section
- [QQ_LL_Doc_ProjectSettings.md](../05_building_block_view/projectsettings/QQ_LL_Doc_ProjectSettings.md) — Settings hierarchy and SQLite test settings
- [QQ_ML_Doc_Reporting.md](../05_building_block_view/koalixcrm/reporting/QQ_ML_Doc_Reporting.md) — Reporting module documentation
- [QQ_ML_Doc_Accounting.md](../05_building_block_view/koalixcrm/accounting/QQ_ML_Doc_Accounting.md) — Accounting module documentation
- [QQ_ML_Doc_Core.md](../05_building_block_view/koalixcrm/core/QQ_ML_Doc_Core.md) — Core module documentation
