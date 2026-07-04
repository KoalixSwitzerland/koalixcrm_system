# Mid-Level Documentation — Reporting App

## Introduction

### Purpose of the Package

The `reporting` app provides the project, time-tracking, and cost-reporting subsystem of koalixcrm. Its core responsibility is to let human resources record work against tasks, let project managers define cost agreements with resources, and let the system aggregate planned versus actual costs and effort into PDF snapshots and SVG charts for stakeholders.

The app sits between the commercial document lifecycle (handled by the `contracts` app) and the user identity layer (handled by `djangoUserExtension`). It owns the authoritative definition of a `Project` and its associated `Task`, `Work`, `ReportingPeriod`, `Agreement`, and `Estimation` records — concepts that other apps reference but do not own.

### Contents Overview

| Sub-package / module | Responsibility |
|---|---|
| `models/` | Domain model: `Project`, `Task`, `Work`, `ReportingPeriod` (and their status lookup tables), `Resource`, `HumanResource`, `ResourcePrice`, `Agreement`, `Estimation`, generic link tables |
| `views/` | DRF ViewSets for all domain entities, template views for the time-tracking UI (`work_report`), recovery views (`reporting_period_missing`, `user_is_not_human_resource`), and the `CreateTaskView` factory |
| `serializers/` | DRF serializers: flat read-write pairs (`*JSONSerializer`), the full project report snapshot (`ProjectReportSerializer`), and the human-resource work-report builder (`WorkReportBuilder` + `HumanResourceWorkReportSerializer`) |
| `services/` | `chart_storage` — builds the cost-overview SVG via matplotlib/pandas and uploads it to S3/MinIO |
| `admin/` | Django Admin registrations for all domain entities, admin actions for PDF export queuing (`create_report_pdf`, `create_work_report_pdf`), and the `ExtendedContractAdmin` extension |
| `signals/` | Empty module — no signal handlers are currently registered |
| `migrations/` | Django schema migrations (`0001_initial.py`, `0002_workspace_scoping.py`) |

### Target Audience

Software development engineers who need to integrate with, extend, or maintain the reporting subsystem of koalixcrm. Readers are expected to be familiar with Django, Django REST Framework, and the koalixcrm multi-tenant (`workspace`) model.

### Glossary

| Term/Acronym | Full Form | Description |
|---|---|---|
| CRM | Customer Relationship Management | The umbrella application (`koalixcrm`) this module belongs to. |
| DRF | Django REST Framework | The HTTP API library used for all JSON endpoints. |
| FK | Foreign Key | A database relationship field pointing to another table. |
| MTI | Multi-Table Inheritance | Django pattern where a child model extends a parent model via a shared primary key. |
| ORM | Object-Relational Mapper | Django's abstraction layer over the database. |
| RP / Reporting Period | — | A calendar-bounded window within a project used to gate work entry and cost confirmation. |
| WSM | WorkspaceScopedModel | The abstract base class (from `core`) that adds a `workspace` FK and a workspace-aware ORM manager to every tenant-scoped model. |
| SVG | Scalable Vector Graphics | Vector image format used for the project cost overview chart. |
| SQS | Amazon Simple Queue Service | AWS managed message queue used to enqueue PDF export jobs consumed by the Java pdf-export-service. |
| FOP | Apache Formatting Objects Processor | Java component that converts XSL-FO markup to PDF. |
| Presigned URL | S3 Presigned URL | A time-limited, self-authenticating URL for a single S3 object. |
| is_done | — | Boolean flag on status lookup models that marks a terminal lifecycle state. |
| is_agreed | — | Boolean flag on `AgreementStatus` that activates an agreement for cost matching. |
| is_obsolete | — | Boolean flag on `EstimationStatus` that marks an estimation as superseded. |

---

## Package Diagram

