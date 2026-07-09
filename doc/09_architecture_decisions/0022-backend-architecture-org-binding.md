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

KoalixCRM follows the org backend ADRs (0001–0012) as-written. The abstract terms in those ADRs
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
| the product's architect role | `dev-kxcrm-architect` |
| the product's Python QA role | `dev-kxcrm-python-qa-engineer` |
| the QA import-boundary gate | greps `koalixcrm_microservices/*` and `*_api_py/*` for forbidden Django imports |
| default branch | `develop` <!-- corrected main→develop 2026-07-08 (QUAQ2-227); this correction is pending product-owner confirmation --> |

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
