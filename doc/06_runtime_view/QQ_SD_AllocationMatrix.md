# Allocation Matrix — koalixcrm

## Overview

This document provides traceability between the functional architecture (use cases documented in
`doc/06_runtime_view/QQ_SD_Use_Case_*.md`) and the structural architecture (Django application
modules documented in `doc/05_building_block_view/`). It covers three allocation views:

1. **Use Case to Component** — which Django apps actively participate in each use case
2. **Use Case to Interface** — which system interfaces (REST API namespaces, Django Admin,
   management-command CLI, and async queues) each use case enters or consumes
3. **Component to Interface** — which interfaces each Django app provides or consumes

koalixcrm is a modular Django monolith. The structural "components" used in this matrix are the
eight Django business-domain apps that run in the `koalixcrm-django` container plus three
cross-cutting infrastructure elements: the `koalixcrm-celery` worker, the `auth` package, and
the `shared` package. The `subscriptions` app is included because it is a named app in the
service catalog even though it has no REST API.

### Reading the Matrices

**Use Case to Component matrix cell values:**

| Symbol | Meaning |
|---|---|
| **X** | The component actively participates in this use case (processes data, executes business logic, or exposes the entry-point view) |
| **T** | The component is triggered asynchronously by this use case (via SQS signal) |
| *(empty)* | No documented evidence of involvement |

**Use Case to Interface matrix cell values:**

| Symbol | Meaning |
|---|---|
| **E** | This use case enters the system through this interface |
| **C** | This use case calls or consumes this interface during execution |
| *(empty)* | No documented evidence of involvement |

**Component to Interface matrix cell values:**

| Symbol | Meaning |
|---|---|
| **P** | The component provides / exposes this interface |
| **C** | The component consumes this interface |
| *(empty)* | No documented evidence of involvement |

### Component Short Names

| Short Name | Full Name |
|---|---|
| `core` | `koalixcrm.core` Django app |
| `contacts` | `koalixcrm.contacts` Django app |
| `contracts` | `koalixcrm.contracts` Django app |
| `products` | `koalixcrm.products` Django app |
| `accounting` | `koalixcrm.accounting` Django app (optional) |
| `reporting` | `koalixcrm.reporting` Django app (optional) |
| `subscriptions` | `koalixcrm.subscriptions` Django app (optional) |
| `djuserext` | `koalixcrm.djangoUserExtension` Django app |
| `auth-pkg` | `koalixcrm.auth` package (OIDC views, DRF authentication backends) |
| `celery` | `koalixcrm-celery` container (Celery worker + SQS poller) |

### Interface Short Names

| Short Name | Full Interface |
|---|---|
| `REST-core` | REST API `/koalixcrm_core/api/v1/<ws>/` |
| `REST-contacts` | REST API `/koalixcrm_contacts/api/v1/<ws>/` |
| `REST-contracts` | REST API `/koalixcrm_contracts/api/v1/<ws>/` |
| `REST-products` | REST API `/koalixcrm_products/api/v1/<ws>/` |
| `REST-accounting` | REST API `/koalixcrm_accounting/api/v1/<ws>/` |
| `REST-reporting` | REST API `/koalixcrm_reporting/api/v1/<ws>/` |
| `Admin-UI` | Django Admin interface (`/admin/`) across all apps |
| `OIDC` | OIDC Authorization Code Flow / end-session endpoint (external Identity Provider) |
| `CLI` | Django management-command CLI (`manage.py`) |
| `SQS-pdf` | AWS SQS PDF export queue (PDFExportCommand messages) |
| `SQS-ms` | AWS SQS microservice queue (CommandEnvelope messages) |

---

## Use Case to Component Allocation Matrix

Rows are use cases grouped by domain file. Columns are the ten components defined above.
Use case identifiers and names link to the domain use-case files.

