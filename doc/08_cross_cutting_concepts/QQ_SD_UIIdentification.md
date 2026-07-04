# UI Identification

## Summary

| Property | Value |
|----------|-------|
| UI Framework | Django Admin with django-grappelli 3.0.10 |
| Base Framework | Django 5.2.13 |
| Application Type | Server-Side Rendered (SSR) — Django template engine |
| Target Platform | Web browser |
| SPA / Separate Frontend | None detected |
| JavaScript Frameworks | None (jQuery provided by Grappelli; no React, Vue, Angular, or similar) |
| Template Engine | Django Template Language (DTL) |
| API Layer | Django REST Framework 3.16.0 with drf-spectacular 0.27.2 (Swagger/Redoc UI) |

The application delivers its user interface exclusively through Django's server-side
template rendering. Django Admin is the primary user-facing shell; django-grappelli
provides a customised Admin skin with its own CSS/JavaScript bundle (including a
bundled jQuery fork exposed as `grp.jQuery`). No separate frontend build process,
no `package.json`, no Node.js toolchain, and no JavaScript framework (React, Vue,
Angular, Svelte, etc.) are present anywhere in the repository.

## UI Project Boundaries

The UI is not a separate deployable project. It is embedded within the single Django
application at the repository root.

| Boundary | Root Path | Contents |
|----------|-----------|----------|
| Primary UI root | `koalixcrm/` | All Django apps that register Admin classes |
| Global template overrides | `koalixcrm/templates/` | Admin change-form, change-list, app-index overrides |
| Core template overrides | `koalixcrm/core/templates/` | Admin base site, workspace header, dashboard widget, and custom CRM views |
| Global static assets | `koalixcrm/core/static/` | XSL document templates, fonts, logo |
| Accounting static assets | `koalixcrm/accounting/static/` | Accounting-specific XSL document templates |
| Project-level static overrides | `projectsettings/static/` | Project-scope XSL document templates, fonts, logo |
| URL configuration | `projectsettings/urls.py` | Root URLconf — mounts Admin, Grappelli, REST API, OIDC auth, and legacy HTML views |
| Dashboard configuration | `projectsettings/dashboard.py` | Grappelli custom dashboard definition |

There are no shared UI library packages, no monorepo workspace siblings, and no
`node_modules` directory.

## Universal Abstraction Mapping

The table below maps the universal UI abstractions to their Django/Grappelli
equivalents as used in this project.

| Universal Term | Django / Grappelli Equivalent | Notes |
|---------------|------------------------------|-------|
| Screen / Page | `ModelAdmin` class registration + URL route | Each registered model produces a change-list page and a change-form page automatically |
| Component / Widget | Django Admin inline (`TabularInline`, `StackedInline`), Grappelli collapsible group | Inlines render as embedded sub-forms within a change-form page |
| Form | Django `ModelForm` + Django Admin form fields | Generated from `ModelAdmin.fields` / `fieldsets`; validated and submitted server-side |
| Wizard / Flow | Multi-step Admin action (intermediate confirmation screens) | Used for payment registration and exception confirmation (crm/admin/ templates) |
| Dialog / Modal | Grappelli popup (related-object lookup, add-another) | Triggered by `showRelatedObjectLookupPopup` / `showAddAnotherPopup` — standard Grappelli mechanism |
| Navigation | Django URL routing + Grappelli breadcrumb band | All navigation is server-side URL transitions; breadcrumbs rendered in `change_form.html` and `change_list.html` |
| Layout | Grappelli grid classes (`grp-module`, `g-d-*`, `l-2cr-fluid`) | CSS grid defined by Grappelli skin |
| State Management | Django session + server-side context processors | `WorkspaceContextMiddleware` injects active workspace; `TimezoneMiddleware` manages user timezone; no client-side state store |
| Theme / Styling | Grappelli skin CSS + inline override styles in templates | No separate design-token file; accent colour per workspace defined as `active_workspace_color` |
| Data Binding | Django Template Language one-way rendering + jQuery AJAX for dynamic dropdowns | `time_reporting.html` uses `$.getJSON` to repopulate the task dropdown when a project is selected |

