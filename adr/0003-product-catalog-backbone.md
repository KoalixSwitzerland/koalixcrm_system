# ADR-0003: Produkt-Katalog-Backbone

## Status
Accepted

## Context

`koalixcrm.products` enthält heute vier Modelle: `ProductType` (Stammdaten eines Katalogeintrags
mit Titel, interner Nummer, Einheit und Steuer), `Product` (eine semantisch leere
Identifier-Hülle mit FK auf `ProductType`), `ProductPrice` (zeitgebundener Preis) und
`customer_group_transform` (Umrechnungsfaktoren für Parteigruppe, Einheit und Währung). Für eine
Multi-Tenant-Plattform mit 10 000+ SKUs über mehrere Branchen (Dienstleistungen, Handelsware,
Fertigung) fehlen Varianten, Mehrsprachigkeit, Medien und ein strukturiertes Lifecycle-Management.
Das bestehende `ProductType`/`Product`-Splitmodell löst dieses Problem nicht, da `Product` keinen
eigenen semantischen Inhalt trägt.

## Decision

`ProductType` wird zu `Product` umbenannt und übernimmt die Rolle des kanonischen Katalogobjekts.
Das bisherige leere `Product`-Modell entfällt; seine Identifikationsfunktion geht auf
`ProductVariant` über. Ergänzend werden `ProductFamily`, `ProductTranslation` und `ProductMedia`
eingeführt. Das `kind`-Enum klassifiziert das Produkt als `SERVICE`, `TRADING_GOOD`,
`MANUFACTURED_GOOD`, `KIT` oder `RAW_MATERIAL`.

## Why

Nur ein einziges, klar benanntes Stammdatenobjekt (`Product`) mit einem maschinenlesbaren
`kind`-Enum ermöglicht es, Klassifizierung (ADR-0004), Preislogik (ADR-0005), Stücklisten
(ADR-0006) und Dienstleistungsprofil (ADR-0007) typsicher am selben Ankerpunkt zu befestigen,
ohne branchenspezifische Unterklassen oder separate Modelle pro Produktkategorie einzuführen.

## Alternatives Considered

- **Je eine `Product`-Unterklasse pro Branche oder Produktkategorie** — abgelehnt: erfordert
  Schema-Änderungen und Code-Deploys pro Branche, skaliert nicht in einer Multi-Tenant-Umgebung,
  verletzt den Grundsatz, dass das Backend als PyPI-Wheel installierbar bleibt und keinen
  Quantalq-eigenen Inhalt enthält.
- **Beibehaltung des `ProductType`/`Product`-Splitmodells** — abgelehnt: `Product` als leere
  Hülle ohne eigene Bedeutung erhöht den Entwickler-Overhead und verdeckt die eigentliche
  Domänensemantik.

## Consequences

### Positive
- Ein einziges kanonisches Katalogobjekt (`Product`) als Ankerpunkt für alle nachgelagerten ADRs
  (ADR-0004 bis ADR-0008).
- Das `kind`-Enum erlaubt typsichere Verzweigungen in Klassifizierung, Stücklisten und
  Dienstleistungsprofil, ohne polymorphe Modellhierarchien einzuführen.
- `ProductVariant` übernimmt die SKU-Identifikation sauber; `ProductFamily` gruppiert
  Varianten einer Linie.
- Mehrsprachigkeit (`ProductTranslation`) und Medien (`ProductMedia`) sind als separate,
  schlanke Tabellen von den Stammdaten entkoppelt.

### Negative
- Die Umbenennung `ProductType` → `Product` erfordert eine Datenmigration und bricht bestehende
  FK-Referenzen aus `contracts` und `accounting` auf die `crm_producttype`-Tabelle (siehe
  OQ-0002).
- Das neue Modell ist umfangreicher als das bisherige; Onboarding neuer Entwickler erfordert
  Kenntnis der Backbone-Entitäten und ihrer Beziehungen.

---