```mermaid
flowchart TD
    subgraph reporting["reporting app"]
        subgraph models_group["models/"]
            project["Project\nRoot aggregate; cost + effort aggregation"]
            task["Task\nWork unit; agreement-first cost pricing"]
            work["Work\nTime entry; start/stop or worked_hours"]
            rp["ReportingPeriod\nCalendar window; gates work entry"]
            resource["Resource / HumanResource\nBillable resource; linked to UserExtension"]
            agreement["Agreement\nNegotiated rate cap per resource+task"]
            estimation["Estimation\nRemaining effort estimate per period"]
            status_tables["Status lookup tables\nProjectStatus, TaskStatus,\nReportingPeriodStatus,\nAgreementStatus, EstimationStatus"]
            link_tables["Generic link tables\nGenericProjectLink, GenericTaskLink"]
        end

        subgraph views_group["views/"]
            viewsets["DRF ViewSets\nProjectViewSet, TaskViewSet,\nWorkViewSet, HumanResourceViewSet,\nReportingPeriodViewSet, etc."]
            ui_views["Template views\nwork_report, reporting_period_missing,\nuser_is_not_human_resource"]
            create_task["CreateTaskView\nFactory: contract → project+tasks"]
        end

        subgraph serializers_group["serializers/"]
            crud_ser["Flat serializers\nProjectJSONSerializer, TaskJSONSerializer,\nWorkJSONSerializer, etc."]
            report_ser["ProjectReportSerializer\nFull snapshot for Java PDF worker"]
            work_report_ser["WorkReportBuilder +\nHumanResourceWorkReportSerializer\nBucket aggregation by day/week/month"]
        end

        subgraph services_group["services/"]
            chart["chart_storage\nbuild SVG (matplotlib + pandas)\nupload to S3/MinIO"]
        end

        subgraph admin_group["admin/"]
            admin_views["Admin views\nProjectAdminView, TaskAdminView,\nWorkAdminView, ReportingPeriodAdmin,\nHumanResourceAdminView"]
            extended_contract["ExtendedContractAdmin\nAdds project link inline\nto contracts admin"]
        end
    end

    viewsets -->|reads/writes| models_group
    ui_views -->|reads/writes| models_group
    create_task -->|creates| models_group
    crud_ser -->|serializes| models_group
    report_ser -->|aggregates| models_group
    work_report_ser -->|aggregates| models_group
    report_ser -->|calls| chart
    admin_views -->|manages| models_group
    extended_contract -->|extends| admin_views
```

*Figure 1: Internal structure of the reporting app. Arrows indicate the direction of primary data access.*

Related low-level documentation:

- [Project, Task, Work and ReportingPeriod models](QQ_LL_Doc_Reporting_ProjectTaskModels.md)
- [Resource, Agreement, Estimation and HumanResource models](QQ_LL_Doc_Reporting_ResourceAgreementModels.md)
- [Views and Serializers](QQ_LL_Doc_Reporting_ViewsSerializers.md)
- [Services and Admin](QQ_LL_Doc_Reporting_ServicesAdmin.md)

---

## Interaction Diagrams

### Time-Tracking Entry (work_report view)

A logged-in user navigates to `/work_report/` to log or edit their daily work. On GET the view renders a formset pre-populated with existing `Work` records in a default date window. On POST with the save action, the view validates each `WorkEntry` form and persists the changes.

```mermaid
sequenceDiagram
    participant U as User (browser)
    participant WRV as work_report view
    participant RSF as RangeSelectionForm
    participant BWEF as BaseWorkEntryFormset
    participant WE as WorkEntry form
    participant W as Work model
    participant RP as ReportingPeriod

    U->>WRV: POST (save, date range, work entries)
    WRV->>RSF: validate range_selection_form
    RSF-->>WRV: cleaned date range
    WRV->>BWEF: load_formset (pre-check union range)
    BWEF->>W: query existing Work in expanded range
    W-->>BWEF: Work queryset
    BWEF-->>WRV: bound formset
    WRV->>WE: validate each form
    WE->>RP: get_reporting_period(work_date)
    RP-->>WE: reporting period or raises ReportingPeriodNotFound
    WE-->>WRV: cleaned_data or validation errors
    WRV->>WE: update_work(request)
    WE->>W: Work.save() or Work.delete()
    W-->>WRV: persisted
    WRV-->>U: re-render with updated formset
```

