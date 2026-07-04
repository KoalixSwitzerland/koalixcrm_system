# Introduction and Goals

## Overview

**koalixcrm** is a free, open-source Customer Relationship Management platform targeting small
businesses with fewer than ten employees. It is licensed under the BSD license and distributed
as a containerised Django application. The system manages contacts, products, commercial
documents (quotations, invoices, purchase orders, and related types), project time tracking,
double-entry accounting, subscriptions, and asynchronous PDF document generation.

The codebase is a modular Django monolith. All eight business-domain apps (`core`, `contacts`,
`contracts`, `products`, `accounting`, `reporting`, `subscriptions`, and `djangoUserExtension`)
share one WSGI process and one PostgreSQL database, while each app enforces its own structural
boundary through a peer-dependency mechanism described in
[Chapter 5: Building Block View](../05_building_block_view/index.md).

The infrastructure stack is orchestrated from the sibling repository
[koalixcrm_system](https://github.com/KoalixSwitzerland/koalixcrm_system).
Setup guides are available in the imported README:
[QQ_IMPORT_README.md](QQ_IMPORT_README.md).

---

## Purpose

koalixcrm solves the following problems for small business operators:

- **Contact management** — tracking organisations, individuals, addresses, phone numbers, and
  email addresses in a single structured contact book.
- **Commercial document lifecycle** — creating and managing quotations, sales orders, invoices,
  purchase orders, credit notes, despatch advices, and payment reminders; each linked to a
  contract and optionally registered in the double-entry accounting system.
- **Project and time tracking** — managing projects, tasks, work records, reporting periods,
  and resource agreements with cost and effort aggregation.
- **Double-entry accounting** — maintaining a chart of accounts, accounting periods, and
  bookings; generating balance sheets and profit/loss statements as PDF documents.
- **High-quality PDF document output** — asynchronously rendering commercial and accounting
  documents via an external Java XSL-FO service, keeping the HTTP response cycle short.

---

## Scope

**Included:**

- All eight Django business-domain apps and their REST APIs.
- The asynchronous PDF export path (SQS queue, external Java PDF export service).
- The OIDC-based authentication layer for browser login and machine-to-machine service
  authentication.
- The multi-tenant workspace isolation mechanism.
- The optional-app fork isolation pattern enabling downstream deployments (such as the WFS
  product) to use only a subset of the apps.

**Excluded:**

- The `koalixcrm_system` sibling repository (infrastructure orchestration, Docker Compose
  configuration).
- The external Java PDF export service (outside the koalixcrm source boundary).
- The OIDC identity provider (external system; Keycloak-compatible).
- AWS SQS and S3 infrastructure (managed external services).

---

## Target Audience

| Audience | How they use this documentation |
|---|---|
| Software architects | Understand the modular monolith structure, peer-dependency graph, and async offload pattern before designing extensions or integrations |
| Software engineers / contributors | Understand package boundaries, inter-module dependencies, and test conventions before contributing |
| Downstream fork teams (e.g. WFS) | Understand the fork-public surface (the five core apps), the optional-app isolation rules, and the swappable dispatcher integration seam |
| Operators / administrators | Understand deployment topology, system requirements, configuration, and upgrade paths |
| CRM users | Understand the feature set and how to navigate the system |

---

## Stakeholders

| Role | Expectations |
|---|---|
| CRM Users | Manage contacts, contracts, products, projects, and accounting through a browser-based admin interface or REST API client |
| Administrators | Configure workspaces, user roles, document templates, and system settings; run upgrades and management commands |
| Downstream fork teams | Consume the fork-public surface of koalixcrm without modifications to the five core apps being invalidated by optional-app updates |
| Open-source contributors | Extend or fix the codebase following the peer-dependency and fork-isolation conventions |
| Hosting operators | Deploy and maintain koalixcrm containers; manage PostgreSQL, SQS, S3, and OIDC provider integrations |

---

## Key Quality Goals

The following quality characteristics are evidenced directly in the codebase and test suite.

| Quality Goal | Evidence |
|---|---|
| **Multi-tenant isolation** | `WorkspaceScopedModel`, `WorkspaceAwareManager`, and `WorkspaceContextMiddleware` enforce per-request workspace scoping via a Python `ContextVar`. All ORM querysets are automatically filtered to the active workspace; the middleware clears the context variable in a `finally` block to prevent cross-request leakage. |
| **Fork isolation** | The five public-surface apps (`core`, `contacts`, `contracts`, `djangoUserExtension`, `products`) must not contain module-level imports from the three optional apps (`accounting`, `reporting`, `subscriptions`). This invariant is enforced by `tests/unit/test_fork_isolation.py` using an AST-level import scanner. |
| **Async PDF generation** | PDF rendering is fully decoupled from the HTTP request cycle. The Django container publishes a `PDFExportCommand` message to an SQS queue and returns HTTP 202 immediately. The external Java service polls the queue, renders via Apache FOP, and writes the result back. This keeps synchronous response times short regardless of document size. |
| **OIDC-first authentication** | Browser login uses the Authorization Code Flow with PKCE via `authlib`. REST API authentication uses RS256-validated Bearer JWTs. Machine-to-machine authentication uses the OAuth 2.0 Client Credentials Grant. A local Django form fallback is provided for development environments without a live OIDC provider. |
| **Modular boundary enforcement** | Each Django app declares `required_peers` and `optional_peers` in its `AppConfig`. The `register_peer_check` helper registers Django system checks that abort startup with a diagnostic error if a required peer app is absent from `INSTALLED_APPS`. |

---

## System Requirements

| Component | Requirement |
|---|---|
| Python | 3.13 |
| Django | 5.2 |
| Database | PostgreSQL 15 (production); SQLite (development / single-user) |
| Message queue | AWS SQS (production); ElasticMQ (development) |
| Object storage | AWS S3 (production); MinIO (development) |
| OIDC provider | Keycloak-compatible (optional; Django form fallback available in development) |
| Containerisation | Docker; orchestration via `koalixcrm_system` (Docker Compose) |
| Supported UI languages | `en-us`, `de`, `fr`, `es`, `pt-br` (set via `KOALIXCRM_LANGUAGE_CODE` environment variable) |

Detailed setup instructions are available in the imported README
([QQ_IMPORT_README.md](QQ_IMPORT_README.md)) and in the deployment view
([Chapter 7: Deployment View](../07_deployment_view/index.md)).

---

## Getting Started

The full setup procedure is documented in the imported README:
[QQ_IMPORT_README.md](QQ_IMPORT_README.md).

The infrastructure is orchestrated from the sibling repository
[koalixcrm_system](https://github.com/KoalixSwitzerland/koalixcrm_system). Two
environment-specific guides are available:

- Local workstation (Docker Desktop): [QQ_IMPORT_docs-setup-local-docker-desktop.md](../07_deployment_view/QQ_IMPORT_docs-setup-local-docker-desktop.md)
- Linux server (VPS / VPC): [QQ_IMPORT_docs-setup-linux-server.md](../07_deployment_view/QQ_IMPORT_docs-setup-linux-server.md)

For a fresh install without existing data:

```bash
python manage.py migrate --settings=projectsettings.settings.development_docker_sqlite_settings
python manage.py createsuperuser --settings=projectsettings.settings.development_docker_sqlite_settings
```

For the upgrade path from v1.14.0 to v2.0.0, see the migration procedure documented in
[QQ_IMPORT_README.md](QQ_IMPORT_README.md) and the architecture decision record in
[Chapter 9: Architecture Decisions](../09_architecture_decisions/index.md).

---

## Anforderungen (System-Design)

Die folgenden funktionalen Anforderungen beschreiben die System-Design-Phase von KoalixCRM
(Produktkatalog, Lagerverwaltung und zugehörige Geschäftsbereiche):

- [REQ-0001: Das System stellt ein einheitliches kanonisches Produktobjekt bereit](requirements/REQ-0001.md)
- [REQ-0002: Das System strukturiert Produkte in Familien und Varianten](requirements/REQ-0002.md)
- [REQ-0003: Das System speichert Produktbeschreibungen in mehreren Sprachen mit Fallback-Kette](requirements/REQ-0003.md)
- [REQ-0004: Das System verwaltet Produktmedien als Verweise auf den Objektspeicher](requirements/REQ-0004.md)
- [REQ-0005: Das System erzwingt den Lifecycle-Status von Produkten](requirements/REQ-0005.md)
- [REQ-0006: Das System stellt einen workspace-isolierten Produktkatalog-API-Endpunkt bereit](requirements/REQ-0006.md)
- [REQ-0007: Das System migriert bestehende ProductType-Daten sicher auf das Product-Modell](requirements/REQ-0007.md)
- [REQ-0008: Das System verwaltet globale und workspace-eigene Klassifizierungsschemata mit hierarchischen Knoten](requirements/REQ-0008.md)
- [REQ-0009: Das System verwaltet typisierte Attributdefinitionen mit Scope, Validierungsregeln und Gruppen](requirements/REQ-0009.md)
- [REQ-0010: Das System speichert Attributwerte in getypten Tabellen mit B-Tree-Indizes](requirements/REQ-0010.md)
- [REQ-0011: Das System verwaltet zeitgebundene Produktpreise mit Parteigruppen-Umrechnungsfaktoren](requirements/REQ-0011.md)
- [REQ-0012: Das System gruppiert Produktpreise in Preislisten nach Kanal oder Kundensegment](requirements/REQ-0012.md)
- [REQ-0013: Das System speichert produktspezifische Maßeinheitenkonversionen datenbankgeführt](requirements/REQ-0013.md)
- [REQ-0014: Das System verknüpft Produkte mit Lieferanten über das Party-Rollenmodell](requirements/REQ-0014.md)
- [REQ-0015: Das System modelliert Stücklisten nach ISA-95 Part 2 für Fertigprodukte und Kits](requirements/REQ-0015.md)
- [REQ-0016: Das System verwaltet Dienstleistungsprofile als 1:1-Erweiterung von Produkten mit kind=SERVICE](requirements/REQ-0016.md)
- [REQ-0017: Das System stellt einen JSONB-Vorhalter für EU-Digitale-Produktpass-Daten bereit](requirements/REQ-0017.md)
- [REQ-0018: Das System verwaltet eine workspace-isolierte, n-stufige Lagerort-Hierarchie mit typisierten Knoten und Barcode-Adressierung](requirements/REQ-0018.md)
- [REQ-0019: Das System führt variantengekeyte Lagermengen als atomare Bestandszeilen mit Eindeutigkeitsgarantie und Bewegungskonsistenz](requirements/REQ-0019.md)
- [REQ-0020: Das System berechnet Available-to-Promise aus virtuellen Bestandszuständen und bindet Reservierungen variantengekeyt an kommerzielle Belege](requirements/REQ-0020.md)
- [REQ-0021: Das System protokolliert jede Lagerbewegung und jeden Lebenszyklus-Touch als unveränderlichen, variantengekeyten Event-Log-Eintrag](requirements/REQ-0021.md)
- [REQ-0022: Das System verfolgt Chargen und Serieneinheiten je ProductVariant mit erzwungener Angabe bei getrackten Bewegungen](requirements/REQ-0022.md)
- [REQ-0023: Das System führt eine lückenlose Lebenszyklus-Historie je SerialUnit im StockMovement-Log](requirements/REQ-0023.md)
- [REQ-0024: Das System verwaltet miet- und kundengeführten Bestand mit Eigentümer-Halter-Trennung](requirements/REQ-0024.md)
- [REQ-0025: Das System unterstützt Kit-Explosion und Fertigungsmontage mit variantengenauer Komponentenbuchung](requirements/REQ-0025.md)
- [REQ-0026: Das System löst gescannte Identifikatoren über eine zentrale, priorisierte Auflösungsschicht auf](requirements/REQ-0026.md)
- [REQ-0027: Das System führt Wareneingänge als Prozess-Aggregat mit Positions-Abweichungsstatus und synchroner Bestandsbuchung](requirements/REQ-0027.md)

---

## Changelog and Release Notes

### Versioning Strategy

The project uses a `major.minor.patch` versioning scheme for stable releases (e.g. `1.14.0`)
and a `V<major>.<minor>-dev<build>` scheme for development snapshots on the active development
branch (e.g. `V1.11-dev810`). Release candidates are tagged with a `-rc<n>` suffix before a
stable tag is created. Starting with the work-in-progress `v2.0.0` milestone the monolithic
`crm` Django app is being split into focused, independently installable apps. See
[Chapter 9: Architecture Decisions](../09_architecture_decisions/index.md) for the recorded
architectural decisions behind this migration.

### Release History

The table below groups the git history by tagged release or, where no tag exists, by time
period. Only significant functional changes are listed; routine CI fixes, test adjustments,
and documentation-only commits are omitted.

#### v2.0.0 (in development — branch `develop`, 2026)

The `v2.0.0` milestone restructures the legacy monolithic `crm` Django app into eight focused,
independently installable Django apps while preserving all `crm_*` database table names for
data compatibility.

| Date | Tag / Commit | Change |
|---|---|---|
| 2026-06-26 | `V1.11-dev810` | Various bugfixes and improvements (QUAQ2-196, QUAQ2-83, QUAQ2-3) |
| 2026-04-29 | `V1.11-dev809` | Commercial document rename; static analysis cleanup |
| 2026-04-24 | `V1.11-dev808` | Rework of PDF creation; introduction of tenant (workspace) isolation |
| 2026-04-18 | `V1.11-dev807` | Reorganisation and rename of the contact models |
| 2026-04-17 | `V1.11-dev806` | Fix residual authentication security findings |
| 2026-04-17 | `V1.11-dev805` | Restore Java-based PDF export service |
| 2026-04-17 | `V1.11-dev804` | Python static code quality improvements |
| 2026-04-17 | `V1.11-dev803` | Refactor: migrate PDF service to Java |
| 2026-04-16 | `V1.11-dev802` | Rename UBL document types |
| 2026-04-15 | — | Preparatory split of the codebase into separate parts for v2.0.0 |

#### v1.14.0 (2024-04-07)

Last stable release of the legacy `crm`-monolith era. Focused on production-container
readiness and security patching.

| Date | Change |
|---|---|
| 2024-04-07 | Correct Dockerfile configuration for production container (#327) |
| 2024-04-06 | Update `psycopg2-binary` to 2.9.9; security patches for Django 3.2.20 and Pillow 7.1.2 (#354) |

#### v1.13.0 (2024-04-04)

| Date | Change |
|---|---|
| 2024-04-04 | Automated PyPI package generation via GitHub Actions (#351) |
| 2024-04-04 | Improve ReadTheDocs configuration and documentation build (#351) |
| 2024-03-31 | Upgrade to Python 3.8 and Django 3.2.20; refactor test suite (#347) |
| 2024-03-31 | Improve production settings security; remove obsolete configuration (#347) |
| 2024-03-31 | Add Codacy coverage reporting integration (#347) |

#### v1.12.1 (2019-06-29)

| Date | Change |
|---|---|
| 2019-06-29 | Upgrade to Django 1.11.21 and `djangorestframework` ≥ 3.9.1 (#289) |
| 2019-06-27 | Add Jenkins pipeline for automated Docker image build and deployment (#283) |
| 2019-06-14 | Extend REST API: add `user` and `user extension` serializers; add time-reporting endpoint (#287) |
| 2019-06-14 | Add REST endpoint for customer email address (#KLX002-14) |
| 2019-05-05 | Fix handling of missing sales document position price (#297) |

#### v1.12dev4 (2018-11-17)

| Date | Change |
|---|---|
| 2018-11-17 | Brazilian Portuguese (`pt_BR`) translation (#220) |
| 2018-11-17 | Code quality improvements across all modules (Codacy findings, #201) |
| 2018-11-17 | Fix planned-cost calculation edge cases in projects (#228, #231) |
| 2018-11-17 | Add `pt_BR` translation files for all Django apps (#220) |

#### v1.11 (2018-01-19)

Stable release following the 1.11.dev2 development series.

| Date | Change |
|---|---|
| 2018-01-19 | Update license year; mark release as V1.11 |
| 2018-01-19 | Improve Jenkins CI pipeline structure (#140) |

#### v1.11.dev2 (2017-08-03)

| Date | Change |
|---|---|
| 2017-08-03 | Pin Django and filebrowser dependency versions to fix startup issues (#12) |

#### v0.13 and earlier (up to 2014)

The earliest tagged release (`V0.13`, 2014-10-15) marked the first packaged version of the
project. The pre-0.13 history (2010–2014) covers the initial development of the CRM core,
including:

| Period | Change |
|---|---|
| 2011 | PDF export for balance sheets and profit/loss statements |
| 2011 | South database migrations for upgradeable schema changes |
| 2011 | Internationalisation (i18n) support; German and English translations |
| 2011 | Django admin customisation via Grappelli |
| 2010 | Initial CRM models: contacts, contracts, quotes, invoices, purchase orders |
| 2010 | Apache FOP-based PDF generation for commercial documents |
| 2010 | Double-entry accounting module with booking and account management |
| 2010 | Currency and rounding support; customer group price calculation |

---

## License and Legal Information

### Licensing

koalixcrm is released under the **BSD 3-Clause License**. The full license text
is located at `LICENSE` in the repository root.

| Attribute | Value |
|---|---|
| License | BSD 3-Clause License |
| Copyright holder | Aaron Riedener, Untereggen, Switzerland |
| Copyright years | 2009–2026 |
| Declared in `setup.py` | `license='BSD'` |

The three conditions of the BSD 3-Clause License are:

1. Redistributions of source code must retain the copyright notice, the list of
   conditions, and the disclaimer.
2. Redistributions in binary form must reproduce the copyright notice, the list
   of conditions, and the disclaimer in the documentation and/or other materials
   provided with the distribution.
3. Neither the name of the copyright holder nor the names of its contributors
   may be used to endorse or promote products derived from this software without
   specific prior written permission.

The software is provided "as is", without any warranty of merchantability or
fitness for a particular purpose. The copyright holder and contributors are not
liable for any damages arising from its use.

---

## Contribution Guidelines

The contribution rules are documented in
`CONTRIBUTING.md` and the associated
`CODE_OF_CONDUCT.md` at the repository root.

### Pre-contribution Discussion

Before making any change, contributors are expected to discuss the intended
change with the repository owners via a GitHub issue, email, or another
agreed channel.

### Pull Request Process

1. Remove any install or build dependencies that were added during development
   before the end of the affected Docker layer.
2. Update `README.md` to reflect interface changes — new environment variables,
   exposed ports, useful file locations, and container parameters.
3. Increment the version number in all `version.py` files to match the version
   that the pull request would represent, following the versioning scheme below.
4. Target the pull request at the `development` branch. The pull request must
   achieve at least 80 % test coverage and must not introduce issues flagged by
   the CI pipeline.
5. After review, a core developer will merge the pull request.

### Versioning Scheme

| Suffix | Stage |
|---|---|
| `1.2.0.dev1` | Development release |
| `1.2.0a1` | Alpha release |
| `1.2.0b1` | Beta release |
| `1.2.0rc1` | Release candidate |
| `1.2.0` | Final release |
| `1.2.0.post1` | Post release |

### Code Style

The project uses [Ruff](https://docs.astral.sh/ruff/) for linting and
formatting. The configuration is in `pyproject.toml`:

- Maximum line length: 119 characters (following Django's contributing guide).
- Active rule sets: `E` (pycodestyle errors), `F` (pyflakes), `W`
  (pycodestyle warnings), `I` (isort).
- Migrations are excluded from reformatting to avoid noisy diffs.

### Code of Conduct

The project follows a Code of Conduct adapted from the
[Contributor Covenant v1.4](http://contributor-covenant.org/version/1/4/).
The full text is in `CODE_OF_CONDUCT.md` at the repository root.

Key standards for participants:

- Use welcoming and inclusive language.
- Respect differing viewpoints and experiences.
- Accept constructive criticism gracefully.
- Focus on what is best for the community.
- Show empathy towards other community members.

Unacceptable behaviour includes harassment, trolling, personal attacks, and
publishing private information without consent. Instances of unacceptable
behaviour may be reported to **aaron.riedener@gmail.com**. All complaints are
kept confidential and will be reviewed and acted upon by the project team.

### No Formal Contributor License Agreement

No Contributor License Agreement (CLA) is required. There is no formal signing
requirement beyond following the pull request process and the Code of Conduct
described above.

---

## Appendix

### References

| Document | Description |
|---|---|
| [QQ_IMPORT_README.md](QQ_IMPORT_README.md) | Human-authored project introduction, feature summary, setup guides, and v1.14.0 → v2.0.0 upgrade procedure |
| [Chapter 4: Solution Strategy](../04_solution_strategy/index.md) | Architecture patterns, technology choices, and design decisions |
| [Chapter 5: Building Block View](../05_building_block_view/index.md) | Structural decomposition — high-level, mid-level, and low-level documentation of all eight apps and infrastructure packages |
| [Chapter 6: Runtime View](../06_runtime_view/index.md) | Use cases and runtime behaviour |
| [Chapter 7: Deployment View](../07_deployment_view/index.md) | CI/CD, containerisation, and infrastructure configuration |
| [Chapter 8: Cross-cutting Concepts](../08_cross_cutting_concepts/index.md) | Security, access control, configuration, entity relations, and UI architecture |
| [Chapter 9: Architecture Decisions](../09_architecture_decisions/index.md) | Recorded architectural decisions, including the v1.14.0 → v2.0.0 app split |
| [Chapter 12: Glossary](../12_glossary/QQ_SD_Glossary.md) | Consolidated glossary of domain-specific terms and acronyms |

---

Based on the [arc42](https://arc42.org) architecture documentation template, licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