| Use Case | `core` | `contacts` | `contracts` | `products` | `accounting` | `reporting` | `subscriptions` | `djuserext` | `auth-pkg` | `celery` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Workspace & Authentication** ([WorkspaceAuth](QQ_SD_Use_Case_WorkspaceAuth.md)) | | | | | | | | | | |
| UC-WA-01 Login via OIDC | X | | | | | | | | X | |
| UC-WA-02 Logout | X | | | | | | | | X | |
| UC-WA-03 Switch Active Workspace | X | | | | | | | | | |
| UC-WA-04 Manage Workspaces | X | | | | | | | | | |
| UC-WA-05 Manage Role Assignments | X | | | | | | | | | |
| UC-WA-06 Initialize Default Templates | X | | | | | | | X | | |
| UC-WA-07 Set Display Timezone | X | | | | | | | | | |
| UC-WA-08 Authenticate via REST API | X | | | | | | | | X | |
| **Contacts** ([Contacts](QQ_SD_Use_Case_Contacts.md)) | | | | | | | | | | |
| UC-CON-01 Manage Organizations | X | X | | | | | | | | |
| UC-CON-02 Manage Personal Contacts | X | X | | | | | | | | |
| UC-CON-03 Convert Org ↔ Contact | X | X | | | | | | | | |
| UC-CON-04 Manage Contact Address Information | X | X | | | | | | | | |
| UC-CON-05 Manage Party Groups | X | X | | | | | | | | |
| UC-CON-06 Manage Party Roles and Memberships | X | X | | | | | | | | |
| UC-CON-07 Manage Organization Relationships | X | X | | | | | | | | |
| **Products & Pricing** ([ProductsPricing](QQ_SD_Use_Case_ProductsPricing.md)) | | | | | | | | | | |
| UC-PP-01 Manage Product Types | X | | | X | | | | | | |
| UC-PP-02 Define Product Pricing Rules | X | X | | X | | | | | | |
| UC-PP-03 Manage Customer Group Price Transforms | X | X | | X | | | | | | |
| UC-PP-04 Manage Currencies, Taxes, and Units | X | | | | | | | | | |
| UC-PP-05 Manage Unit and Currency Conversions | X | | | X | | | | | | |
| UC-PP-06 Assign Product Category | X | | | X | X | | | | | |
| **Contracts & Sales** ([ContractsSales](QQ_SD_Use_Case_ContractsSales.md)) | | | | | | | | | | |
| UC-CS-01 Manage Contracts | X | X | X | | | | | X | | |
| UC-CS-02 Create Commercial Document from Contract | X | X | X | X | | | | X | | |
| UC-CS-03 Manage Commercial Documents | X | X | X | X | | | | X | | |
| UC-CS-04 Convert Between Document Types | X | X | X | X | | | | X | | |
| UC-CS-05 Register Invoice in Accounting | X | | X | | X | | | | | |
| UC-CS-06 Register Payment in Accounting | X | | X | | X | | | | | |
| UC-CS-07 Manage Document Line Items and Attachments | X | X | X | X | | | | X | | |
| UC-CS-08 Manage Subscriptions | X | X | X | | | | X | | | |
| **Accounting** ([Accounting](QQ_SD_Use_Case_Accounting.md)) | | | | | | | | | | |
| UC-ACC-01 Manage Chart of Accounts | X | | | | X | | | | | |
| UC-ACC-02 Manage Accounting Periods | X | | | | X | | | | X | |
| UC-ACC-03 Manage Double-Entry Bookings | X | | | | X | | | | | |
| UC-ACC-04 Manage Product Categories | X | | | X | X | | | | | |
| UC-ACC-05 Assign Tax Accounts | X | | | | X | | | | | |
| UC-ACC-06 Generate Balance Sheet PDF | X | | | | X | | | X | | T |
| UC-ACC-07 Generate P&L Statement PDF | X | | | | X | | | X | | T |
| UC-ACC-08 Register Invoice in Accounting | X | | X | | X | | | | | |
| UC-ACC-09 Register Payment in Accounting | X | | X | | X | | | | | |
| **Reporting & Export** ([ReportingExport](QQ_SD_Use_Case_ReportingExport.md)) | | | | | | | | | | |
| UC-REP-01 Manage Projects and Tasks | X | X | | | | X | | X | | |
| UC-REP-02 Record Work (Formset Screen) | X | | | | | X | | X | | |
| UC-REP-03 Record Work (Admin CRUD) | X | | | | | X | | | | |
| UC-REP-04 Manage Resource Agreements and Estimations | X | | | | | X | | X | | |
| UC-REP-05 Manage Reporting Periods | X | | | | | X | | | | |
| UC-REP-06 Generate Project Report PDF (async) | X | | | | | X | | X | | T |
| UC-REP-07 Generate Work Report PDF (async) | X | | | | | X | | X | | T |
| UC-REP-08 Generate Commercial Document PDF (async) | X | | X | | | | | X | | T |
| UC-REP-09 Poll PDF Export Process Status | X | | | | | | | | | |
| **User Extensions** ([UserExtensions](QQ_SD_Use_Case_UserExtensions.md)) | | | | | | | | | | |
| UC-UEX-01 Manage Document Templates | X | X | | | | | | X | | |
| UC-UEX-02 Manage Template Sets | X | | | | | | | X | | |
| UC-UEX-03 Manage User Extensions | X | | | | | | | X | | |
| UC-UEX-04 Manage User Contact Information | X | X | | | | | | X | | |
| UC-UEX-05 Read Document Template via REST API | X | | | | | | | X | | |
| UC-UEX-06 Bootstrap Default Templates | X | X | | | | | | X | | |

