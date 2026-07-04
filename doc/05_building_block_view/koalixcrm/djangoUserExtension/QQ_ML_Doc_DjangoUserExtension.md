# DjangoUserExtension — Mid-Level Documentation

## Introduction

### Purpose of the Package

The `koalixcrm.djangoUserExtension` Django application manages the configuration
that ties a Django user to the PDF export pipeline. It provides:

- A hierarchy of XSL-based document templates (`DocumentTemplate` and ten MTI
  subclasses) stored in Amazon S3.
- A `TemplateSet` aggregator that maps one template to each supported document kind.
- A `UserExtension` profile that links a Django `auth.User` to a `TemplateSet` and
  a default currency.
- REST endpoints that expose template metadata and serve file assets as short-lived
  presigned S3 URLs.
- GDPR-compliant contact assignment models (`UserAddressAssignment`,
  `UserPhoneAssignment`, `UserEmailAssignment`) that link users to contact-app
  party records.

### Contents Overview

| Sub-module | Contents |
|---|---|
| `models/document_template.py` | `DocumentTemplate` base class and ten MTI subclasses |
| `models/template_set.py` | `TemplateSet` aggregator and `OptionTemplateSet` admin class |
| `models/user_extension.py` | `UserExtension`, three contact-assignment models, `OptionUserExtension` admin class |
| `models/text_paragraph.py` | `TextParagraphInDocumentTemplate`, `InlineTextParagraph` admin inline |
| `serializers/` | Seven serializer modules covering templates, template sets, user extensions, and users |
| `views/` | `DocumentTemplateViewSet`, `UserExtensionViewSet`, and two redirect views |
| `admin/` | Admin registrations for all models |
| `exceptions.py` | Five custom exception classes |
| `const/purpose.py` | `PURPOSESADDRESSINUSEREXTENTION` constant |

### Target Audience

Software development engineers who need to use, modify, or extend the
`djangoUserExtension` application; in particular those integrating with the
PDF export pipeline or extending user profile data.

### Glossary

| Term/Acronym | Full Form | Description |
|---|---|---|
| MTI | Multi-Table Inheritance | Django pattern where each subclass has its own database table linked to the parent table by a one-to-one primary key. |
| FOP | Formatting Objects Processor | Apache FOP; a print formatter driven by XSL-FO stylesheets, used to generate PDFs. |
| XSL / XSL-FO | Extensible Stylesheet Language / Formatting Objects | Stylesheet language for describing print output; used here to drive FOP. |
| TemplateSet | — | An aggregate that groups one `DocumentTemplate` subtype per document kind, assigned to a `UserExtension`. |
| WorkspaceScopedModel | — | Base class that adds a `workspace` FK to support multi-tenant data isolation. |
| S3 | Amazon Simple Storage Service | Object-store backend where XSL, FOP config, and logo files are stored. |
| Presigned URL | — | Time-limited AWS S3 URL that grants unauthenticated download access to a single object. |
| MTI subtype | — | A concrete child class that inherits from `DocumentTemplate`; each subtype corresponds to one document kind. |
| PDF worker | — | The external service or process that calls the REST API to obtain template metadata and file assets before rendering a PDF. |

---

## Package Diagram

The diagram below shows the structural relationships between the main components
of the `djangoUserExtension` package and its interaction with the S3 storage
backend and Django's `auth.User`.

```mermaid
flowchart TD
    subgraph djangoUserExtension["djangoUserExtension Package"]
        UE["UserExtension\nLinks user to TemplateSet\nand default currency"]
        TS["TemplateSet\nAggregates 10 typed\nDocumentTemplate FKs"]
        DT["DocumentTemplate\nBase: xsl_file, fop_config_file, logo\n(10 MTI subclasses)"]
        TPL["TextParagraphInDocumentTemplate\nFreeform text blocks\nper template"]
        VIEWS["DocumentTemplateViewSet\nExposes metadata +\npresigned-URL redirects"]
        SER["DocumentTemplateJSONSerializer\nProduces *_href fields\nfor PDF worker"]
    end

    AUTHUSER["auth.User\nexternal"]
    S3["Amazon S3\nexternal storage"]
    CONTACTS["contacts.Party models\nexternal"]
    PDFWORKER["PDF Worker\nexternal consumer"]

    UE -->|default_template_set FK| TS
    UE -->|user FK| AUTHUSER
    UE -->|address/phone/email assignments| CONTACTS
    TS -->|10 typed FKs| DT
    DT -->|stores files in| S3
    TPL -->|document_template FK| DT
    VIEWS -->|uses| SER
    SER -->|serializes| DT
    PDFWORKER -->|GET metadata| VIEWS
    VIEWS -->|302 redirect| S3
```

