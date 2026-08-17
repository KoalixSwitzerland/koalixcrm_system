# 0022 — Backend Architecture: Org-Wide ADR Binding

- **Status:** Accepted
- **Date:** 2026-07-05

## Context

The KoalixCRM backend is governed by QuantalQ's **org-wide backend-architecture ADRs** (CRUD-REST
API style; Django as a CRUD-only provider with algorithms in workers; `*_api_py` inter-service
contracts; SQS messaging; async `*Process` orchestration; one-file-per-model code structure).
Those ADRs are maintained centrally in the internal `template-backend-dionysos` repository and are
**not reproduced here** — this repo is open source, and the ADR documents are internal QuantalQ
material. This binding is the small, public lookup table that maps the org ADRs' abstract terms to
the concrete names KoalixCRM uses, so the codebase stays self-describing without vendoring the
internal ADRs.

## Decision

KoalixCRM follows the org backend ADRs (0001–0013) as-written. The abstract terms in those ADRs
bind to the following KoalixCRM concretes:

| Abstract term (org ADR) | KoalixCRM concrete |
|-------------------------|--------------------|
| org SQS command-envelope package | `koalixcrm_mq_commands` (`envelope.py`, `BaseCommand`, command classes) |
| the product's microservice package | `koalixcrm_microservices` (Celery app + SQS poller) |
| the shared `BaseAPIClient` | `koalixcrm.shared.api_client.BaseAPIClient` |
| shared utils package | `koalixcrm_utils` |
| canonical DTO path | `{app}_api_py/dto/{m}.py` (**singular** `dto/`) |
| legacy factory path(s) | none — all factories migrated to `koalixcrm/{app}/tests/factories/{m}_factory.py` (QUAQ2-227) |
| legacy test path(s) | `tests/legacy_crm/` — holds `test_version_increase.py` only (a package-version meta-test asserting `koalixcrm.version.KOALIXCRM_VERSION` bumped; no single app owner). Root cross-cutting tiers `tests/e2e/`, `tests/integration/`, and `tests/test_fork_isolation.py` are ADR-0012 tiers, not legacy. |
| pytest import mode | `prepend` (pytest default) — every `{app}/tests/` and `{app}/tests/factories/` is a Python package (`__init__.py`), so module names stay unique and no `--import-mode=importlib` is needed |
| the group–role–workspace grant model | `koalixcrm.core.models.access.RoleInWorkspace` (`group`, `workspace`, `role`) |
| the canonical access module | `koalixcrm/core/access.py` — currently `effective_roles()`, `user_workspaces()`, `permissions_for_role()` only; `roles_in_workspace()` and the unrestricted-actor predicate are **not yet implemented** (see layer status below) |
| the per-app role-policy module | not yet implemented — target `koalixcrm/{app}/permission_policy.py` |
| the service-account group setting | not yet implemented — M2M clients are auto-provisioned by username in `koalixcrm/auth/m2m_authentication.py` |
| the product's architect role | `dev-kxcrm-architect` |
| the product's Python QA role | `dev-kxcrm-python-qa-engineer` |
| the QA import-boundary gate | greps `koalixcrm_microservices/*` and `*_api_py/*` for forbidden Django imports |
| default branch | `develop` <!-- corrected main→develop 2026-07-08 (QUAQ2-227); this correction is pending product-owner confirmation --> |

## Isolation & authorization layer status (org ADR-0008 / ADR-0013)

Org ADR-0013 requires this status to be recorded rather than assumed. **Line 3 / layer 0 is not a
maturity gap but a compliance gap:** ADR-0008 has required authorization of the URL `workspace_id`
since it was accepted, and KoalixCRM mounts every app under that segment without checking it.

