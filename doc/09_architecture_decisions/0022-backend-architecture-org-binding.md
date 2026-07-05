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

KoalixCRM follows the org backend ADRs (0001–0006) as-written. The abstract terms in those ADRs
bind to the following KoalixCRM concretes:

| Abstract term (org ADR) | KoalixCRM concrete |
|-------------------------|--------------------|
| org SQS command-envelope package | `koalixcrm_mq_commands` (`envelope.py`, `BaseCommand`, command classes) |
| the product's microservice package | `koalixcrm_microservices` (Celery app + SQS poller) |
| the shared `BaseAPIClient` | `koalixcrm.shared.api_client.BaseAPIClient` |
| shared utils package | `koalixcrm_utils` |
| canonical factory path | `tests/factories/` |
| canonical DTO path | `{app}_api_py/dto/{m}.py` (**singular** `dto/`) |
| the product's architect role | `dev-kxcrm-architect` |
| the product's Python QA role | `dev-kxcrm-python-qa-engineer` |
| the QA import-boundary gate | greps `koalixcrm_microservices/*` and `*_api_py/*` for forbidden Django imports |
| default branch | `main` |

## Notes

- Verified against the codebase: `BaseAPIClient` lives in `koalixcrm/shared/api_client.py`; the
  `*_api_py` clients subclass it (`KoalixCRMContactsAPIClient`, `KoalixCRMContractsAPIClient`,
  `KoalixCRMAccountingAPIClient`, …). `*_api_py` packages use a **singular** `dto/` directory
  (`contacts_api_py/dto/`, `products_api_py/dto/`, …). Factories live under `tests/factories/`.
  `koalixcrm_mq_commands` holds `envelope.py` + command classes; `koalixcrm_microservices` holds
  `celery_app.py` + `sqs_poller.py`.
- KoalixCRM's DTO directory (`dto/`, singular) and `tests/factories/` layout differ from other
  QuantalQ backends. The org ADR (code-structure) deliberately leaves the DTO/factory paths
  product-bound, so `dto/` is **canonical** for KoalixCRM — not "legacy".

## Consequences

- New backend work in this repo is reviewed against the org ADRs via `dev-kxcrm-architect` /
  `dev-kxcrm-python-qa-engineer`. When the org ADRs change, update only this table — the rules
  themselves are not duplicated here.
