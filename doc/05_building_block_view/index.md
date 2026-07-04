# Building Block View

This chapter describes the static decomposition of koalixcrm into building blocks — services,
packages, modules, and source files — and their dependencies. The structure mirrors the source
code package hierarchy under the repository root.

## Decomposition Overview

koalixcrm is organised into five layers of documentation:

1. **Service Architecture** — the two deployable containers (`koalixcrm-django` and
   `koalixcrm-celery`) and the external systems they depend on.
2. **Component Architecture** — the internal Django app and package structure within each
   container, including the peer-dependency graph.
3. **High-Level (HL)** — one document per top-level package, covering the overall purpose,
   responsibilities, and relationships of the package.
4. **Mid-Level (ML)** — one document per module or Django app, covering the internal structure,
   class hierarchy, and inter-module dependencies.
5. **Low-Level (LL)** — one document per source file or logical cluster of source files,
   covering class diagrams, method-level flow diagrams, and implementation details.

## Service and Component Architecture

| Document | Description |
|---|---|
| [QQ_SD_ServiceArchitecture.md](QQ_SD_ServiceArchitecture.md) | Runtime service topology, container catalog, modular boundary enforcement, and SQS offload path |
| [QQ_SD_ComponentArchitecture.md](QQ_SD_ComponentArchitecture.md) | Internal package structure for each of the eight Django apps and the peer-dependency graph |

## Service Documentation

| Document | Description |
|---|---|
| [QQ_SD_ServiceDocumentation_DjangoApp.md](QQ_SD_ServiceDocumentation_DjangoApp.md) | Detailed documentation of the `koalixcrm-django` service: Gunicorn configuration, WSGI entry point, request handling, and settings loading |
| [QQ_SD_ServiceDocumentation_CeleryWorker.md](QQ_SD_ServiceDocumentation_CeleryWorker.md) | Detailed documentation of the `koalixcrm-celery` service: Celery app configuration, Beat scheduler, SQS poller, and M2M authentication |

## High-Level Documentation

| Document | Description |
|---|---|
| [QQ_HL_Doc_KoalixCRM.md](QQ_HL_Doc_KoalixCRM.md) | High-level overview of the entire koalixcrm codebase: architecture overview, domain model, design patterns, and testing approach |

## Mid-Level Documentation

| Package | Document | Description |
|---|---|---|
| `koalixcrm/accounting` | [QQ_ML_Doc_Accounting.md](koalixcrm/accounting/QQ_ML_Doc_Accounting.md) | Double-entry bookkeeping app: accounts, periods, bookings, balance sheet and P&L PDF generation |
| `koalixcrm/auth` | [QQ_ML_Doc_Auth.md](koalixcrm/auth/QQ_ML_Doc_Auth.md) | OIDC and JWT authentication layer: Authorization Code Flow, Bearer JWT validation, M2M client credentials |
| `koalixcrm/contacts` | [QQ_ML_Doc_Contacts.md](koalixcrm/contacts/QQ_ML_Doc_Contacts.md) | Contact management app: Party hierarchy (Organization, PartyContact), addresses, phone numbers, emails, party groups and roles |
| `koalixcrm/contracts` | [QQ_ML_Doc_Contracts.md](koalixcrm/contracts/QQ_ML_Doc_Contracts.md) | Commercial document lifecycle app: contracts, quotations, sales orders, invoices, purchase orders, credit notes |
| `koalixcrm/core` | [QQ_ML_Doc_Core.md](koalixcrm/core/QQ_ML_Doc_Core.md) | Core infrastructure app: workspace model, multi-tenant isolation (ContextVar), peer-dependency system checks, PDF export dispatch |
| `koalixcrm/djangoUserExtension` | [QQ_ML_Doc_DjangoUserExtension.md](koalixcrm/djangoUserExtension/QQ_ML_Doc_DjangoUserExtension.md) | User extension app: document templates, template sets, user profile extensions, and contact assignments |
| `koalixcrm/products` | [QQ_ML_Doc_Products.md](koalixcrm/products/QQ_ML_Doc_Products.md) | Product catalog app: product types, pricing rules, customer group transforms, currencies, taxes, and units |
| `koalixcrm/reporting` | [QQ_ML_Doc_Reporting.md](koalixcrm/reporting/QQ_ML_Doc_Reporting.md) | Project reporting app: projects, tasks, work records, reporting periods, resource agreements, async PDF report generation |
| `koalixcrm/shared` | [QQ_ML_Doc_Shared.md](koalixcrm/shared/QQ_ML_Doc_Shared.md) | Shared utilities: `WorkspaceScopedModel`, `WorkspaceAwareManager`, `BaseModelViewSet`, workspace-scoped mixins |
| `koalixcrm/subscriptions` | [QQ_ML_Doc_Subscriptions.md](koalixcrm/subscriptions/QQ_ML_Doc_Subscriptions.md) | Subscriptions optional app: recurring subscription management injected into contracts admin via plugin interface |
| `koalixcrm_microservices` | [QQ_ML_Doc_Microservices.md](koalixcrm_microservices/QQ_ML_Doc_Microservices.md) | Celery application package: Celery app configuration, task routing, SQS command envelope processing |
| `koalixcrm_utils` | [QQ_ML_Doc_Utils.md](koalixcrm_utils/QQ_ML_Doc_Utils.md) | Utility package: SQS dispatch helpers, S3 storage utilities, and boto3 wrappers |

