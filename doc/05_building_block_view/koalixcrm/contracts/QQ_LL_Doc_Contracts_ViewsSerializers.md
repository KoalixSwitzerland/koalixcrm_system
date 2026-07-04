# Low-Level Documentation: Contracts — Views and Serializers

## Introduction

### Scope

This document covers all Django REST Framework ViewSets, mixins, and serializer classes defined in:

| Directory | Files |
|-----------|-------|
| `koalixcrm/contracts/views/` | `contract_view_set.py`, `invoice_view_set.py`, `quotation_view_set.py`, `sales_order_view_set.py`, `purchase_order_view_set.py`, `credit_note_view_set.py`, `despatch_advice_view_set.py`, `payment_reminder_view_set.py`, `commercial_document_media_view_set.py`, `commercial_document_position_view_set.py`, `nested_detail_mixin.py`, `newdocument.py` |
| `koalixcrm/contracts/serializers/` | `contract_serializer.py`, `commercial_document_serializer.py`, `invoice_serializer.py`, `quotation_serializer.py`, `sales_order_serializer.py`, `purchase_order_serializer.py`, `credit_note_serializer.py`, `despatch_advice_serializer.py`, `payment_reminder_serializer.py`, `commercial_document_position_serializer.py`, `commercial_document_media_serializer.py`, `nested_commercial_document.py` |

### Target Audience

The primary audience for this documentation is the software development engineer responsible for maintaining the REST API layer or developing integrations with the contracts module.

### Glossary

| Term/Acronym | Full Form | Description |
|--------------|-----------|-------------|
| DRF | Django REST Framework | The HTTP API framework used for ViewSets and serializers |
| ViewSet | — | DRF class that bundles list/create/retrieve/update/destroy actions |
| CRUD serializer | — | Flat serializer used for normal read/write API operations |
| Nested serializer | — | Deep, read-only serializer that assembles the full document snapshot for the PDF worker |
| PDF worker | — | External Java service that consumes the nested endpoint and renders PDFs |
| UBL | Universal Business Language | OASIS XML standard; the nested shape mirrors the UBL document structure |
| Party | — | Abstract contact entity; resolved to `Organization` or `PartyContact` at runtime |
| Tax summary | — | Aggregation of position tax amounts grouped by tax rate |
| Workspace | — | Tenant-level scoping entity; all queryset filters use `workspace` |

---

## Detailed Component

### ViewSets

#### ContractViewSet

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class ContractViewSet {
            +QuerySet queryset
            +ContractJSONSerializer serializer_class
            +get_queryset() QuerySet
            +perform_create(serializer) None
        }
    }

    class BaseModelViewSet:::external {
        <<external: shared>>
    }
    class Contract:::external {
        <<external: contracts.models>>
    }
    class ContractJSONSerializer:::external {
        <<external: contracts.serializers>>
    }
    class Workspace:::external {
        <<external: core>>
    }

    ContractViewSet --|> BaseModelViewSet
    ContractViewSet --> Contract : queryset
    ContractViewSet --> ContractJSONSerializer : serializer_class
    ContractViewSet --> Workspace : perform_create

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 1 — ContractViewSet

`ContractViewSet` exposes the standard CRUD operations for `Contract`. It inherits from `BaseModelViewSet` (from `koalixcrm.shared`), which provides DRF's `ModelViewSet` behaviour with any project-wide permission and authentication configuration.

##### `get_queryset()`

Signature: `get_queryset(self) -> QuerySet[Contract]`

Returns all contracts for superusers, filters by the request's `active_workspace` for regular users, and returns an empty queryset when no workspace is active.

```mermaid
flowchart TD
    A([Start]) --> B{user.is_superuser?}
    B -->|Yes| R1([Return all contracts])
    B -->|No| C{active_workspace set?}
    C -->|Yes| R2([Return filtered by workspace])
    C -->|No| R3([Return empty queryset])
```

Figure 2 — ContractViewSet.get_queryset flow

##### `perform_create(serializer)`

Signature: `perform_create(self, serializer: BaseSerializer) -> None`

Resolves the workspace for the new object. For superusers without an active workspace, it falls back to creating or reusing a `Default Workspace` record. Then saves with `workspace=active`.

---

