# ADR-0001: Replacement of Django Admin with a Next.js-based Admin UI

- **Status:** Accepted (Option C — Refine.dev on Next.js)
- **Date:** 2026-04-24 (revised same day; original draft proposed Option E)
- **Deciders:** @scaphilo, @Hacont
- **Context tag:** Frontend / Admin UI / CRUD
- **Supersedes:** —
- **Superseded by:** —

## 1. Context

koalixcrm currently ships its operator UI through the built-in Django Admin
(themed with `django-grappelli==3.0.10`, backed by `django-filebrowser`).
The business logic is already split into domain apps (`contacts`, `contracts`,
`accounting`, `products`, `reporting`, `core`) each with a sibling
`*_api_py/` package that exposes DRF endpoints. Dependencies already include
`djangorestframework==3.16.0`, `django-filter==25.1`, and
`djangorestframework-xml==2.0.0`.

### 1.1 Target runtime architecture (out of scope for this ADR to decide,
but in scope as *constraints*)

```
           ┌────────────┐      ┌──────────────────────┐
  client ──▶ nginx/ALB  ├──┬──▶│  Next.js admin (SSR) │
           └────────────┘  │   └──────────┬───────────┘
                           │              │ REST/JSON
                           │              ▼
                           │   ┌──────────────────────┐
                           └──▶│  Django = CRUD API   │ ◀── ModelAdmin
                               │  (DRF only, no HTML) │     metadata
                               └──────────┬───────────┘
                                          │
                          ┌───────────────┼──────────────────┐
                          ▼               ▼                  ▼
                    PostgreSQL      ElasticMQ / SQS     MinIO / S3
                                          │
                                          ▼
                            Celery workers  +  Java/Spring beans
                                          (domain microservices)
```

### 1.2 Constraints derived from the target architecture

1. Django must remain the **single writer** to the relational model but should
   expose **only** a JSON REST API — no server-rendered HTML, no session-
   based admin.
2. Async workloads leave Django via **ElasticMQ/SQS → Celery** (Python) and
   **SQS → Spring beans** (Java). The admin UI must therefore be able to
   surface job status retrieved from a read-side API, not from Django's
   request/response cycle.
3. Object storage is **MinIO/S3**; `django-filebrowser`'s local-filesystem
   assumption must be replaced by signed-URL upload/download flows.
4. Routing terminates at **nginx / AWS ALB**; SSR for the admin is an
   acceptable but optional benefit (internal tool, low SEO value, but
   perceived speed matters).
5. **Non-functional, explicit requirement:** the *configuration* of each
   screen (fields shown in list, filters, search, inlines, permissions)
   should stay in Python, ideally in the existing `admin.py` files. The
   JS frontend should be a **generic renderer** of that configuration, not
   a parallel source of truth.
6. **License requirement:** all core components in the final stack must be
   under a **permissive OSI license** (MIT / BSD / Apache-2.0). Copyleft
   (GPL/AGPL) or source-available / BSL / "commercial extensions required
   for business use" licenses are disqualifying for koalixcrm distribution.

## 2. Decision drivers

| # | Driver                                                  | Weight |
|---|---------------------------------------------------------|--------|
| D1 | License is permissive (MIT/BSD/Apache-2.0)             | must   |
| D2 | ModelAdmin config stays in `admin.py` (Python)         | ~~must~~ dropped — see §4 |
| D3 | Works with existing DRF API layout                     | must   |
| D4 | Next.js / SSR capable                                  | must   |
| D5 | Mature, maintained, non-trivial community              | should |
| D6 | Incremental migration path (per domain app)            | should |
| D7 | Low boilerplate per new model                           | should |
| D8 | File/attachment handling via S3/MinIO presigned URLs   | should |

## 3. Options considered

### Option A — Keep Django Admin, modernize with **Django Unfold**

- **What:** Tailwind-based reskin of the built-in admin. `admin.py` stays
  authoritative. Adds dashboard widgets, sidebar, dark mode.
- **License:** MIT.
- **D1** ✅ · **D2** ✅ (by definition) · **D3** n/a · **D4** ❌ (no Next.js,
  still server-rendered HTML) · **D5** ✅ · **D6** ✅ (drop-in) · **D7** ✅
  · **D8** ⚠️ (still needs a storage backend like `django-storages`).
- **Verdict:** Does *not* satisfy the target architecture (Django would keep
  rendering HTML). Included as a **baseline / fallback** only.

### Option B — **React-Admin** (Marmelab) as a separate SPA

- **What:** React + Material-UI framework, ships data providers, list/edit
  views, bulk actions, undo. Backed by DRF via a community data provider.