## Backbone-Entitäten

**`Product`** (workspace-scoped) trägt: `sku`, `gtin` (GS1 GTIN), `mpn`
(Hersteller-Artikelnummer), `kind`-Enum `{SERVICE, TRADING_GOOD, MANUFACTURED_GOOD, KIT,
RAW_MATERIAL}`, `lifecycle_status`, `base_uom` (FK `core.Unit`), `tax_class` (FK `core.Tax`),
`brand` sowie Audit-Felder.

**`ProductFamily`** (workspace-scoped) gruppiert Varianten einer Produktlinie.

**`ProductVariant`** (workspace-scoped) hält eine Zeile pro tatsächlicher SKU mit FK auf
`ProductFamily` und Varianten-Achsen-Werten (z. B. Farbe × Größe).

**`ProductTranslation`** (workspace-scoped) speichert mehrsprachige Bezeichnung,
Kurzbeschreibung und Langbeschreibung je Sprach-Code.

**`ProductMedia`** (workspace-scoped) verwaltet Bilder, Datenblätter und Zertifikate als
Verweise auf den S3/MinIO-Objektspeicher. Der `media_type`-Enum kennt genau die Werte
`image`, `datasheet` und `certificate`; andere Werte werden vom System abgewiesen.

---

## Lifecycle-Status-Werte

`Product.lifecycle_status` kennt genau die Werte `DRAFT`, `ACTIVE`, `DISCONTINUED`, `ARCHIVED`
und `EXTERNAL_ONLY`. Die erlaubten Übergänge sind:

- `DRAFT` → `ACTIVE`
- `DRAFT` → `EXTERNAL_ONLY`
- `ACTIVE` → `DISCONTINUED`
- `DISCONTINUED` → `ARCHIVED`

Die Übergänge `ARCHIVED` → `ACTIVE`, `DISCONTINUED` → `DRAFT` und `EXTERNAL_ONLY` → jeder
andere Wert sind nicht erlaubt.

`EXTERNAL_ONLY` bezeichnet Produkte, die von einer Werkstatt oder einem Servicebetrieb
gewartet oder repariert werden, ohne dass das Produkt im eigenen Katalog verkauft oder gelagert
wird. Ein typisches Beispiel ist ein Einzelmodell, das ein Kunde zur Reparatur einliefert und
das ausschließlich als Fremdeinheit (`SerialUnit` mit `Product.lifecycle_status = EXTERNAL_ONLY`)
verwaltet wird. Der `lifecycle_status = EXTERNAL_ONLY`-Wert wahrt die Invariante, dass jede
`SerialUnit` einen `Product`-FK trägt, ohne einen vollständigen Katalogstammsatz zu erzwingen.

---

## Übersetzungs-Fallback-Kette

Fordert ein API-Consumer eine Produktbezeichnung in einer Sprache an, für die keine
`ProductTranslation` existiert, wendet das System die folgende dreistufige Kette an:

1. Übersetzung der workspace-weit konfigurierten Fallback-Sprache.
2. Erste vorhandene Übersetzung des Produkts (beliebige Sprache, deterministisch nach
   `language_code` geordnet).
3. Leerer String für alle Textfelder; das Produkt-Objekt selbst wird nicht unterdrückt.

---

## Workspace-Scoping-Matrix