#### Document ViewSets (InvoiceViewSet, QuotationViewSet, PurchaseOrderViewSet, CreditNoteViewSet, DespatchAdviceViewSet, PaymentReminderViewSet)

All six document-type ViewSets share the same structural pattern and are documented together.

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class InvoiceViewSet {
            +InvoiceNestedSerializer nested_serializer_class
            +get_queryset() QuerySet
            +perform_create(serializer) None
        }
        class QuotationViewSet {
            +QuotationNestedSerializer nested_serializer_class
        }
        class PurchaseOrderViewSet {
            +PurchaseOrderNestedSerializer nested_serializer_class
        }
        class CreditNoteViewSet {
            +CreditNoteNestedSerializer nested_serializer_class
        }
        class DespatchAdviceViewSet {
            +DespatchAdviceNestedSerializer nested_serializer_class
        }
        class PaymentReminderViewSet {
            +PaymentReminderNestedSerializer nested_serializer_class
        }
    }

    class NestedDetailMixin:::external {
        <<external: contracts.views>>
    }
    class BaseModelViewSet:::external {
        <<external: shared>>
    }

    InvoiceViewSet --|> NestedDetailMixin
    InvoiceViewSet --|> BaseModelViewSet
    QuotationViewSet --|> NestedDetailMixin
    QuotationViewSet --|> BaseModelViewSet
    PurchaseOrderViewSet --|> NestedDetailMixin
    PurchaseOrderViewSet --|> BaseModelViewSet
    CreditNoteViewSet --|> NestedDetailMixin
    CreditNoteViewSet --|> BaseModelViewSet
    DespatchAdviceViewSet --|> NestedDetailMixin
    DespatchAdviceViewSet --|> BaseModelViewSet
    PaymentReminderViewSet --|> NestedDetailMixin
    PaymentReminderViewSet --|> BaseModelViewSet

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 3 — Document ViewSet inheritance

Each ViewSet:

- Sets `queryset` to `DocumentType.objects.all()`
- Sets `serializer_class` to the flat `DocumentTypeJSONSerializer`
- Sets `nested_serializer_class` to the deep `DocumentTypeNestedSerializer`
- Overrides `get_queryset()` with the same workspace-scoping logic as `ContractViewSet`
- Overrides `perform_create()` with the same workspace-assignment logic

`SalesOrderViewSet` follows the same pattern but does not mix in `NestedDetailMixin` — no nested endpoint is exposed for sales orders.

---

#### NestedDetailMixin

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class NestedDetailMixin {
            +nested_serializer_class: type | None
            +nested(request, pk, **kwargs) Response
        }
    }

    class DRFAction:::external {
        <<external: rest_framework>>
    }

    NestedDetailMixin --> DRFAction : @action decorator

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 4 — NestedDetailMixin

`NestedDetailMixin` adds a single read-only detail action at `GET /{model}/{id}/nested/`. It resolves the object via `self.get_object()` (which applies the ViewSet's normal permission and queryset pipeline) and serializes it through `nested_serializer_class`.

Subclasses must set `nested_serializer_class`; failing to do so raises `NotImplementedError` at call time, not at class-definition time.

The mixin deliberately keeps normal write operations flowing through the simpler flat serializers, so the nested endpoint is purely for reading — intended for the PDF worker to obtain a complete document snapshot in one request.

##### `nested(request, pk, **kwargs)`

```mermaid
flowchart TD
    A([GET request]) --> B{nested_serializer_class set?}
    B -->|No| ERR[Raise NotImplementedError]
    B -->|Yes| C["get_object() — applies queryset and permissions"]
    C --> D[Serialize with nested_serializer_class]
    D --> E([Return Response])
```

Figure 5 — NestedDetailMixin.nested flow

---

#### CommercialDocumentMediaViewSet

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class CommercialDocumentMediaViewSet {
            +queryset
            +CommercialDocumentMediaJSONSerializer serializer_class
            +http_method_names = [get, post, head, options]
            +get_queryset() QuerySet
            +perform_create(serializer) None
        }
    }

    class CreateModelMixin:::external {
        <<external: rest_framework>>
    }
    class RetrieveModelMixin:::external {
        <<external: rest_framework>>
    }
    class ListModelMixin:::external {
        <<external: rest_framework>>
    }
    class GenericViewSet:::external {
        <<external: rest_framework>>
    }

    CommercialDocumentMediaViewSet --|> CreateModelMixin
    CommercialDocumentMediaViewSet --|> RetrieveModelMixin
    CommercialDocumentMediaViewSet --|> ListModelMixin
    CommercialDocumentMediaViewSet --|> GenericViewSet

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 6 — CommercialDocumentMediaViewSet