- **License:** **MIT** for the core (`ra-core`, `ra-ui-materialui`,
  `react-admin`). A paid **react-admin Enterprise Edition** exists with
  additional modules (audit log, RBAC, realtime, editable datagrid) — these
  are *optional* and not required for basic use.
- **D1** ✅ (core MIT; avoid EE modules to stay permissive) · **D2** ❌
  (configuration is JS/TSX `<Resource/>` declarations; duplicates `admin.py`)
  · **D3** ✅ · **D4** ❌ (explicit: no SSR/Next.js support) · **D5** ✅
  (very mature) · **D6** ✅ · **D7** ✅ (least boilerplate of all JS options)
  · **D8** ✅ (file input components + presigned URL pattern documented).
- **Verdict:** Best DX of the JS options, but violates **D2** (Python-
  authoritative config) and **D4** (no Next.js). Fails the user-stated
  must-haves.

### Option C — **Refine.dev** as a Next.js app

- **What:** Headless React framework for internal tools. Next.js App Router
  supported. Backend-agnostic; generic REST / simple-rest data provider
  works against DRF. Pick any UI kit (Ant Design, MUI, Chakra, shadcn).
- **License:** **MIT** throughout (`@refinedev/core`, data providers,
  Next.js router binding). An optional hosted/AI product ("Refine AI") is
  commercial but not required.
- **D1** ✅ · **D2** ❌ by default (resources configured in TSX) — can be
  salvaged by driving `<Resource/>` props from a Django-served metadata
  endpoint (see Option E) · **D3** ✅ · **D4** ✅ · **D5** ✅ · **D6** ✅
  · **D7** ⚠️ (more boilerplate than react-admin per the Marmelab
  comparison) · **D8** ✅.
- **Verdict:** Strong on architecture fit, weak on **D2** unless combined
  with a Python-side metadata provider.

### Option D — **Unfold Turbo** boilerplate (Django + Next.js)

- **What:** Official Unfold-team scaffold bundling Django (with Unfold
  admin) + a Next.js app with a pre-wired user/group/auth bridge.