## Low-Level Documentation

### `koalixcrm/accounting`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Accounting.md](koalixcrm/accounting/QQ_LL_Doc_Accounting.md) | Models, admin registrations, and service logic for the accounting app |
| [QQ_LL_Doc_AccountingApiPy.md](koalixcrm/accounting_api_py/QQ_LL_Doc_AccountingApiPy.md) | DRF ViewSets and serializers for the accounting REST API |

### `koalixcrm/auth`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Auth.md](koalixcrm/auth/QQ_LL_Doc_Auth.md) | OIDC Authorization Code Flow, JWT validation, M2M authentication, and session management |

### `koalixcrm/contacts`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Contacts_Models.md](koalixcrm/contacts/QQ_LL_Doc_Contacts_Models.md) | Party, Organization, PartyContact, Address, PhoneNumber, and Email models |
| [QQ_LL_Doc_Contacts_ViewsAdminManagement.md](koalixcrm/contacts/QQ_LL_Doc_Contacts_ViewsAdminManagement.md) | Django Admin registrations, bulk actions (convert Organization to Contact), and REST API views |
| [QQ_LL_Doc_ContactsApiPy.md](koalixcrm/contacts_api_py/QQ_LL_Doc_ContactsApiPy.md) | DRF ViewSets and serializers for the contacts REST API |

### `koalixcrm/contracts`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Contracts_Admin.md](koalixcrm/contracts/QQ_LL_Doc_Contracts_Admin.md) | Django Admin registrations, inline management, and bulk actions for commercial documents |
| [QQ_LL_Doc_Contracts_Models.md](koalixcrm/contracts/QQ_LL_Doc_Contracts_Models.md) | Contract, CommercialDocument MTI hierarchy, and related models |
| [QQ_LL_Doc_Contracts_ViewsSerializers.md](koalixcrm/contracts/QQ_LL_Doc_Contracts_ViewsSerializers.md) | DRF serializers and ViewSets for the contracts REST API |
| [QQ_LL_Doc_ContractsApiPy.md](koalixcrm/contracts_api_py/QQ_LL_Doc_ContractsApiPy.md) | Per-app API router and URL configuration for contracts |

### `koalixcrm/core`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Core_Infrastructure.md](koalixcrm/core/QQ_LL_Doc_Core_Infrastructure.md) | AppConfig, peer-dependency system checks, PDF export signal handler, workspace middleware, and management commands |
| [QQ_LL_Doc_Core_Models.md](koalixcrm/core/QQ_LL_Doc_Core_Models.md) | Workspace, RoleInWorkspace, PDFExportProcess, and WorkspaceScopedModel base models |
| [QQ_LL_Doc_CoreApiPy.md](koalixcrm/core_api_py/QQ_LL_Doc_CoreApiPy.md) | DRF ViewSets and serializers for the core REST API (workspaces, PDF export processes) |