Figure 1 — DjangoUserExtension package structure and external interactions

- [DjangoUserExtension LL Documentation](QQ_LL_Doc_DjangoUserExtension.md)

---

## Interaction Diagrams

### Template Resolution and PDF Export Handoff

This process describes the sequence from the moment a PDF worker requests
template metadata through to downloading the XSL-FO stylesheet from S3. The
entry point is `GET /api/document_templates/{id}/`.

The PDF worker first fetches the template metadata to obtain `*_href` URLs, then
follows those URLs to receive presigned S3 redirects, and finally downloads the
binary assets directly from S3.

```mermaid
sequenceDiagram
    participant PW as PDF Worker
    participant DTVS as DocumentTemplateViewSet
    participant SER as DocumentTemplateJSONSerializer
    participant DT as DocumentTemplate (DB)
    participant S3 as Amazon S3

    PW->>DTVS: GET /api/document_templates/{id}/
    DTVS->>DT: get_object()
    DT-->>DTVS: DocumentTemplate instance
    DTVS->>SER: serialize(instance, request)
    SER->>SER: _detail_url(instance, "xsl")
    SER-->>DTVS: {id, title, xsl_href, fop_config_href, logo_href}
    DTVS-->>PW: 200 JSON metadata

    PW->>DTVS: GET /api/document_templates/{id}/xsl/
    DTVS->>DT: get_object()
    DTVS->>S3: presigned_get_url_for_field(xsl_file)
    S3-->>DTVS: presigned URL (time-limited)
    DTVS-->>PW: 302 redirect to presigned URL
    PW->>S3: GET presigned URL
    S3-->>PW: XSL-FO binary file
```

Figure 2 — Template resolution and PDF export handoff sequence

### UserExtension Resolution for PDF Context

This process describes how a PDF export caller resolves user context — phone
number, e-mail, and template references — before rendering a document. The entry
point is `UserExtension.objects_to_serialize(object_to_create_pdf, reference_user)`.

```mermaid
sequenceDiagram
    participant CALLER as PDF Export Caller
    participant UE as UserExtension (static)
    participant TS as TemplateSet
    participant DT as DocumentTemplate

    CALLER->>UE: objects_to_serialize(doc, reference_user)
    UE->>UE: filter UserExtension by reference_user
    alt No UserExtension found
        UE-->>CALLER: raise UserExtensionMissing
    end
    UE->>UE: filter UserPhoneAssignment by user
    alt No phone found
        UE-->>CALLER: raise UserExtensionPhoneAddressMissing
    end
    UE->>UE: filter UserEmailAssignment by user
    alt No email found
        UE-->>CALLER: raise UserExtensionEmailAddressMissing
    end
    UE-->>CALLER: [User, UserExtension, PhoneNumber, PartyEmail]

    CALLER->>UE: get_user_extension(django_user)
    UE-->>CALLER: UserExtension instance
    CALLER->>TS: get_template_set("WorkReport")
    TS-->>CALLER: WorkReportTemplate instance
    CALLER->>DT: get_xsl_file()
    DT-->>CALLER: FieldFile (xsl_file)
```

Figure 3 — UserExtension resolution and template lookup for PDF context

---

## Class Diagrams per Package

### DocumentTemplate Hierarchy