- **License:** MIT.
- **D1** ✅ · **D2** ⚠️ (Django admin stays Python-driven; the Next.js
  side is hand-rolled per screen — no auto-generation) · **D3** ✅ (DRF
  pattern) · **D4** ✅ · **D5** ⚠️ (young, small community) · **D6** ⚠️
  (opinionated; hard to retrofit onto an existing Django project of
  koalixcrm's size) · **D7** ❌ (every screen hand-written) · **D8** ✅.
- **Verdict:** Nice reference project, but doesn't solve the core
  "configure once in Python" problem.

### Option E — **Metadata-driven**: `django-api-admin` (or equivalent)
      \+ Refine/React-Admin as a generic renderer

- **What:** A Django package that **reads the existing `ModelAdmin`
  registrations** (`list_display`, `list_filter`, `search_fields`,
  `fieldsets`, `inlines`, `readonly_fields`, `actions`, `get_queryset`,
  permissions) and exposes both the *data* endpoints and a *schema /
  metadata* endpoint (`OPTIONS` / dedicated `/admin-api/config/`). A
  generic Next.js client (built on Refine, or hand-rolled with
  React Query + a UI kit) fetches that metadata at runtime and
  renders list/detail/form views.
- **Candidate packages** (all MIT):
  - [`django-api-admin`](https://pypi.org/project/django-api-admin/) —
    closest mirror of `django.contrib.admin`; drf-spectacular integration.
  - [`django-rest-admin`](https://github.com/inmagik/django-rest-admin) —
    explicitly exposes a *meta* endpoint describing all registered models,
    "to build a client application acting as an admin".
  - [`django-restful-admin`](https://pypi.org/project/django-restful-admin/)
    — `admin.py`-style registration, DRF endpoints.
  - [`modern-django-admin`](https://github.com/asbilim/modern-django-admin)
    — returns UI component hints per field; targets Next.js explicitly.
    ⚠️ License not declared on the repo landing page — **must be
    verified** before adoption (would be disqualifying under D1 if not
    permissive).
- **D1** ✅ (for the first three; TBD for the fourth) · **D2** ✅ (the
  whole point) · **D3** ✅ · **D4** ✅ (Next.js is just the renderer)
  · **D5** ⚠️ (all of these are small projects; bus factor is real)
  · **D6** ✅ (can ship one app at a time; others stay on classic admin)
  · **D7** ✅ (new model = `admin.register(Model)` and it appears)
  · **D8** ⚠️ (file uploads need custom field type → presigned MinIO URL).
- **Verdict:** The only option that satisfies **all three** musts
  (license, Python-authoritative config, DRF-only Django). The risk is
  maturity of the metadata packages, which we mitigate by treating the
  metadata contract as our own (copy/fork the package if needed).

## 4. Decision

**Adopt Option C — Refine.dev on Next.js**, accepting that resource
configuration lives in TSX rather than Python (D2 is dropped):

1. Build the frontend as a **Next.js 14+ App Router** project, using
   **Refine (`@refinedev/core` + `@refinedev/nextjs-router`, MIT)** for
   routing, auth, and the data-provider layer. Resources are declared
   directly in TSX via `<Resource/>`; we accept the duplication relative
   to `admin.py` as the price of staying inside Refine's idiomatic
   model and avoiding a bespoke metadata renderer.
2. Pick a UI kit per the standard Refine integrations
   (shadcn/ui + Tailwind preferred for permissive licensing and
   styling control). Use Refine's CRUD primitives (`useTable`,
   `useForm`, `<List/>`, `<Edit/>`, `<Show/>`) rather than building
   our own.
3. Django keeps the existing DRF endpoints in each `*_api_py/`
   package and serves them under `/api/` (no metadata endpoint
   required for the chosen approach). The DRF `simple-rest`-style
   data provider in Refine consumes them directly. Pagination,
   filtering, and ordering follow DRF conventions.
4. Replace `django-filebrowser` with `django-storages[s3]` +
   server-issued **presigned MinIO/S3 URLs**; the Next.js admin
   uploads directly to object storage, then PATCHes the resulting key
   back to Django.
5. Async indicators (Celery / Java bean job state) are surfaced by a
   read-only Django endpoint backed by the workers' result store
   (or a dedicated status table written by the workers via SQS
   round-trip). The admin UI polls or subscribes, never dispatches
   work directly.
6. nginx/ALB terminates TLS, routes `/api/*` to Django and all other
   paths to the Next.js app.
7. The classic Django Admin (`/admin/`) stays mounted during
   migration as an emergency fallback and for screens not yet ported.

### Rationale

- **License (D1):** every must-have component (Django, DRF, Refine
  core + Next.js router binding, Next.js, Tailwind, shadcn,
  `django-storages`) is MIT or BSD. We explicitly avoid Refine's
  commercial "Refine AI" add-on. No GPL, AGPL, BSL, or
  "commercial for production" terms in the critical path.
- **Why D2 was dropped:** maintaining a Python-authoritative metadata
  contract requires either a small/fragile third-party package
  (`django-api-admin` et al.) or owning a non-trivial metadata layer
  ourselves. Both carry real bus-factor risk. We prefer to live with
  TSX resource declarations and rely on Refine's mature, off-the-shelf
  primitives.
- **Next.js / SSR (D4):** Refine has first-class Next.js App Router
  support; perceived speed for an internal tool matters and the SSR
  path keeps doors open for future customer-facing portals on the
  same stack.
- **Clean Django role (D3, target architecture):** Django shrinks to
  models + DRF + auth, runs behind the ALB as a pure JSON service —
  matching how MinIO, ElasticMQ, and the Java beans are already
  consumed. `django-grappelli` and `django-filebrowser` can be
  removed once cut-over completes.
- **Incremental migration (D6):** the classic admin stays mounted at
  `/admin/` throughout migration; the Next.js UI is added alongside
  and cut over app-by-app.

### What we accept by dropping D2

- `admin.py` and the Refine resources are two parallel catalogues of
  "what fields appear where, what filters exist, what's editable".
  They can drift out of sync. This is a known cost.
- Mitigation: once the Next.js admin covers a domain app fully, the
  classic admin for that app becomes optional and can be deleted —
  removing the second catalogue rather than keeping both alive.

## 5. Consequences

### Positive

- Single Python source of truth for admin screens; developers keep the
  `admin.py` workflow they already use.
- Django's surface area shrinks to "models + DRF + metadata"; easier to
  scale horizontally behind the ALB, easier to containerize, no session/
  CSRF coupling for the JS app (use JWT or DRF token auth).
- Next.js front door gives us SSR, better perceived performance, and a
  path to add customer-facing portals later on the same stack.
- Object storage, queueing, and microservice work (Celery / Java beans)
  stop being coupled to Django request lifecycles.

### Negative / Risks

- **Two catalogues of screens** (R1, replaces former R1): `admin.py`
  and Refine `<Resource/>` declarations both describe what appears
  where. They can drift. Mitigation: delete the classic admin per
  app once its Refine equivalent is complete, so the duplication is
  bounded in time, not permanent.
- **Per-screen boilerplate** (R2): Refine has less generation than a
  metadata renderer would; each new model needs a hand-written
  resource (list / show / edit). Acceptable trade-off in exchange
  for using mature, idiomatic primitives.
- **File handling rewrite** (R3): dropping `django-filebrowser` means
  every model using `FileBrowseField` must migrate to `FileField` +
  S3 backend. Inventory needed before cut-over.
- **Auth redesign** (R4): admin session cookies disappear. Choose JWT
  (short-lived) + refresh, or DRF `TokenAuthentication`, and integrate
  with the future OIDC plan (see `project_koalixcrm_no_keycloak`).
- **DRF data-provider edge cases** (R5): the simple-rest data
  provider expects certain pagination/filter conventions. Custom DRF
  viewsets may need small adjustments (or a custom data provider) to
  match.

### Neutral

- Grappelli and filebrowser dependencies can be removed once the cut-over
  completes.
- The Django admin URL (`/admin/`) can remain mounted as an emergency
  fallback indefinitely — it costs nothing.

## 6. License matrix (summary)

| Component                    | License       | Verified |
|------------------------------|---------------|----------|
| Django                       | BSD-3-Clause  | ✅        |
| Django REST framework        | BSD-2-Clause  | ✅        |
| `django-api-admin`           | MIT           | ✅ (PyPI) |
| `django-rest-admin` (inmagik)| MIT           | ✅ (repo) |
| `django-storages`            | BSD-3-Clause  | ✅        |
| Next.js                      | MIT           | ✅        |
| Refine (`@refinedev/*` core) | MIT           | ✅        |
| React-Admin (core)           | MIT           | ✅        |
| React-Admin Enterprise       | Commercial    | ✅ (avoid)|
| Django Unfold                | MIT           | ✅        |
| Unfold Turbo                 | MIT           | ✅        |
| `modern-django-admin`        | **unspecified** | ❌ (DO NOT adopt until clarified) |

## 7. Migration outline (non-binding, for planning)

1. **Phase 0 — spike** (1–2 weeks): scaffold a Next.js 14+ App Router
   project with Refine + shadcn/ui; configure the DRF simple-rest
   data provider against the existing `contacts_api_py` endpoints;
   build a Refine `<Resource/>` for `contacts.Contact` (list / show /
   edit) backed by MinIO-presigned uploads.
2. **Phase 1 — foundation**: auth (JWT), nginx/ALB routing split
   (`/api/*` → Django, rest → Next.js), `django-storages[s3]` swap,
   CI for the Next.js app.
3. **Phase 2 — per-app rollout**: migrate `contacts` → `products` →
   `contracts` → `accounting` → `reporting` → `core`. Each app:
   Refine resources written in TSX, classic admin for that app
   removed once the Refine version reaches parity.
4. **Phase 3 — decommission**: remove `django-grappelli`,
   `django-filebrowser`, and any remaining `admin.site.register`
   customizations.

## 8. Open questions

- Q1: Per-field **custom widgets** (signature pad, PDF preview,
  invoice line editor) — written as bespoke React components inside
  the relevant Refine resource. Confirm the inventory of such
  widgets before Phase 2 to avoid surprises mid-rollout.
- Q2: Which auth provider bridges the Next.js app? (Current project
  memo: no Keycloak in local setup — decision still open.)
- Q3: Do Java microservices need admin-side visibility (job submit,
  cancel, retry) or is read-only status enough for v1?
- Q4: Should we keep one Refine project for the whole admin, or
  split per domain app once the codebase grows? Defer until after
  Phase 2 starts.

## 9. References

- [Refine (GitHub)](https://github.com/refinedev/refine) — MIT
- [Refine licence page](https://refine.dev/docs/3.xx.xx/licence/)
- [React-Admin (GitHub)](https://github.com/marmelab/react-admin) — MIT core
- [React-Admin vs Refine (Marmelab)](https://marmelab.com/blog/2023/07/04/react-admin-vs-refine.html)
- [Refine vs React-Admin (Refine blog)](https://refine.dev/blog/refine-vs-react-admin/)
- [Django Unfold (GitHub)](https://github.com/unfoldadmin/django-unfold) — MIT
- [Unfold Turbo](https://github.com/unfoldadmin/turbo) — MIT
- [`django-api-admin` (PyPI)](https://pypi.org/project/django-api-admin/) — MIT
- [`django-rest-admin` (inmagik)](https://github.com/inmagik/django-rest-admin)
- [`django-restful-admin` (PyPI)](https://pypi.org/project/django-restful-admin/)
- [`modern-django-admin`](https://github.com/asbilim/modern-django-admin) — license TBD
- [DRF Metadata docs](https://www.django-rest-framework.org/api-guide/metadata/)