---

## Use Case to Interface Allocation Matrix

Columns represent the system interfaces. **E** = entry point (use case is initiated through this
interface). **C** = consumed (use case calls this interface during execution).

| Use Case | `REST-core` | `REST-contacts` | `REST-contracts` | `REST-products` | `REST-accounting` | `REST-reporting` | `Admin-UI` | `OIDC` | `CLI` | `SQS-pdf` | `SQS-ms` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Workspace & Authentication** | | | | | | | | | | | |
| UC-WA-01 Login via OIDC | | | | | | | E | E | | | |
| UC-WA-02 Logout | | | | | | | E | C | | | |
| UC-WA-03 Switch Active Workspace | | | | | | | E | | | | |
| UC-WA-04 Manage Workspaces | | | | | | | E | | | | |
| UC-WA-05 Manage Role Assignments | | | | | | | E | | | | |
| UC-WA-06 Initialize Default Templates | | | | | | | | | E | | |
| UC-WA-07 Set Display Timezone | | | | | | | E | | | | |
| UC-WA-08 Authenticate via REST API | E | E | E | E | E | E | | C | | | |
| **Contacts** | | | | | | | | | | | |
| UC-CON-01 Manage Organizations | | E | | | | | E | | | | |
| UC-CON-02 Manage Personal Contacts | | E | | | | | E | | | | |
| UC-CON-03 Convert Org ↔ Contact | | | | | | | E | | | | |
| UC-CON-04 Manage Contact Address Information | | E | | | | | E | | | | |
| UC-CON-05 Manage Party Groups | | E | | | | | E | | | | |
| UC-CON-06 Manage Party Roles and Memberships | | E | | | | | E | | | | |
| UC-CON-07 Manage Organization Relationships | | E | | | | | E | | | | |
| **Products & Pricing** | | | | | | | | | | | |
| UC-PP-01 Manage Product Types | | | | E | | | E | | | | |
| UC-PP-02 Define Product Pricing Rules | | | | E | | | E | | | | |
| UC-PP-03 Manage Customer Group Price Transforms | | | | E | | | E | | | | |
| UC-PP-04 Manage Currencies, Taxes, and Units | E | | | | | | E | | | | |
| UC-PP-05 Manage Unit and Currency Conversions | E | | | E | | | E | | | | |
| UC-PP-06 Assign Product Category | | | | | | | E | | | | |
| **Contracts & Sales** | | | | | | | | | | | |
| UC-CS-01 Manage Contracts | | | E | | | | E | | | | |
| UC-CS-02 Create Commercial Document from Contract | | | | | | | E | | | | |
| UC-CS-03 Manage Commercial Documents | | | E | | | | E | | | | |
| UC-CS-04 Convert Between Document Types | | | | | | | E | | | | |
| UC-CS-05 Register Invoice in Accounting | | | | | | | E | | | | |
| UC-CS-06 Register Payment in Accounting | | | | | | | E | | | | |
| UC-CS-07 Manage Document Line Items and Attachments | | | E | | | | E | | | | |
| UC-CS-08 Manage Subscriptions | | | | | | | E | | | | |
| **Accounting** | | | | | | | | | | | |
| UC-ACC-01 Manage Chart of Accounts | | | | | E | | E | | | | |
| UC-ACC-02 Manage Accounting Periods | | | | | E | | E | | | | |
| UC-ACC-03 Manage Double-Entry Bookings | | | | | E | | E | | | | |
| UC-ACC-04 Manage Product Categories | | | | | E | | E | | | | |
| UC-ACC-05 Assign Tax Accounts | | | | | | | E | | | | |
| UC-ACC-06 Generate Balance Sheet PDF | E | | | | E | | E | | | C | |
| UC-ACC-07 Generate P&L Statement PDF | E | | | | E | | E | | | C | |
| UC-ACC-08 Register Invoice in Accounting | | | | | | | E | | | | |
| UC-ACC-09 Register Payment in Accounting | | | | | | | E | | | | |
| **Reporting & Export** | | | | | | | | | | | |
| UC-REP-01 Manage Projects and Tasks | | | | | | E | E | | | | |
| UC-REP-02 Record Work (Formset Screen) | | | | | | C | E | | | | |
| UC-REP-03 Record Work (Admin CRUD) | | | | | | E | E | | | | |
| UC-REP-04 Manage Resource Agreements and Estimations | | | | | | E | E | | | | |
| UC-REP-05 Manage Reporting Periods | | | | | | E | E | | | | |
| UC-REP-06 Generate Project Report PDF (async) | C | | | | | C | E | | | C | |
| UC-REP-07 Generate Work Report PDF (async) | C | | | | | C | E | | | C | |
| UC-REP-08 Generate Commercial Document PDF (async) | C | | C | | | | E | | | C | |
| UC-REP-09 Poll PDF Export Process Status | E | | | | | | E | | | | |
| **User Extensions** | | | | | | | | | | | |
| UC-UEX-01 Manage Document Templates | | | | | | | E | | | | |
| UC-UEX-02 Manage Template Sets | | | | | | | E | | | | |
| UC-UEX-03 Manage User Extensions | | | | | | | E | | | | |
| UC-UEX-04 Manage User Contact Information | | | | | | | E | | | | |
| UC-UEX-05 Read Document Template via REST API | E | | | | | | | | | | |
| UC-UEX-06 Bootstrap Default Templates | | | | | | | | | E | | |