### `koalixcrm/djangoUserExtension`

| Document | Description |
|---|---|
| [QQ_LL_Doc_DjangoUserExtension.md](koalixcrm/djangoUserExtension/QQ_LL_Doc_DjangoUserExtension.md) | UserExtension, DocumentTemplate MTI hierarchy, TemplateSet, and contact assignment models |

### `koalixcrm/products`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Products.md](koalixcrm/products/QQ_LL_Doc_Products.md) | ProductType, ProductPrice, CustomerGroupTransform, Tax, Unit, Currency, and UnitTransform models |
| [QQ_LL_Doc_ProductsApiPy.md](koalixcrm/products_api_py/QQ_LL_Doc_ProductsApiPy.md) | DRF ViewSets and serializers for the products REST API |

### `koalixcrm/reporting`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Reporting_ProjectTaskModels.md](koalixcrm/reporting/QQ_LL_Doc_Reporting_ProjectTaskModels.md) | Project, Task, Work, and ReportingPeriod models |
| [QQ_LL_Doc_Reporting_ResourceAgreementModels.md](koalixcrm/reporting/QQ_LL_Doc_Reporting_ResourceAgreementModels.md) | ResourceAgreement, Estimation, and HumanResource models |
| [QQ_LL_Doc_Reporting_ServicesAdmin.md](koalixcrm/reporting/QQ_LL_Doc_Reporting_ServicesAdmin.md) | Django Admin registrations, async PDF report generation actions, and time-tracking view |
| [QQ_LL_Doc_Reporting_ViewsSerializers.md](koalixcrm/reporting/QQ_LL_Doc_Reporting_ViewsSerializers.md) | DRF ViewSets and serializers for the reporting REST API |
| [QQ_LL_Doc_ReportingApiPy.md](koalixcrm/reporting_api_py/QQ_LL_Doc_ReportingApiPy.md) | Per-app API router and URL configuration for reporting |

### `koalixcrm/shared`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Shared.md](koalixcrm/shared/QQ_LL_Doc_Shared.md) | WorkspaceScopedModel, WorkspaceAwareManager, BaseModelViewSet, and workspace-scoped permission mixins |

### `koalixcrm/subscriptions`

| Document | Description |
|---|---|
| [QQ_LL_Doc_Subscriptions.md](koalixcrm/subscriptions/QQ_LL_Doc_Subscriptions.md) | Subscription model, KoalixcrmPluginInterface implementation, and admin injection into contracts |

### Infrastructure Packages

| Package | Document | Description |
|---|---|---|
| `koalixcrm_microservices` | [QQ_LL_Doc_Microservices.md](koalixcrm_microservices/QQ_LL_Doc_Microservices.md) | Celery application bootstrap, task router, and SQS CommandEnvelope processing |
| `koalixcrm_mq_commands` | [QQ_LL_Doc_MQCommands.md](koalixcrm_mq_commands/QQ_LL_Doc_MQCommands.md) | Message command data classes: `CommandEnvelope`, `PDFExportCommand` |
| `koalixcrm_utils` | [QQ_LL_Doc_Utils.md](koalixcrm_utils/QQ_LL_Doc_Utils.md) | SQS and S3 utility functions, boto3 client factories |
| `projectsettings` | [QQ_LL_Doc_ProjectSettings.md](projectsettings/QQ_LL_Doc_ProjectSettings.md) | Django settings modules: base settings, development overlays, and production overlay |

## Imported Source Documents

| Document | Description |
|---|---|
| [QQ_IMPORT_docs-architecture-optional-apps.md](QQ_IMPORT_docs-architecture-optional-apps.md) | Human-authored description of the optional-app fork isolation pattern: which apps form the public surface and how optional peers are activated at runtime without import-time dependencies |