## Folder-to-Abstraction Mapping

| Folder Path | Universal Abstraction | Framework-Specific Term | Contents Summary |
|------------|----------------------|------------------------|------------------|
| `koalixcrm/templates/admin/` | Screen / Page | Admin template overrides | 3 files: `app_index.html`, `change_form.html`, `change_list.html` — project-wide overrides of Grappelli admin page chrome |
| `koalixcrm/core/templates/admin/` | Screen / Page, Component / Widget | Admin base + workspace component | 3 files: `base_site.html` (Admin base), `workspace_header.html` (workspace colour band), `dashboard/workspace_switcher.html` (Grappelli dashboard module) |
| `koalixcrm/core/templates/crm/admin/` | Screen / Page, Form, Wizard / Flow | Custom CRM views | 4 files: `exception.html`, `register_payment.html`, `set_timezone.html`, `time_reporting.html` |
| `koalixcrm/core/admin/` | Screen / Page | ModelAdmin registrations | Currency, tax, unit, workspace, role-in-workspace, PDF export process admin classes |
| `koalixcrm/contracts/admin/` | Screen / Page, Form | ModelAdmin registrations | Commercial document, contract, quotation, sales order, invoice, credit note, purchase order, despatch advice, payment reminder admin classes |
| `koalixcrm/contacts/admin/` | Screen / Page, Form | ModelAdmin registrations | Organization, party, address, phone number, email, billing cycle, party group admin classes |
| `koalixcrm/products/admin/` | Screen / Page, Form | ModelAdmin registrations | ProductType admin class |
| `koalixcrm/accounting/admin/` | Screen / Page, Form | ModelAdmin registrations | Accounting model admin classes |
| `koalixcrm/reporting/admin/` | Screen / Page, Form | ModelAdmin registrations | Project, task, agreement, estimation, human resource, work, reporting period admin classes (21 files) |
| `koalixcrm/djangoUserExtension/admin/` | Screen / Page, Form | ModelAdmin registrations | Document template, template set, user extension admin classes |
| `koalixcrm/subscriptions/admin/` | Screen / Page, Form | ModelAdmin registrations | Subscription admin classes |
| `koalixcrm/reporting/views/` | Screen / Page, Form | Django function-based views | `time_tracking.py` (time-reporting form), `create_task.py`, `reporting_period_missing.py`, `user_is_not_human_resource.py` |
| `koalixcrm/core/views/` | Screen / Page, Form | Django class-based / function-based views | `set_timezone.py` (timezone selection form), `workspace_switch.py` (workspace switch POST handler) |
| `koalixcrm/core/static/default_templates/` | Theme / Styling | XSL document templates | 13 XSL files for invoice, quotation, sales order, purchase order, despatch advice, work report, project report in `de/` and `en/`; fonts and font config in `generic/` |
| `koalixcrm/accounting/static/default_templates/` | Theme / Styling | XSL document templates | 4 XSL files for balance sheet and profit-loss statement in `de/` and `en/` |
| `projectsettings/static/default_templates/` | Theme / Styling | XSL document templates | 18 XSL files covering all document types (including balance sheet and profit-loss statement) in `de/` and `en/`; fonts and font config in `generic/` |
| `koalixcrm/*/locale/` | Theme / Styling (i18n) | Django gettext `.po`/`.mo` files | Translation catalogues for `de`, `en`, `es`, `fr`, `pt_BR` across 7 apps |
| `projectsettings/dashboard.py` | Navigation, Layout | Grappelli `CustomIndexDashboard` | Defines module groups, model lists, and link lists for the Admin dashboard landing page |
| `projectsettings/urls.py` | Navigation | Django URLconf | Root URL dispatcher: Admin, Grappelli, OIDC auth, REST API, legacy HTML reporting views |

## Feature Inventory

The Admin-based UI is organised around the domain modules defined by the Django app
structure. Each feature group maps to one or more `ModelAdmin` classes registered
under a Grappelli dashboard group.