---

## Component to Interface Allocation Matrix

Rows are components. **P** = provides / exposes this interface. **C** = consumes this interface.

| Component | `REST-core` | `REST-contacts` | `REST-contracts` | `REST-products` | `REST-accounting` | `REST-reporting` | `Admin-UI` | `OIDC` | `CLI` | `SQS-pdf` | `SQS-ms` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| `core` | P | | | | | | P | C | P | P | P |
| `contacts` | | P | | | | | P | | P | | |
| `contracts` | | | P | | | | P | | | P | |
| `products` | | | | P | | | P | | | | |
| `accounting` | | | | | P | | P | | | P | |
| `reporting` | | | | | | P | P | | | P | |
| `subscriptions` | | | | | | | P | | | | |
| `djuserext` | C | | | | | | P | | P | | |
| `auth-pkg` | C | | | | | | | C | | | |
| `celery` | | | | | | | | C | | | C |

**Notes on the Component to Interface matrix:**

- `core` provides `REST-core` (currencies, taxes, units, transforms, pdf-export-processes,
  document-templates endpoints), the `Admin-UI` for its own models (Workspace, RoleInWorkspace,
  Currency, Tax, Unit, PDFExportProcess, etc.), and all management commands (`CLI`) shared by
  the application. It consumes `OIDC` indirectly via middleware and the auth package; it
  publishes to `SQS-pdf` via the `post_save` signal dispatcher, and exposes the `SQS-ms`
  command envelope poller through the celery worker.
- `djuserext` registers its admin UI for DocumentTemplate, TemplateSet, UserExtension, and
  assignment models. Its `DocumentTemplateViewSet` is mounted under the `REST-core` prefix
  (not its own prefix), hence `C` rather than `P` for `REST-core`. It consumes `REST-contacts`
  for address/phone/email records and provides management commands via `CLI` (the
  `koalixcrm_install_defaulttemplates` command is in `core`, but its seeding of `djuserext`
  objects is performed from the same command).
- `auth-pkg` consumes `OIDC` (IdP token endpoint, JWKS endpoint) and makes its authentication
  classes available to the DRF pipeline for all REST interfaces; it does not independently
  own any REST namespace, hence `C` for `REST-core`.
- `celery` consumes `SQS-ms` (the CommandEnvelope queue) and `OIDC` (M2M client-credentials
  grant for the worker token).