*Figure 2: Sequence for a successful work-entry save via the time-tracking UI.*

### Project Cost Report PDF Export

A project manager selects one or more projects in the Django Admin and triggers the "Create report PDF" action. The PDF is rendered asynchronously by the Java pdf-export-service.

```mermaid
sequenceDiagram
    participant PM as Project Manager (admin)
    participant PA as ProjectAdminView
    participant PEP as PDFExportProcess (core)
    participant PRS as ProjectReportSerializer
    participant CS as chart_storage service
    participant S3 as S3 / MinIO
    participant Java as Java PDF worker

    PM->>PA: admin action: create_report_pdf
    PA->>PEP: PDFExportProcess.objects.create(project, template)
    PEP-->>PA: process row created
    PA-->>PM: success message (N jobs queued)
    Java->>PRS: GET /projects/{id}/report-data/
    PRS->>CS: upload_project_cost_overview_svg(project)
    CS->>S3: put_object(SVG bytes)
    S3-->>CS: uploaded
    CS->>S3: generate_presigned_url
    S3-->>CS: presigned URL
    CS-->>PRS: presigned URL
    PRS-->>Java: JSON snapshot (costs, effort, chart URL)
    Java->>S3: GET presigned URL (download SVG)
    Java->>Java: FOP: render XSL → PDF
    Java->>S3: upload PDF
    Java->>PEP: write result URL back to process row
```

*Figure 3: Asynchronous PDF export sequence for a project cost report.*

### Cost Aggregation Data Flow

When `ProjectReportSerializer` calls `Project.effective_costs()` (and related methods), aggregation flows down the domain hierarchy.

```mermaid
flowchart LR
    PRS(["ProjectReportSerializer\nget_effective_costs_confirmed"])
    P["Project\neffective_costs()"]
    T["Task\neffective_costs()"]
    A["Agreement\nmatch_with_work()"]
    W["Work\neffort_seconds()"]
    RP2["ResourcePrice\nfallback price lookup"]

    PRS -->|calls| P
    P -->|iterates tasks| T
    T -->|iterates work records| W
    T -->|checks agreements| A
    A -->|matched: rate from agreement| T
    W -->|unmatched: looks up| RP2
    RP2 -->|default price per hour| T
    T -->|currency.round| P
    P -->|currency.round| PRS
```

*Figure 4: Aggregation path from serializer call to individual Work records and pricing sources.*

---

## Class Diagrams per Package

### Core domain model

```mermaid
classDiagram
    direction TB

    class Project {
        +project_name : CharField
        +project_status : FK ProjectStatus
        +default_currency : FK Currency
        +effective_costs(period, confirmed) Decimal
        +planned_costs(period, remaining) Decimal
        +is_reporting_allowed() bool
    }
    class Task {
        +title : CharField
        +project : FK Project
        +status : FK TaskStatus
        +effective_costs(period, confirmed) Decimal
        +planned_costs(period, remaining) Decimal
        +is_reporting_allowed() bool
    }
    class Work {
        +date : DateField
        +human_resource : FK HumanResource
        +task : FK Task
        +reporting_period : FK ReportingPeriod
        +effort_seconds() float
        +confirmed() bool
    }
    class ReportingPeriod {
        +begin : DateField
        +end : DateField
        +project : FK Project
        +status : FK ReportingPeriodStatus
        +get_reporting_period(project, date)$ RP
        +is_reporting_allowed() bool
    }

    Project "1" --> "*" Task
    Project "1" --> "*" ReportingPeriod
    Task "1" --> "*" Work
    Work --> ReportingPeriod
```

*Figure 5: Core domain model relationships. For full field and method lists see [QQ_LL_Doc_Reporting_ProjectTaskModels.md](QQ_LL_Doc_Reporting_ProjectTaskModels.md).*

### Resource and pricing model