| Feature | Screens (Admin Pages) | Key Forms / Wizards | State / Context | Description |
|---------|----------------------|---------------------|-----------------|-------------|
| Workspace Management | Workspace change-list, Workspace change-form, Workspace switcher (dashboard module + header band) | Workspace switch POST form (`workspace_header.html`, `workspace_switcher.html`) | Active workspace in session via `WorkspaceContextMiddleware` | Multi-tenant workspace isolation; users can switch the active workspace from the dashboard or from the header band on every page |
| Authentication / OIDC | Login selection view, OAuth callback view, Logout view | Login form (Keycloak redirect) | Django session | OIDC-based login via Keycloak; login selection page offers provider choice |
| Contacts (Parties) | Organization, PartyContact, Party, PartyRole, OrganizationMembership, OrganizationRelationship, Address, AddressAssignment, PhoneNumber, PhoneAssignment, PartyEmail, EmailAssignment change-lists and change-forms | Contact creation and edit forms; related-object lookups via Grappelli popup | Workspace-scoped session | CRUD management of organisations, persons, addresses, phone numbers, and e-mail addresses |
| Products | ProductType change-list, ProductType change-form | Product type creation and edit form | Workspace-scoped session | Management of product types |
| Contracts and Commercial Documents | Contract, Quotation, SalesOrder, Invoice, CreditNote, PurchaseOrder, DespatchAdvice, PaymentReminder change-lists and change-forms; payment registration intermediate form | Change forms with inline document positions; payment amount entry wizard (`register_payment.html`); exception confirmation wizard (`exception.html`) | Workspace-scoped session | Full commercial document lifecycle from quotation to invoice and credit note; PDF export triggered via Admin actions |
| Accounting | Accounting model change-lists and change-forms; balance sheet and profit-loss XSL templates | Accounting entry forms | Workspace-scoped session | Chart of accounts, transactions, balance sheet, profit-loss statement |
| Reporting (Projects and Time) | Project, Task, Agreement, Estimation, HumanResource, Work, ReportingPeriod change-lists and change-forms; time-tracking form page | Time-tracking formset (`time_reporting.html`) with dynamic task dropdown; timezone selection form (`set_timezone.html`) | Workspace-scoped session; user timezone in session via `TimezoneMiddleware` | Project and task management; time and expense reporting by human resources |
| User Extensions and Templates | DocumentTemplate, TemplateSet, UserExtension change-lists and change-forms | Document template and user extension edit forms | Workspace-scoped session | Per-user document template sets; XSL/font upload via django-filebrowser |
| Subscriptions | Subscription change-list, Subscription change-form | Subscription edit form | Workspace-scoped session | Subscription record management |
| API Documentation (Swagger / Redoc) | Per-app Swagger UI and Redoc pages | None (read-only schema browsers) | Stateless | drf-spectacular generates OpenAPI schemas per app; SpectacularSwaggerView and SpectacularRedocView provide interactive API documentation |

## Recommended Documentation Tasks

The following `QQ_UI_Doc_*.md` files are recommended to document the UI in detail.
Each corresponds to one feature group identified above.

| Recommended File | Scope |
|-----------------|-------|
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_WorkspaceManagement.md` | Workspace switch screen, header band component, dashboard switcher widget, session state |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_Authentication.md` | Login selection screen, OIDC redirect flow, logout |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_Contacts.md` | Organization, party, address, phone, e-mail change-list and change-form screens |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_Products.md` | ProductType change-list and change-form screens |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_CommercialDocuments.md` | Contract, quotation, sales order, invoice, credit note, purchase order, despatch advice, payment reminder screens; payment registration wizard; exception confirmation wizard |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_Accounting.md` | Accounting change-list and change-form screens; balance sheet and profit-loss statement XSL output |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_Reporting.md` | Project, task, agreement, estimation, human resource, work, reporting period screens; time-tracking formset screen; timezone selection screen |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_UserExtensions.md` | Document template, template set, user extension screens; filebrowser integration |
| `doc/08_cross_cutting_concepts/QQ_UI_Doc_AdminGlobal.md` | Global Admin page chrome: `change_form.html`, `change_list.html`, `app_index.html` overrides; Grappelli skin; navigation and breadcrumb structure |