`CommercialDocumentMediaViewSet` deliberately omits update and delete mixins. Media records are append-only: the PDF worker POSTs a row after a successful S3 upload; no client may modify or remove it. The `http_method_names` list enforces this at the ViewSet level. `permission_classes` includes both `IsAuthenticated` and `ModelPermissionsWithListView` (a project-level custom permission class).

---

#### CommercialDocumentPositionViewSet

Standard `BaseModelViewSet` for line items. `get_queryset()` and `perform_create()` follow the same workspace-scoping pattern as the other ViewSets.

---

#### CreateNewDocumentView

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class CreateNewDocumentView {
            +create_new_document(calling_model_admin, request, calling_model, requested_document_type, redirect_to) HttpResponseRedirect$
        }
    }

    class CommercialDocument:::external {
        <<external: contracts.models>>
    }
    class Contract:::external {
        <<external: contracts.models>>
    }

    CreateNewDocumentView --> CommercialDocument : calling_model (may be)
    CreateNewDocumentView --> Contract : calling_model (may be)

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 7 — CreateNewDocumentView

`CreateNewDocumentView` is a utility class (not a DRF ViewSet) that encapsulates the admin-action document-creation flow. Its sole static method `create_new_document` instantiates a `requested_document_type`, calls `create_from_reference(calling_model)`, and returns a redirect to the new document's admin change view. On `TemplateSetMissingInContract` or `TemplateMissingInTemplateSet` it redirects to the relevant admin page and messages the user with an error instead of raising an HTTP 404.

##### `create_new_document(calling_model_admin, request, calling_model, requested_document_type, redirect_to)` (static)

```mermaid
flowchart TD
    A([Admin action]) --> B[Instantiate new document of requested type]
    B --> C["new_document.create_from_reference(calling_model)"]
    C --> D{Success?}
    D -->|Yes| E[message_user: created]
    E --> F([Redirect to new document change page])
    D -->|TemplateSetMissingInContract| G[Redirect to contract admin]
    G --> H[message_user: ERROR Missing Templateset]
    D -->|TemplateMissingInTemplateSet| I[Redirect to template set admin]
    I --> J[message_user: ERROR Missing template]
    D -->|Other exception| K[Raise Http404]
```

Figure 8 — create_new_document flow

---

### Serializers

#### Flat (CRUD) Serializers

All flat serializers follow the same minimal pattern: a `ModelSerializer` subclass with a `Meta` inner class. Most declare `fields = '__all__'` with `read_only_fields = ('workspace',)` where applicable.

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class ContractJSONSerializer {
            +Meta model = Contract
            +Meta fields = id, staff, description, buyer_party, supplier_party, default_currency, default_template_set, last_modified_by
        }
        class CommercialDocumentJSONSerializer {
            +Meta model = CommercialDocument
            +Meta fields = __all__
        }
        class InvoiceJSONSerializer {
            +Meta model = Invoice
            +Meta fields = __all__
            +Meta read_only_fields = workspace
        }
        class QuotationJSONSerializer {
            +Meta model = Quotation
            +Meta fields = __all__
            +Meta read_only_fields = workspace
        }
        class SalesOrderJSONSerializer {
            +Meta model = SalesOrder
            +Meta fields = __all__
        }
        class PurchaseOrderJSONSerializer {
            +Meta model = PurchaseOrder
            +Meta fields = __all__
        }
        class CreditNoteJSONSerializer {
            +Meta model = CreditNote
            +Meta fields = __all__
        }
        class DespatchAdviceJSONSerializer {
            +Meta model = DespatchAdvice
            +Meta fields = __all__
        }
        class PaymentReminderJSONSerializer {
            +Meta model = PaymentReminder
            +Meta fields = __all__
        }
        class CommercialDocumentPositionJSONSerializer {
            +Meta model = CommercialDocumentPosition
            +Meta fields = __all__
        }
        class CommercialDocumentMediaJSONSerializer {
            +Meta model = CommercialDocumentMedia
            +Meta fields = id, commercial_document, pdf_export_process, s3_url, s3_key, status, media_type, created_by, created_at, last_updated_at
            +Meta read_only_fields = id, created_at, last_updated_at
        }
    }

    class ModelSerializer:::external {
        <<external: rest_framework>>
    }

    ContractJSONSerializer --|> ModelSerializer
    CommercialDocumentJSONSerializer --|> ModelSerializer
    InvoiceJSONSerializer --|> ModelSerializer
    QuotationJSONSerializer --|> ModelSerializer
    SalesOrderJSONSerializer --|> ModelSerializer
    PurchaseOrderJSONSerializer --|> ModelSerializer
    CreditNoteJSONSerializer --|> ModelSerializer
    DespatchAdviceJSONSerializer --|> ModelSerializer
    PaymentReminderJSONSerializer --|> ModelSerializer
    CommercialDocumentPositionJSONSerializer --|> ModelSerializer
    CommercialDocumentMediaJSONSerializer --|> ModelSerializer

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 9 — Flat serializer hierarchy