```mermaid
classDiagram
    direction LR

    class HumanResource {
        +user : FK UserExtension
        +resource_contribution_project(from, to) list
    }
    class ResourcePrice {
        +resource : FK Resource
        +price : Decimal
        +currency : FK Currency
    }
    class Agreement {
        +task : FK Task
        +resource : FK Resource
        +amount : Decimal
        +date_from : DateField
        +date_until : DateField
        +match_with_work(work) bool
    }
    class Estimation {
        +task : FK Task
        +resource : FK Resource
        +reporting_period : FK ReportingPeriod
        +amount : Decimal
        +calculated_costs(bucket_start, bucket_end) Decimal
    }

    HumanResource --|> Resource
    ResourcePrice --> Resource
    Agreement --> Task
    Agreement --> Resource
    Estimation --> Task
    Estimation --> ReportingPeriod
```

*Figure 6: Resource, pricing and estimation relationships. For full detail see [QQ_LL_Doc_Reporting_ResourceAgreementModels.md](QQ_LL_Doc_Reporting_ResourceAgreementModels.md).*

### Views and serializers

```mermaid
classDiagram
    direction TB

    class ProjectViewSet {
        +report_data(request, pk) Response
    }
    class HumanResourceViewSet {
        +work_report_data(request, pk) Response
    }
    class ProjectReportSerializer {
        +get_project_cost_overview_url(obj) str
    }
    class WorkReportBuilder {
        +build() dict
    }
    class CreateTaskView {
        +create_project(admin, request, document, redirect) HttpResponseRedirect
    }

    ProjectViewSet --> ProjectReportSerializer : report_data action
    HumanResourceViewSet --> WorkReportBuilder : work_report_data action
    WorkReportBuilder --> HumanResourceWorkReportSerializer
    ProjectReportSerializer --> chart_storage : get_project_cost_overview_url
```

*Figure 7: Key ViewSet and serializer relationships. For full class hierarchies and serializer pairs see [QQ_LL_Doc_Reporting_ViewsSerializers.md](QQ_LL_Doc_Reporting_ViewsSerializers.md).*

---

## Design Patterns Used

**Aggregate root.** `Project` is the aggregate root for the project/task/work hierarchy. All cross-cutting cost and effort computations (`effective_costs`, `planned_costs`, `effective_effort`) flow downward: `Project` → `Task` → `Work` / `Estimation`. External callers interact with the aggregate through `Project` rather than directly querying child models.

**Status-flag–driven lifecycle.** Each entity (`Project`, `Task`, `ReportingPeriod`, `Agreement`, `Estimation`) has an associated status lookup model with one or two boolean flags (`is_done`, `is_agreed`, `is_obsolete`). These flags gate write operations (`is_reporting_allowed`, `confirmed`) and drive cost aggregation filters (confirmed vs. not-confirmed).

**Agreement-first cost allocation.** `Task.effective_costs()` matches each `Work` record against active `Agreement` entries first. Work not covered by any agreement falls back to the resource's default `ResourcePrice`. The `Agreement.match_with_work()` method encapsulates the matching logic, keeping the allocation algorithm in `Task.effective_costs()` free of agreement-specific conditions.

**Option / writable serializer pair.** Every domain entity with nested FK fields uses two DRF serializers: a read-only `Option*` variant for use as a nested field in other serializers, and a full `*JSONSerializer` with explicit `create`/`update` that resolves nested objects by `id`. This avoids DRF's writable nested serializer complexity while keeping read and write response shapes consistent.

**Builder pattern for report snapshots.** `WorkReportBuilder` separates the bucket-aggregation logic from serialization. `HumanResourceViewSet.work_report_data` instantiates the builder, calls `build()`, then passes the resulting plain dict through `HumanResourceWorkReportSerializer`. This makes the aggregation logic independently testable.