| Entität              | Scoping   |
|----------------------|-----------|
| `Product`            | workspace |
| `ProductFamily`      | workspace |
| `ProductVariant`     | workspace |
| `ProductTranslation` | workspace |
| `ProductMedia`       | workspace |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend und dem geschlossenen
Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke; kein Domänen-Code darf
direkt in den Frontend-Build importiert werden.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben den in ADR-0001
definierten `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus.

**ADR-0002 (Admin-UI-Framework):** Das Next.js-Frontend greift ausschließlich über DRF-Endpunkte
auf Backbone-Entitäten zu; keine direkte Python-Modell-Referenz im Frontend.

**ADR-0004 (Klassifizierung & Attribute)** baut auf `Product` und dem `kind`-Enum auf.

**ADR-0005 (Preise & Einheiten)** referenziert `Product` als Ankerpunkt für `ProductPrice` und
`UnitOfMeasureConversion`.

**ADR-0006 (Beschaffung & Stücklisten)** nutzt `Product` mit `kind = MANUFACTURED_GOOD` als
Ausgangspunkt für `BillOfMaterials`.

**ADR-0007 (Dienstleistungsprofil)** verknüpft `ServiceProfile` 1:1 mit `Product` wo
`kind = SERVICE`.

**ADR-0008 (Digital Product Passport)** hängt `ProductPassport` per 1:1-FK an `Product`.

## Changelog
- 2026-05-03: Erstentscheidung. Herausgelöst aus dem vormaligen omnibus ADR-0003
  (Produkt-Katalog-Domänenmodell).
- 2026-05-03: `lifecycle_status`-Enum-Werte und erlaubte Übergänge ratifiziert
  (zuvor dokumentierte Annahme in REQ-0005). `media_type`-Enum-Werte ratifiziert
  (zuvor dokumentierte Annahme in REQ-0004). Übersetzungs-Fallback-Kette ratifiziert
  (zuvor dokumentierte Annahme in REQ-0003).
- 2026-05-03: `lifecycle_status = EXTERNAL_ONLY` als fünfter Enum-Wert ergänzt. Ermöglicht
  Reparatur-/Servicebetrieb für Fremdeinheiten ohne vollständigen Katalogstammsatz; wahrt die
  Invariante, dass jede `SerialUnit` einen `Product`-FK trägt (ADR-0012, ADR-0015).
- 2026-05-05: Amendment — Layer-1-Felder des Minimal Core Product ergänzt (siehe unten).

---

## Amendment 2026-05-05 — Layer-1-Felder des Minimal Core Product

Zur Ergänzung der `Product`-Backbone-Entitäten-Definition: Die folgenden Felder sind direkte
Spalten auf `Product` (Layer 1 im Schichtenmodell nach ADR-0004 Amendment 2026-05-05 und
ADR-0018). Sie sind stets vorhanden, lizenzfrei und unabhängig von jeder
Layer-3-Klassifizierung.

| Feld                  | Typ                     | Pflicht | Hinweis                                                              |
|-----------------------|-------------------------|---------|----------------------------------------------------------------------|
| `weight_kg`           | decimal (≥ 0)           | nein    | Physisches Bruttogewicht; steuert kanonischen Schlüssel `koalix.weight_kg` (ADR-0018) |
| `dimensions_length_m` | decimal (≥ 0)           | nein    | Länge in Metern                                                      |
| `dimensions_width_m`  | decimal (≥ 0)           | nein    | Breite in Metern                                                     |
| `dimensions_height_m` | decimal (≥ 0)           | nein    | Höhe in Metern                                                       |
| `manufacturer_party`  | FK `core.Party` (null.) | nein    | Hersteller; verweist auf Party-Modell (ADR-0001)                     |
| `country_of_origin`   | string ISO 3166-1 α-2   | nein    | Ursprungsland; steuert kanonischen Schlüssel `koalix.country_of_origin` (ADR-0018) |

Die bereits in der ursprünglichen Backbone-Definition gelisteten Felder `sku`, `gtin`, `mpn`,
`kind`, `lifecycle_status`, `base_uom`, `tax_class` und `brand` sowie mehrsprachige
Bezeichnung und Beschreibung über `ProductTranslation` sind ebenfalls Layer-1-Felder.

Kanonische Schlüssel (ADR-0018), die aus Layer-1-Feldern abgeleitet werden (`koalix.weight_kg`,
`koalix.country_of_origin`), lesen ihren Wert aus der direkten `Product`-Spalte; ein
separater EAV-Eintrag ist nicht erforderlich.