`ContractJSONSerializer` is the only serializer that enumerates fields explicitly rather than using `'__all__'`, excluding auto-timestamps and workspace from the API surface.

`CommercialDocumentMediaJSONSerializer` declares an explicit field list and marks `id`, `created_at`, and `last_updated_at` as read-only, preventing clients from overriding system-assigned values.

---

#### Nested Serializers (`nested_commercial_document.py`)

The nested serializer module assembles the full document snapshot that the Java PDF worker reads via `GET /{document-type}/{id}/nested/`. It contains several collaborating classes.

```mermaid
classDiagram
    direction LR

    namespace contracts {
        class _BaseCommercialDocumentNestedSerializer {
            +type: SerializerMethodField
            +party: PartyNestedSerializer
            +currency: CurrencyJSONSerializer
            +items: SerializerMethodField
            +tax_summary: SerializerMethodField
            +user_extension: SerializerMethodField
            +get_type(obj) str
            +get_items(obj) list
            +get_tax_summary(obj) list
            +get_user_extension(obj) int | None
            -_positions(obj) list
        }
        class InvoiceNestedSerializer {
            +Meta fields += payable_until, payment_bank_reference, status
        }
        class QuotationNestedSerializer {
            +Meta fields += valid_until, status
        }
        class DespatchAdviceNestedSerializer {
            +Meta fields += tracking_reference, status
        }
        class PurchaseOrderNestedSerializer {
            +Meta fields += status
        }
        class PaymentReminderNestedSerializer {
            +Meta fields += payable_until, payment_bank_reference, iteration_number, status
        }
        class CreditNoteNestedSerializer {
            +Meta fields += corrects_invoice, issue_date, reason, status
        }
        class PartyNestedSerializer {
            +get_type(obj) str
            +get_organization(obj) dict | None
            +get_contact(obj) dict | None
            +get_postal_addresses(obj) list
            +get_phone_numbers(obj) list
            +get_email_addresses(obj) list
        }
        class PositionNestedSerializer {
            +product_type: ProductTypeNestedSerializer
            +unit: OptionUnitJSONSerializer
        }
        class ProductTypeNestedSerializer {
            +get_tax_rate(obj) str | None
        }
        class NestedAddressSerializer {
            +get_street(obj) Any
            +get_town(obj) Any
            +get_country(obj) Any
        }
    }

    class ModelSerializer:::external {
        <<external: rest_framework>>
    }
    class Serializer:::external {
        <<external: rest_framework>>
    }

    _BaseCommercialDocumentNestedSerializer --|> ModelSerializer
    InvoiceNestedSerializer --|> _BaseCommercialDocumentNestedSerializer
    QuotationNestedSerializer --|> _BaseCommercialDocumentNestedSerializer
    DespatchAdviceNestedSerializer --|> _BaseCommercialDocumentNestedSerializer
    PurchaseOrderNestedSerializer --|> _BaseCommercialDocumentNestedSerializer
    PaymentReminderNestedSerializer --|> _BaseCommercialDocumentNestedSerializer
    CreditNoteNestedSerializer --|> _BaseCommercialDocumentNestedSerializer
    PartyNestedSerializer --|> Serializer
    PositionNestedSerializer --|> ModelSerializer
    ProductTypeNestedSerializer --|> Serializer
    NestedAddressSerializer --|> Serializer
    _BaseCommercialDocumentNestedSerializer --> PartyNestedSerializer : party
    _BaseCommercialDocumentNestedSerializer --> PositionNestedSerializer : items
    PositionNestedSerializer --> ProductTypeNestedSerializer : product_type

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

Figure 10 — Nested serializer hierarchy

##### `_BaseCommercialDocumentNestedSerializer`

The private base class provides the common fields for all document types. It is not registered to a model directly; subclasses set their `Meta.model` to the concrete Django model. The `fields` tuple is inherited and extended by each subclass via Python class-level `Meta` inheritance.

**`get_type(obj)`** returns the concrete Python class name (e.g. `"Invoice"`) so the PDF worker can select the correct XmlBuilder without inspecting the URL.

**`_positions(obj)`** (private) fetches `CommercialDocumentPosition` rows for the document ordered by `position_number`. It is called by both `get_items` and `get_tax_summary` to avoid code duplication, but this means two separate DB queries are issued per nested request — one for items, one for the tax summary.

**`get_items(obj)`** passes the position list through `PositionNestedSerializer` and returns the serialized list.

**`get_tax_summary(obj)`** delegates to the module-level `_compute_tax_summary` function.

**`get_user_extension(obj)`** looks up the `UserExtension` linked to `obj.staff_id` and returns its `id` (or `None`). The PDF worker fetches the full user-extension data from a separate endpoint using this id.

##### `PartyNestedSerializer`

Resolves a `Party` to either an `Organization` or a `PartyContact` by querying the MTI satellite tables directly. Returns a discriminated structure with a `type` field (`"organization"` / `"contact"` / `"party"`) so the PDF worker can pick the appropriate name rendering path.

Address, phone, and email data are sourced from the assignment tables (`AddressAssignment`, `PhoneAssignment`, `EmailAssignment`) rather than from embedded fields, reflecting the loose-link contact model.

```mermaid
flowchart TD
    A([Serialize Party]) --> B{Organization row exists?}
    B -->|Yes| C[type = organization]
    C --> D[Fill organization dict: legal_name, legal_form, etc.]
    D --> G[Serialize addresses, phones, emails from assignments]
    G --> Z([Return])
    B -->|No| E{PartyContact row exists?}
    E -->|Yes| F[type = contact]
    F --> H[Fill contact dict: prefix, given_name, family_name]
    H --> G
    E -->|No| I[type = party]
    I --> G