**Async job dispatch via PDFExportProcess.** Admin actions `create_report_pdf` and `create_work_report_pdf` do not synchronously produce PDFs. They create `PDFExportProcess` rows consumed by the Java pdf-export-service via SQS. The Java worker fetches the JSON snapshot from the `/report-data/` endpoint, calls `ProjectReportSerializer` (which in turn calls `chart_storage` to generate and upload the SVG), renders the XSL template via FOP, and posts a `CommercialDocumentMedia` record when done.

**Extended admin via unregister/re-register.** `ExtendedContractAdmin` adds a project-link inline to the `Contract` change page without modifying the `contracts` application itself. The reporting `admin/__init__.py` unregisters `Contract` from the contracts admin and re-registers it with the extended class.

---

## Dependencies on Other Modules

| Dependency | Direction | What is used |
|---|---|---|
| `koalixcrm.core` | Inbound | `WorkspaceScopedModel` (base class), `Currency.round()`, `Unit`, `PDFExportProcess`, `WorkspaceScopedModelAdmin`, `WorkspaceScopedViewSetMixin`, `BaseModelViewSet` |
| `koalixcrm.djangoUserExtension` | Inbound | `UserExtension` (FK on `HumanResource`, `ResourceManager`), `TemplateSet` (FK on `Project`) |
| `koalixcrm.products` | Inbound | `Price` — base class for `ResourcePrice` via MTI |
| `koalixcrm.contacts` | None (no direct dependency; contacts are reached through `contracts` → `Party`) | — |
| `koalixcrm.contracts` | Outbound (optional) | `Contract`, `CommercialDocument`, `CommercialDocumentPosition` — read by `CreateTaskView` when converting a commercial document into a project |
| `koalixcrm.shared` | Inbound | `BaseModelViewSet` |
| `koalixcrm_utils` | Inbound | `get_s3_client` — used by `chart_storage` for S3/MinIO access |

---

## External Dependencies

| Requirement | Version/Details | Notes |
|---|---|---|
| Django | ≥ 3.2 | ORM, admin, forms, signals |
| Django REST Framework | 3.x | All ViewSets and serializers |
| `drf-spectacular` | Any | `extend_schema_field` decorator on serializer method fields |
| `matplotlib` | ≥ 3.5 | Headless chart rendering (`Agg` backend); `chart_storage` |
| `pandas` | Compatible with installed matplotlib | DataFrame assembly for cost overview chart |
| `python-dateutil` | Any | `relativedelta` used in `WorkReportBuilder.build` for month arithmetic |
| PostgreSQL (via Django ORM) | Any version supported by Django | All model persistence |
| S3 / MinIO | AWS SDK compatible | Object storage for SVG chart upload and presigned URL generation |

---

## Testing

The LL documentation references no dedicated test files within the `reporting` app. Test coverage information is not available in the source files examined. Price and effort calculation logic in `Task.effective_costs()`, `Project.effective_costs()`, and `Estimation.calculated_costs()` constitutes the highest-value unit-test targets given the branching cost allocation algorithm.

---

## Appendix

### References

- [Project, Task, Work and ReportingPeriod models](QQ_LL_Doc_Reporting_ProjectTaskModels.md)
- [Resource, Agreement, Estimation and HumanResource models](QQ_LL_Doc_Reporting_ResourceAgreementModels.md)
- [Views and Serializers](QQ_LL_Doc_Reporting_ViewsSerializers.md)
- [Services and Admin](QQ_LL_Doc_Reporting_ServicesAdmin.md)
- Django ORM documentation: <https://docs.djangoproject.com/en/stable/topics/db/models/>
- Django REST Framework: <https://www.django-rest-framework.org/>
- matplotlib Agg backend: <https://matplotlib.org/stable/users/explain/figure/backends.html>

### List of Illustrations

| Figure | Title |
|---|---|
| Figure 1 | Reporting app internal structure |
| Figure 2 | Time-tracking entry sequence |
| Figure 3 | Project cost report PDF export sequence |
| Figure 4 | Cost aggregation data flow |
| Figure 5 | Project / Task / Work / ReportingPeriod relationships |
| Figure 6 | Resource / Agreement / Estimation relationships |
| Figure 7 | Key ViewSet and serializer relationships |