- `accounting` provides `REST-accounting` and its Admin UI. It also enqueues `PDFExportProcess`
  records (triggering `SQS-pdf` dispatch via `core`'s signal), hence `P` for `SQS-pdf` is
  attributed to the originating push (the signal itself lives in `core`; `P` here reflects
  which app's admin action initiates the flow).
- `reporting` similarly initiates `SQS-pdf` dispatch from its project/human-resource admin
  actions.
- `contracts` initiates `SQS-pdf` dispatch from `OptionCommercialDocument.create_pdf` admin
  action.

---

## Coverage Analysis

### Summary Statistics

| Metric | Value |
|---|---|
| Total use cases | 52 |
| Total components | 10 |
| Total interfaces | 11 |
| Use cases with at least one component allocated | 52 (100 %) |
| Components allocated to at least one use case | 10 (100 %) |
| Interfaces allocated to at least one component | 11 (100 %) |

### Component Coverage Detail

| Component | Use Cases Involving This Component | Count | % of 52 |
|---|---|---|---|
| `core` | All 52 use cases | 52 | 100 % |
| `djuserext` | UC-WA-06, UC-CS-01–04, UC-CS-07, UC-ACC-02, UC-ACC-06–07, UC-REP-01–02, UC-REP-04, UC-REP-06–08, UC-UEX-01–06 | 22 | 42 % |
| `contacts` | UC-CON-01–07, UC-PP-02–03, UC-CS-01–04, UC-CS-07–08, UC-REP-01, UC-UEX-01, UC-UEX-04, UC-UEX-06 | 18 | 35 % |
| `contracts` | UC-CS-01–08, UC-ACC-08–09, UC-REP-08 | 11 | 21 % |
| `reporting` | UC-REP-01–09 | 9 | 17 % |
| `accounting` | UC-PP-06, UC-CS-05–06, UC-ACC-01–09 | 13 | 25 % |
| `products` | UC-PP-01–03, UC-PP-05–06, UC-CS-02–04, UC-CS-07, UC-ACC-04 | 9 | 17 % |
| `auth-pkg` | UC-WA-01–02, UC-WA-08 | 3 | 6 % |
| `celery` | UC-ACC-06–07, UC-REP-06–08 (triggered asynchronously) | 5 | 10 % |
| `subscriptions` | UC-CS-08 | 1 | 2 % |

### High-Coupling Components (> 60 % use case participation)

| Component | Participation | Note |
|---|---|---|
| `core` | 100 % (52 / 52) | Every use case passes through `core` because `WorkspaceScopedModel`, `WorkspaceContextMiddleware`, `PDFExportProcess`, and shared lookup models (Currency, Tax, Unit) are infrastructure elements used by all other apps. `core` also hosts the `PDFExportProcess` Admin screen and the management commands referenced by two use cases. High coupling here reflects intentional infrastructure role, not design instability. |

### Orphan Components

No orphan components were found. All ten components are referenced by at least one use case.
The `subscriptions` app participates only in UC-CS-08 (Manage Subscriptions), which is the
lowest coverage of any component. Should the subscriptions plugin be removed from a deployment,
it would become a genuine orphan with respect to the remaining use-case set.

### Unallocated Use Cases

No unallocated use cases were found. All 52 use cases have at least one component allocated.

### Interface Coverage Summary

| Interface | Components Providing It | Components Consuming It | Use Cases Using It |
|---|---|---|---|
| `REST-core` | `core` | `djuserext`, `auth-pkg` | UC-WA-08, UC-PP-04–05, UC-ACC-06–07, UC-REP-06–09, UC-UEX-05 |
| `REST-contacts` | `contacts` | — | UC-WA-08, UC-CON-01–07, UC-PP-02–03 |
| `REST-contracts` | `contracts` | — | UC-WA-08, UC-CS-01, UC-CS-03, UC-CS-07, UC-REP-08 |
| `REST-products` | `products` | — | UC-WA-08, UC-PP-01–03, UC-PP-05 |
| `REST-accounting` | `accounting` | — | UC-WA-08, UC-ACC-01–04, UC-ACC-06–07 |
| `REST-reporting` | `reporting` | — | UC-WA-08, UC-REP-01–07 |
| `Admin-UI` | `core`, `contacts`, `contracts`, `products`, `accounting`, `reporting`, `subscriptions`, `djuserext` | — | All use cases except UC-WA-08, UC-WA-06, UC-UEX-06 |
| `OIDC` | — (external) | `core`, `auth-pkg`, `celery` | UC-WA-01–02, UC-WA-08 |
| `CLI` | `core`, `contacts`, `djuserext` | — | UC-WA-06, UC-UEX-06 |
| `SQS-pdf` | `core` (dispatch), `accounting`, `reporting`, `contracts` (initiate) | `celery`? | UC-ACC-06–07, UC-REP-06–08 (publish); UC-REP-09 (poll) |
| `SQS-ms` | `core` (poller config), `celery` | — | UC-WA-08 (Celery M2M auth path) |

**Note on `SQS-ms`:** The microservice SQS queue (`CommandEnvelope`) is consumed by the `celery`
worker, but `TASK_ROUTES` is currently empty. No use case routes functional work through this
queue at present. The interface is allocated because the queue infrastructure is live and the
Celery worker is a documented actor.

### Observations

1. **`core` as universal cross-cut.** The `core` app participates in every use case because
   it provides the workspace-isolation substrate (`WorkspaceScopedModel`, middleware, manager)
   and the PDF export pipeline (`PDFExportProcess`, signals, dispatcher) that underpin all
   other apps. This is expected given the design intent documented in
   [QQ_SD_ServiceArchitecture.md](../05_building_block_view/QQ_SD_ServiceArchitecture.md).

2. **`subscriptions` single-use-case coverage.** The `subscriptions` app is referenced by
   exactly one use case (UC-CS-08). If the deployment profile removes this optional app, no
   other use case is affected.

3. **`celery` participates only via async trigger.** The Celery worker is always listed with
   a **T** (triggered) marker, never **X**. This reflects the documented state: `TASK_ROUTES`
   is empty. The worker receives SQS messages but dispatches no Python tasks. The PDF rendering
   pipeline bypasses the worker entirely and is handled by the external Java PDF Export Service.

4. **No REST API for `djuserext` directly.** The `djangoUserExtension` app's single REST
   exposure (`document-templates/`) is mounted under the `REST-core` prefix, not under a
   dedicated namespace. All write operations for this domain are Admin-only. This is reflected
   in the Component to Interface matrix where `djuserext` is shown as a consumer of `REST-core`
   rather than a provider.

5. **`accounting` models are not workspace-scoped.** The `accounting` app's models (`Account`,
   `AccountingPeriod`, `Booking`) are global records. The URL path carries a `<workspace_id>`
   segment for routing consistency, but the data is not filtered by workspace. This is a noted
   structural discrepancy documented in
   [QQ_SD_Use_Case_Accounting.md](QQ_SD_Use_Case_Accounting.md).

---

## Cross-References

- Use case detail files — [Chapter 06: Runtime View](index.md):
  - [QQ_SD_Use_Case_WorkspaceAuth.md](QQ_SD_Use_Case_WorkspaceAuth.md)
  - [QQ_SD_Use_Case_Contacts.md](QQ_SD_Use_Case_Contacts.md)
  - [QQ_SD_Use_Case_ProductsPricing.md](QQ_SD_Use_Case_ProductsPricing.md)
  - [QQ_SD_Use_Case_ContractsSales.md](QQ_SD_Use_Case_ContractsSales.md)
  - [QQ_SD_Use_Case_Accounting.md](QQ_SD_Use_Case_Accounting.md)
  - [QQ_SD_Use_Case_ReportingExport.md](QQ_SD_Use_Case_ReportingExport.md)
  - [QQ_SD_Use_Case_UserExtensions.md](QQ_SD_Use_Case_UserExtensions.md)
- Building block view — [Chapter 05: Building Block View](../05_building_block_view/):
  - [QQ_SD_ServiceArchitecture.md](../05_building_block_view/QQ_SD_ServiceArchitecture.md)
  - [QQ_SD_ComponentArchitecture.md](../05_building_block_view/QQ_SD_ComponentArchitecture.md)
- Entry points and URL routing — [Chapter 03: System Scope and Context](../03_system_scope_and_context/):
  - [QQ_SD_EntryPoints.md](../03_system_scope_and_context/QQ_SD_EntryPoints.md)