| Layer | Status | Concrete / tracking |
|-------|--------|---------------------|
| ADR-0008 line 1 — scoped model | in force | `WorkspaceScopedModel` + `koalixcrm/core/managers/workspace_aware.py` |
| ADR-0008 line 2 — crypto binding | not applicable yet | no tenant-scoped credential store in this product |
| ADR-0008 line 3 — URL authorization (ADR-0013 layer 0) | **not in force — compliance gap** | [koalixcrm#428](https://github.com/KoalixSwitzerland/koalixcrm/issues/428). Scoping comes from the session via `WorkspaceContextMiddleware`; the `<workspace_id>` path segment is unauthorized. |
| ADR-0008 line 4 — RLS | planned | no Postgres RLS policies |
| ADR-0008 line 5 — connection-bound outbound calls | planned | |
| ADR-0013 layer 1 — model permissions | in force | `koalixcrm/shared/permissions.py::ModelPermissionsWithListView`; three `stock` views declare `IsAuthenticated` only (in scope of #428) |
| ADR-0013 layer 2 — workspace-scoped role narrowing | not in force | [koalixcrm#429](https://github.com/KoalixSwitzerland/koalixcrm/issues/429) — needs layer 0 (#428) and the policy projection first |
| ADR-0013 — role-policy projection | not in force | [koalixcrm#429](https://github.com/KoalixSwitzerland/koalixcrm/issues/429). `Group.permissions` is maintained by hand. `koalixcrm/core/access.py::permissions_for_role()` is a hardcoded role→verb map referenced **only from tests** — no production path consults a role. |
| ADR-0008 / ADR-0013 negative-test gate | not in force | no test proves a crossed-workspace read/write is rejected (#428, #429) |

### Role vocabulary

The role names are **not yet aligned with WFS** and diverge in two ways ([#429](https://github.com/KoalixSwitzerland/koalixcrm/issues/429) Part A):
KoalixCRM stores snake_case values (`line_manager`) where WFS stores PascalCase names
(`LineManager`), and four WFS roles are missing entirely — `ContractResponsible`, `OFFICE`,
`SYSTEM` (the service-account role the projection needs) and `Kiosk` (the shared-terminal role,
member of `ROLES_WITHOUT_BASELINE`, which is granted nothing it is not explicitly given).
KoalixCRM also models roles as `models.TextChoices` where WFS uses a `Role` table; the projection
works with either shape, and the decision to keep `TextChoices` is recorded here rather than
treated as accidental.

## Notes

- Verified against the codebase: `BaseAPIClient` lives in `koalixcrm/shared/api_client.py`; the
  `*_api_py` clients subclass it (`KoalixCRMContactsAPIClient`, `KoalixCRMContractsAPIClient`,
  `KoalixCRMAccountingAPIClient`, …). `*_api_py` packages use a **singular** `dto/` directory
  (`contacts_api_py/dto/`, `products_api_py/dto/`, …). Factories now live per-app under
  `koalixcrm/{app}/tests/factories/{m}_factory.py`, and api_py contract tests under
  `koalixcrm/{app}_api_py/tests/`, per org ADR-0012 (test-code-structure & packaging)
  (migrated in QUAQ2-227; the former root `tests/factories/` tree is gone).
  `koalixcrm_mq_commands` holds `envelope.py` + command classes (its
  `koalixcrm_mq_commands/tests/` package holds the django-free contract test);
  `koalixcrm_microservices` holds `celery_app.py` + `sqs_poller.py`.
- KoalixCRM's DTO directory (`dto/`, singular) differs from other QuantalQ backends. The org ADR
  (code-structure) deliberately leaves the DTO path product-bound, so `dto/` is **canonical** for
  KoalixCRM — not "legacy". The factory/test layout is no longer product-bound: org ADR-0012 fixes
  it org-wide at `koalixcrm/{app}/tests/factories/` and `{app}/tests/`, and KoalixCRM now follows
  that (QUAQ2-227).
- The production Docker image and the public PyPI sdist/wheel exclude all `**/tests/` packages
  (`.dockerignore` + prod-Dockerfile prune step; `setup.py` `find_packages(exclude=…)` +
  `MANIFEST.in` prune), so `factory_boy`/`Faker` never enter the production import graph
  (ADR-0012 packaging rule). Test-only deps are available via `pip install koalix-crm[test]`.

## Consequences

- New backend work in this repo is reviewed against the org ADRs via `dev-kxcrm-architect` /
  `dev-kxcrm-python-qa-engineer`. When the org ADRs change, update only this table — the rules
  themselves are not duplicated here.