```

Figure 11 — PartyNestedSerializer resolution flow

##### `_compute_tax_summary(positions)` (module-level function)

Signature: `_compute_tax_summary(positions: list[CommercialDocumentPosition]) -> list[dict[str, str]]`

Aggregates positions by tax rate into an `OrderedDict`. For each position it resolves the rate key from (1) `product_type.tax.get_tax_rate()`, (2) `position_tax_rate`, or (3) the literal string `"unknown"`. Accumulates `taxable_amount` and `tax_amount` per bucket, then converts all `Decimal` values to strings for JSON serialization.

```mermaid
flowchart TD
    A([Start]) --> B[buckets = OrderedDict]
    B --> C[For each position]
    C --> D{product_type.tax set?}
    D -->|Yes| E["rate_key = str(tax.get_tax_rate())"]
    D -->|No| F{position_tax_rate set?}
    F -->|Yes| G["rate_key = str(position_tax_rate)"]
    F -->|No| H["rate_key = 'unknown'"]
    E --> I[Add price and tax amounts to bucket]
    G --> I
    H --> I
    I --> J{More positions?}
    J -->|Yes| C
    J -->|No| K[Convert Decimal values to str]
    K --> L([Return list of buckets])
```

Figure 12 — _compute_tax_summary flow

---

## In-Memory State

None of the serializer or ViewSet classes maintain in-memory state between requests. `_compute_tax_summary` operates on a list passed in per call.

The `_positions()` private method on `_BaseCommercialDocumentNestedSerializer` is called twice per nested request (once for `get_items`, once for `get_tax_summary`). The results are not cached between the two calls.

---

## Access to External Interfaces

| Interface | Type of Call | Expected Duration | Notes |
|-----------|--------------|-------------------|-------|
| Django ORM — position query | Blocking DB read | ~10–30 ms | Called twice per nested request (items + tax summary) |
| Django ORM — Organization/PartyContact MTI lookup | Blocking DB read | ~5–20 ms | Two separate queries per nested serialization |
| Django ORM — AddressAssignment / PhoneAssignment / EmailAssignment | Blocking DB read | ~5–20 ms per assignment type | Three queries inside `PartyNestedSerializer` |
| Django ORM — UserExtension | Blocking DB read | ~5–10 ms | One query in `get_user_extension` |
| `Workspace.objects.get_or_create` | Blocking DB read/write | ~5–20 ms | Called in `perform_create` for superusers without active workspace |

The nested endpoint issues a relatively high number of DB queries per request (positions ×2, party type check ×2, assignments ×3, user extension ×1 = at minimum 9 queries). No `select_related` or `prefetch_related` is applied at the ViewSet level. Information not available: whether the project uses database-level query logging or N+1 detection in CI.

---

## Security

### Assets

| Asset | Description | Security Measure | Assessment of Criticality |
|-------|-------------|------------------|---------------------------|
| Workspace tenancy | All querysets filter by `active_workspace` | Enforced in every `get_queryset` override; superusers bypass | Uncritical for superusers by design; non-superusers cannot access other workspaces |
| Authentication | DRF permission classes on each ViewSet | `IsAuthenticated` + `ModelPermissionsWithListView` on media ViewSet; inherited from `BaseModelViewSet` on others | Information not available: the exact permission classes on `BaseModelViewSet` |

---

## Design Patterns Used

### Mixin Composition

`NestedDetailMixin` and `BaseModelViewSet` are composed via Python multiple inheritance. Each ViewSet that needs the nested endpoint simply adds `NestedDetailMixin` to its base class list.

### Discriminated Serializer

`PartyNestedSerializer` uses a `type` discriminator field to tell the consumer which branch of the party data is populated, avoiding a union type.

### Separate Read/Write Serializers

The flat `*JSONSerializer` classes handle write operations; the `*NestedSerializer` classes handle the deep read for the PDF worker. This separation ensures that write validation stays simple and the nested shape can evolve independently of the CRUD API.

### Append-Only ViewSet

`CommercialDocumentMediaViewSet` restricts itself to create/list/retrieve by only mixing in the three relevant DRF mixins and restricting `http_method_names`.

---

## External Dependencies

| Requirement | Version/Details | Notes |
|-------------|-----------------|-------|
| Django REST Framework | ≥ 3.14 (inferred) | All ViewSets and serializers depend on `rest_framework` |
| `koalixcrm.shared.base_model_view_set` | Internal | Provides `BaseModelViewSet` |
| `koalixcrm.shared.permissions` | Internal | Provides `ModelPermissionsWithListView` |
| `koalixcrm.core.serializers` | Internal | Provides `CurrencyJSONSerializer`, `OptionTaxJSONSerializer`, `OptionUnitJSONSerializer` |
| `koalixcrm.contacts.models` | Internal | Provides `Organization`, `PartyContact`, `AddressAssignment`, `PhoneAssignment`, `EmailAssignment` |
| `koalixcrm.djangoUserExtension.models` | Internal | Provides `UserExtension` |

---

## Appendix

### References

- Source files: `koalixcrm/contracts/views/`, `koalixcrm/contracts/serializers/`
- Related documentation: [`QQ_LL_Doc_Contracts_Models.md`](./QQ_LL_Doc_Contracts_Models.md), [`QQ_LL_Doc_Contracts_Admin.md`](./QQ_LL_Doc_Contracts_Admin.md)

### List of Illustrations

| Figure | Title |
|--------|-------|
| Figure 1 | ContractViewSet |
| Figure 2 | ContractViewSet.get_queryset flow |
| Figure 3 | Document ViewSet inheritance |
| Figure 4 | NestedDetailMixin |
| Figure 5 | NestedDetailMixin.nested flow |
| Figure 6 | CommercialDocumentMediaViewSet |
| Figure 7 | CreateNewDocumentView |
| Figure 8 | create_new_document flow |
| Figure 9 | Flat serializer hierarchy |
| Figure 10 | Nested serializer hierarchy |
| Figure 11 | PartyNestedSerializer resolution flow |
| Figure 12 | `_compute_tax_summary` flow |