```mermaid
classDiagram
    class DocumentTemplate {
        +CharField title
        +FileField xsl_file
        +FileField fop_config_file
        +FileField logo
        +get_fop_config_file() FieldFile
        +get_xsl_file() FieldFile
    }
    class InvoiceTemplate
    class QuotationTemplate
    class DespatchAdviceTemplate
    class PaymentReminderTemplate
    class PurchaseOrderTemplate
    class SalesOrderTemplate
    class ProfitLossStatementTemplate
    class BalanceSheetTemplate
    class MonthlyProjectSummaryTemplate
    class WorkReportTemplate

    DocumentTemplate <|-- InvoiceTemplate
    DocumentTemplate <|-- QuotationTemplate
    DocumentTemplate <|-- DespatchAdviceTemplate
    DocumentTemplate <|-- PaymentReminderTemplate
    DocumentTemplate <|-- PurchaseOrderTemplate
    DocumentTemplate <|-- SalesOrderTemplate
    DocumentTemplate <|-- ProfitLossStatementTemplate
    DocumentTemplate <|-- BalanceSheetTemplate
    DocumentTemplate <|-- MonthlyProjectSummaryTemplate
    DocumentTemplate <|-- WorkReportTemplate
```

Figure 4 — DocumentTemplate MTI class hierarchy

### TemplateSet and UserExtension

```mermaid
classDiagram
    class TemplateSet {
        +BigAutoField id
        +CharField title
        +ForeignKey invoice_template
        +ForeignKey quotation_template
        +ForeignKey despatch_advice_template
        +ForeignKey payment_reminder_template
        +ForeignKey sales_order_template
        +ForeignKey purchase_order_template
        +ForeignKey profit_loss_statement_template
        +ForeignKey balance_sheet_statement_template
        +ForeignKey monthly_project_summary_template
        +ForeignKey work_report_template
        +get_template_set(required_template_set) DocumentTemplate
    }

    class UserExtension {
        +BigAutoField id
        +ForeignKey user
        +ForeignKey default_template_set
        +ForeignKey default_currency
        +objects_to_serialize(object_to_create_pdf, reference_user) list
        +get_user_extension(django_user) UserExtension
        +get_template_set(template_set) DocumentTemplate
        +get_fop_config_file(template_set) FieldFile
        +get_xsl_file(template_set) FieldFile
    }

    class UserAddressAssignment {
        +ForeignKey user
        +ForeignKey address
        +CharField purpose
        +BooleanField is_primary
        +DateField valid_from
        +DateField valid_to
    }

    class UserPhoneAssignment {
        +ForeignKey user
        +ForeignKey phone_number
        +CharField purpose
        +BooleanField is_primary
    }

    class UserEmailAssignment {
        +ForeignKey user
        +ForeignKey email
        +CharField purpose
        +BooleanField is_primary
    }

    UserExtension --> TemplateSet : default_template_set
    TemplateSet --> DocumentTemplate : 10 typed FKs
    UserExtension "1" --> "*" UserAddressAssignment : user
    UserExtension "1" --> "*" UserPhoneAssignment : user
    UserExtension "1" --> "*" UserEmailAssignment : user
```

Figure 5 — TemplateSet and UserExtension class relationships

---

## Design Patterns Used

### Multi-Table Inheritance (MTI) for Templates

`DocumentTemplate` uses Django's MTI so that `TemplateSet` can hold ten
type-safe FK references — one per document kind — while all templates share a
single base table (`djangouserextension_documenttemplate`) for the common
`title`, `xsl_file`, `fop_config_file`, and `logo` fields. Each subclass (e.g.
`InvoiceTemplate`) has its own table containing only the Django-generated
one-to-one link back to the base table. None of the subclasses add domain fields;
their purpose is type identity.

### Aggregator / Registry Pattern for TemplateSet

`TemplateSet.get_template_set(required_template_set: str)` uses a static
string-to-attribute mapping dictionary to resolve which of the ten FK fields to
return. This avoids a chain of `if/elif` branches and allows the mapping to be
inspected as data. Callers pass a string key (e.g. `"Invoice"`, `"WorkReport"`)
and receive the corresponding `DocumentTemplate` subtype instance or an exception.

### S3 Presigned URL Pattern

