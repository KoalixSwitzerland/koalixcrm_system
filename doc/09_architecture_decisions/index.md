# Architecture Decisions

This chapter documents significant architecture decisions made during the development of koalixcrm.
Two human-authored decision documents have been imported from the repository and are the primary
authoritative sources for this chapter.

## Imported decision documents

### Monolith-to-apps split (v1.14.0 to v2.0.0)

The document [QQ_IMPORT_docs-migration-v1-14-0-to-v2-0-0.md](QQ_IMPORT_docs-migration-v1-14-0-to-v2-0-0.md)
describes the architectural decision to decompose the legacy monolithic `crm` Django app into
focused, independently installable apps. The key decisions recorded there are:

- **Preserve `db_table` names across the split.** Every model keeps its legacy `crm_*` table name
  so the underlying SQL schema is untouched by the refactoring. This allows data migrations and
  fresh installs to use the same code path.
- **Idempotent migration operations (`CreateModelIfNotExists` / `AddFieldIfNotExists`).** Custom
  migration operations (in `koalixcrm/migration_utils.py`) are no-ops when the target table or
  column already exists, making the same migration file safe on both legacy and fresh databases.
- **`sync_split_migrations` management command.** Rather than requiring operators to run
  `migrate --fake`, a dedicated command reconciles the `django_migrations` table of legacy
  deployments against the new cross-app migration graph before the first `migrate` run. Both
  container entrypoints invoke this command unconditionally.
- **Three-phase data migration for the Party data model.** The restructure of `Contact`,
  `Customer`, `Supplier`, and `Person` into a unified Party model follows a create → backfill →
  verify sequence, where the verify step aborts the migration if data-integrity invariants fail,
  ensuring the destructive drop step never runs on an inconsistent database.
- **XSLT template migration.** The switch from Django's `serializers.serialize('xml', ...)` to a
  hand-rolled `<koalixcrm-export>` XML tree (produced by `XmlAggregator` in the Java
  `pdf-export-service`) requires mechanical rewriting of all XSL templates. Sixteen rewrite rules
  are documented in the source document.

### CommercialDocument and Address field changes

The document [QQ_IMPORT_docs-migration-commercial-document-and-address-fields.md](QQ_IMPORT_docs-migration-commercial-document-and-address-fields.md)
records the decisions behind a schema + data migration affecting two models:

- **Rename `external_reference` to `party_reference`** on `CommercialDocument` and add a new
  `ext_business_appl_references` JSONField for storing references to external business
  applications (e.g. Bexio, Comatic). The rename is a pure `RenameField`; the JSON field starts
  empty for existing rows.
- **Restructure `Address` from unstructured address lines to `street` + `number` + three
  additional lines.** The split heuristic is language-conditional: European locales use a trailing
  digit-token rule; English locales use a leading digit-token rule. Where no rule matches, the
  entire original `address_line_1` goes into `street` and `number` is left blank — no data is
  discarded.
- **Three-file migration pattern.** The destructive `RemoveField` step is kept in its own
  migration file, separated from the additive and data-migration steps, so deployments can stop
  and verify the split at an intermediate state before committing to the irreversible drop.

## System-Design ADRs (geplant)

The following architecture decisions define the domain model and data schema for the system-design
phase of KoalixCRM (planning, product catalog, stock management, and related business domains).

- [ADR-0001 — Contact and Party Data Model](0001-contact-and-party-data-model.md)
- [ADR-0002: Replacement of Django Admin with a Next.js-based Admin UI](0002-admin-ui-framework.md)
- [ADR-0003: Produkt-Katalog-Backbone](0003-product-catalog-backbone.md)
- [ADR-0004: Klassifizierung und erweiterbare Attribute (getypte EAV-Tabellen)](0004-classification-and-extensible-attributes.md)
- [ADR-0005: Preisgestaltung und Maßeinheiten](0005-pricing-units-of-measure.md)
- [ADR-0006: Beschaffung und Stücklisten](0006-sourcing-and-bill-of-materials.md)
- [ADR-0007: Dienstleistungsprofil](0007-service-profile.md)
- [ADR-0008: Digital Product Passport — JSONB-Vorhalter](0008-digital-product-passport-placeholder.md)
- [ADR-0009: Lager-Domänen-Backbone](0009-stock-domain-backbone.md)
- [ADR-0010: Lagerbestandszustände und Reservierungen](0010-stock-states-and-reservations.md)
- [ADR-0011: Lager- und Lebenszyklus-Ereignis-Log](0011-stock-movements-and-event-log.md)
- [ADR-0012: Lebenszeit, Charge, Los und Seriennummernverfolgung](0012-lifetime-batch-lot-serial-tracking.md)
- [ADR-0013: Miet- und Kundengeführter Bestand](0013-customer-held-rental-stock.md)
- [ADR-0014: Montage/Kitting und geteilter Bestand](0014-assembly-kitting-and-split-stock.md)
- [ADR-0015: Geräte-Lebenszyklus-Historie](0015-unit-lifecycle-history.md)
- [ADR-0016: Identifier-Registry und Barcode-Auflösung](0016-identifier-registry-and-barcode-resolution.md)
- [ADR-0017: GoodsReceipt als Prozess-Aggregat](0017-goods-receipt-as-process-aggregate.md)
- [ADR-0018: Kanonisches Produktattribut-Vokabular](0018-canonical-product-attribute-vocabulary.md)
- [ADR-0019: Produkt-`kind`-Invarianten und Gating abhängiger Objekte](0019-product-kind-invariants.md)
- [ADR-0020: Gemeinsame deklarative Attributvalidierung zwischen Backend und Frontend](0020-shared-declarative-attribute-validation.md)
- [ADR-0021: Produkt-Variantengranularität — Topologie, Schlüsselung und Attribut-Vererbungskaskade](0021-produkt-variantengranularitaet-topologie-schluesselung-attributkaskade.md)
- [Architektur-Übersicht KoalixCRM](architecture_overview.md)

## Further decisions

Additional architecture decisions not covered by the imported documents above should be added to
this chapter manually (for example as ADR files in `doc/09_architecture_decisions/`).