Template binary assets (`xsl_file`, `fop_config_file`, `logo`) are never
included in API responses. Instead, `DocumentTemplateJSONSerializer` emits
`*_href` fields that point to detail action URLs on `DocumentTemplateViewSet`.
When a client follows one of those URLs, the viewset calls
`presigned_get_url_for_field` (from `koalixcrm_utils`) and returns a `302`
redirect to a time-limited S3 URL. The binary transfer occurs directly between
the client and S3, keeping the application server out of the data path.

### Delegation for File Accessor Methods

`UserExtension.get_fop_config_file` and `get_xsl_file` are thin delegation
methods: they call `get_template_set` to resolve the `DocumentTemplate`, then
forward the call to the template's own file accessor. This separates the concern
of template resolution (which document kind to use for this user) from the concern
of file retrieval (whether the file field is populated).

---

## External Dependencies

| Requirement | Version/Details | Notes/Assumptions |
|---|---|---|
| Django | >= 3.2 (BigAutoField default) | Required for all ORM models, admin, and auth integration |
| djangorestframework | Not pinned in this module | Used for all serializers and viewsets |
| `koalixcrm_utils.s3_storage.TemplateFileStorage` | Internal package | S3-backed `FileField` storage; applied to all three file fields on `DocumentTemplate` |
| `koalixcrm_utils.presigned_urls.presigned_get_url_for_field` | Internal package | Generates time-limited S3 presigned download URLs; called in `DocumentTemplateViewSet._redirect_to_field` |
| `koalixcrm.core.models.workspace_scoped.WorkspaceScopedModel` | Internal | Base class providing multi-tenant workspace FK for all models in this package |
| `koalixcrm.contacts` — `Address`, `PhoneNumber`, `PartyEmail` | Internal app | Target models for the three contact-assignment FK fields on `UserExtension` |
| `django.contrib.auth.User` | Django built-in | Referenced by `UserExtension.user` FK and all three contact-assignment models |
| `factory_boy` | Test dependency | Used in `tests/factories/djangoUserExtension/` to build model instances in tests |

---

## Testing

No dedicated unit-test module exists for `djangoUserExtension` in the
`tests/` tree. Test coverage is provided indirectly through:

- **Factory classes** in `tests/factories/djangoUserExtension/`:
  `factory_document_template.py`, `factory_template_set.py`, and
  `factory_user_extension.py` supply `DocumentTemplate` subtype instances,
  `TemplateSet`, and `UserExtension` objects to integration and end-to-end tests
  in other modules (e.g. `legacy_crm`, `reporting_api_py`, `e2e`).
- **End-to-end tests** in `tests/e2e/` use `StandardUserExtensionFactory` as
  a prerequisite fixture, exercising the full resolution path (user →
  UserExtension → TemplateSet → DocumentTemplate) indirectly.
- **Integration tests** in `tests/core_api_py/test_pdf_service_endpoints.py`
  and `tests/reporting_api_py/` exercise the REST endpoints that depend on
  `UserExtension` and `DocumentTemplate` data.

There are no unit tests that exercise `TemplateSet.get_template_set`,
`UserExtension.objects_to_serialize`, or `DocumentTemplateViewSet._redirect_to_field`
in isolation.

---

## Appendix

### References

- [DjangoUserExtension LL Documentation](QQ_LL_Doc_DjangoUserExtension.md)

### List of Illustrations

| Figure | Title |
|---|---|
| Figure 1 | DjangoUserExtension package structure and external interactions |
| Figure 2 | Template resolution and PDF export handoff sequence |
| Figure 3 | UserExtension resolution and template lookup for PDF context |
| Figure 4 | DocumentTemplate MTI class hierarchy |
| Figure 5 | TemplateSet and UserExtension class relationships |

### Additional Resources and Further Reading

- [Django Multi-Table Inheritance documentation](https://docs.djangoproject.com/en/stable/topics/db/models/#multi-table-inheritance)
- [Amazon S3 Presigned URLs — AWS documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Apache FOP documentation](https://xmlgraphics.apache.org/fop/)
