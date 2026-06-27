# ADR-0009: Lager-Domänen-Backbone

## Status
Accepted

## Context

KoalixCRM unterstützt Mandanten mit physischen Warenströmen: Handelsware, Fertigprodukte,
Rohmaterial und Kits. Das bestehende System trägt keinen strukturierten Lagerbestand; Produkte
aus ADR-0003 sind bislang reine Katalogobjekte ohne Standort- und Mengeninformation. Für eine
Multi-Tenant-Plattform mit 10 000+ SKUs und 10 Millionen+ Lagereinheiten werden drei
Grundprobleme gleichzeitig sichtbar: (1) Lagerhaltung erfordert eine n-stufige Standorthierarchie,
die auch automatisierte Hochregallager (AS/RS) abbildet; (2) jede physische Einheit muss einem
Produkt × Standort × Chargenkennzeichnung × Eigentümer zugeordnet sein; (3) Gebinde und Paletten
(Handling Units) müssen als eigene Entitäten mit GS1-SSCC-Identifikation existieren. Die
Produktdomäne ist in ADR-0003 (`Product`, `kind`-Enum, `ProductVariant`) vollständig definiert;
das vorliegende ADR führt ausschließlich neue Lagerentitäten ein und erweitert `Product` um ein
`tracking_mode`-Enum.

## Decision

Der Stock-Backbone besteht aus vier Kernentitäten: `Location` (n-stufige Standorthierarchie mit
`location_type`-Enum), `OnHandRecord` (Lagermengenzeile pro Produkt × Variante × Standort ×
Chargen- oder Serienreferenz × Eigentümerschaft), `HandlingUnit` (SSCC-codierbares Gebinde als
Aggregationsebene über mehrere `OnHandRecord`-Zeilen) sowie einer additiven Erweiterung von
`Product` um das `tracking_mode`-Enum (`NONE`, `BATCH`, `SERIAL`). Alle workspace-eigenen
Entitäten erben `WorkspaceScopedModel`; globale Lookup-Tabellen (Standorttypen, SSCC-Schema)
sind plattformweit gültig und nicht workspace-gebunden.

## Why

Eine n-stufige `Location`-Hierarchie mit einem einzelnen rekursiven Selbstverweis — statt
flacher Standorttabelle mit Typstring — ermöglicht beliebige Tiefe (Warehouse → Zone → Gang →
Regal → Fach → Bin) ohne Schema-Änderung und hält gleichzeitig die Integritätsregel, dass jeder
Standort genau einen übergeordneten Knoten oder keinen (Root) hat, auf Datenbankebene durch. Die
Trennung von `OnHandRecord` und `HandlingUnit` entspricht dem WMS-Kanonmuster und erlaubt, dass
eine Palette (ein `HandlingUnit`) mehrere Produkte enthält, ohne Mengendaten zu duplizieren.

## Alternatives Considered

- **Flache Standorttabelle mit Standortstring (z. B. `"WH01/ZONE-A/R03/S02/B04"`)** — abgelehnt:
  Standortnamen sind nicht normiert, Hierarchieabfragen erfordern String-Parsing, Umbenennung
  eines Zwischenknotens muss alle nachgelagerten Strings manuell aktualisieren; keine
  referentielle Integrität auf Bin-Ebene möglich.
- **Separate Tabellen je Hierarchieebene (WarehouseTable, ZoneTable, RackTable …)** — abgelehnt:
  die Anzahl Ebenen variiert je Mandant und Lagertyp; schema-feste Ebenen erfordern
  Django-Applikations-Deploys, wenn ein Mandant eine zusätzliche Ebene benötigt.
- **Einzelne polymorphe `StockItem`-Zeile mit Standortstring statt `OnHandRecord` × Standort** —
  abgelehnt: verhindert effiziente Bestandsabfragen per Standort, Charge oder Seriennummer;
  keine referentielle Integrität auf `Location`-Ebene.
- **`HandlingUnit`-Aggregation über JSONB-Array in `Location`** — abgelehnt: keine referentielle
  Integrität auf einzelne `OnHandRecord`-Zeilen; JSONB-Abfragen skalieren nicht bei Millionen
  von Lagereinheiten.

## Consequences

### Positive
- Die `Location`-Hierarchie bildet jede WMS-Topologie ab, einschließlich AS/RS-Adressen
  (mehrstufige numerische Koordinaten als `external_ref`), Hochregalfächer und Kundenlagerplätze
  (ADR-0013), ohne Schema-Änderung.
- `OnHandRecord` × `Location` × Charge/Serienreferenz erlaubt FEFO- und FIFO-Picklisten direkt
  aus der Datenbank (ADR-0012).
- `HandlingUnit` mit GS1-SSCC-Feld ermöglicht Barcode-gestütztes Wareneingangs- und
  Versandscanning ohne externe Middleware.
- Das `tracking_mode`-Enum auf `Product` steuert typsicher, welche Lagerentitäten (ADR-0012)
  angelegt werden dürfen.
- `WorkspaceScopedModel` auf allen workspace-eigenen Entitäten isoliert Lagerbestände zwischen
  Mandanten vollständig.

### Negative
- Der rekursive Selbstverweis in `Location` erfordert bei tiefer Hierarchie (5+ Ebenen) explizite
  Tiefenbegrenzung oder CTE-Abfragen auf PostgreSQL; Standard-Django-ORM reicht für
  Baumoperationen allein nicht aus (externe Bibliothek oder rekursives SQL erforderlich).
- `OnHandRecord` enthält keine aggregierten Mengen per se; die verfügbare Menge (Available to
  Promise) ergibt sich aus der Summe über mehrere Zeilen plus Reservierungen (ADR-0010); einfache
  Lagermengenabfragen ohne Aggregation liefern kein direktes Ergebnis.
- Die additive `tracking_mode`-Erweiterung auf `Product` (ADR-0003) erfordert eine
  Datenmigration für bestehende `Product`-Zeilen (Defaultwert `NONE`).

---

## Backbone-Entitäten

**`Location`** (workspace-scoped) — Ein Knoten in der n-stufigen Standorthierarchie.
Felder: `parent` (nullable FK auf sich selbst), `location_type` (Enum: `WAREHOUSE`, `ZONE`,
`AISLE`, `RACK`, `SHELF`, `LAYER`, `BIN`, `VIRTUAL`, `CUSTOMER_SITE`), `code` (mandanteneindeutige
Kurzbezeichnung), `name`, `external_ref` (optionaler AS/RS-Adressstring oder
Barcode-Klartext), `is_active`.

**`OnHandRecord`** (workspace-scoped) — Eine atomare Lagermengenzeile.
Felder: FK auf `Product` (ADR-0003), FK auf `ProductVariant` (ADR-0003, nullable), FK auf
`Location`, FK auf `Batch` (ADR-0012, nullable), FK auf `SerialUnit` (ADR-0012, nullable),
`owner_type` (Enum: `OWN`, `CUSTOMER_CONSIGNMENT`, `RENTAL`), FK auf `Party` (ADR-0001,
nullable, für Fremdbestand), `qty_on_hand` (Dezimal), `uom` (FK `core.Unit`). Der
zusammengesetzte Unique-Constraint lautet: `(workspace, product, variant, location, batch,
serial_unit, owner_type, owner_party)`.

**`HandlingUnit`** (workspace-scoped) — Ein physisches Gebinde (Palette, Karton, Tray).
Felder: `sscc` (GS1 SSCC-18-Zeichenkette, workspace-eindeutig), FK auf `parent_handling_unit`
(nullable, für Schachtelung), FK auf `Location`, `hu_type` (Enum: `PALLET`, `CARTON`, `TRAY`,
`CAGE`, `OTHER`), `is_open`. `OnHandRecord`-Zeilen tragen einen optionalen FK auf
`HandlingUnit`.

**`Product.tracking_mode`** (additives Feld auf `Product` aus ADR-0003) — Enum:
`NONE` (keine Einzel- oder Chargenverfolgung), `BATCH` (Chargentracking, ADR-0012),
`SERIAL` (Seriennummerntracking, ADR-0012). Defaultwert `NONE`; Applikationsschicht erzwingt
die Konsistenz mit den angelegten `Batch`- und `SerialUnit`-Einträgen.

---

## Workspace-Scoping-Matrix

| Entität                     | Scoping   | Begründung                                                                          |
|-----------------------------|-----------|--------------------------------------------------------------------------------------|
| `Location`                  | workspace | Lagerstruktur ist mandantenspezifisch; Kundenlagerplätze referenzieren workspace-eigene `Party` |
| `OnHandRecord`              | workspace | Lagermengen sind Mandantendaten                                                      |
| `HandlingUnit`              | workspace | SSCC-Vergabe und Gebindestruktur sind mandantenspezifisch                            |
| `Product.tracking_mode`     | (gehört zu `Product`, workspace-scoped, ADR-0003) | — |
| `LocationType`-Enum-Werte   | global    | Typen sind plattformweit stabil; kein mandantenspezifischer Bedarf; als Enum im Code |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
`tracking_mode`-Feld ist eine additive Erweiterung von `Product` (ADR-0003) und erfordert keine
Closed-Source-Abhängigkeit. Das REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend
und dem geschlossenen Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke; kein
Domänen-Code darf direkt in den Frontend-Build importiert werden.

---

## Standards-Verankerung

| Standard              | Verwendung im Modell                                                         |
|-----------------------|------------------------------------------------------------------------------|
| GS1 SSCC              | `HandlingUnit.sscc` (18-stellige Seriennummer des Versandbehälters)          |
| GS1 GTIN              | `Product.gtin` (ADR-0003) — nicht redefiniert, nur referenziert              |
| WMS-Kanonmuster       | Zone/Gang/Regal/Fach/Bin-Hierarchie; Putaway-Strategien und Pick-Logik       |
| ISO/IEC 15459         | `SerialUnit`-Identifikation (ADR-0012) verweist auf diesen Standard          |

---

## Entitätsdiagramm (konzeptionell)

```
Product (ADR-0003)          Location (workspace)
  └─ tracking_mode             └─ parent → Location (rekursiv)
  └─ ProductVariant

OnHandRecord (workspace)
  ├─ FK → Product
  ├─ FK → ProductVariant (nullable)
  ├─ FK → Location
  ├─ FK → Batch (ADR-0012, nullable)
  ├─ FK → SerialUnit (ADR-0012, nullable)
  ├─ FK → HandlingUnit (nullable)
  ├─ owner_type (OWN | CUSTOMER_CONSIGNMENT | RENTAL)
  └─ FK → Party (ADR-0001, nullable)

HandlingUnit (workspace)
  ├─ sscc (GS1 SSCC)
  ├─ FK → parent_handling_unit (nullable)
  └─ FK → Location
```

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Alle workspace-scoped Entitäten erben
`WorkspaceScopedModel`. `OnHandRecord.owner_party` referenziert `Party` für Fremdbestand und
Mietbestand (ADR-0013).

**ADR-0002 (Admin-UI-Framework):** Lagerentitäten sind über DRF-Endpunkte exponiert; keine
direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `OnHandRecord` referenziert `Product` und
`ProductVariant`. Das `tracking_mode`-Enum ist eine additive Erweiterung von `Product`:
„Das `kind`-Enum klassifiziert das Produkt als `SERVICE`, `TRADING_GOOD`, `MANUFACTURED_GOOD`,
`KIT` oder `RAW_MATERIAL`" (ADR-0003); `tracking_mode` ergänzt dieses Enum orthogonal für
die Lagerverfolgung.

**ADR-0005 (Preisgestaltung und Maßeinheiten):** `OnHandRecord.uom` referenziert `core.Unit`
analog zur Einheitenlogik in ADR-0005.

**ADR-0006 (Beschaffung und Stücklisten):** `BomItem` (ADR-0006) referenziert
Komponenten-`Product`; diese Komponenten tragen `OnHandRecord`-Zeilen (ADR-0014 regelt
die Commitment-Semantik).

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `OnHandRecord` trägt FK auf `Batch`
und `SerialUnit`.

**ADR-0013 (Miet- und Kundenstandbestand):** `OnHandRecord.owner_type = RENTAL` und
`owner_party` sind die Datenbankgrundlage für Mietflottentracking.

## Amendments

### Amendment 2026-05-04 — Expliziter `LAYER`-Enum-Wert für vierstellige Lagerhierarchie

UC-0008 (Lagerbestand per Barcode lokalisieren) und UC-0009 (Komponentenentnahme mit
Bestandsbestätigung) verwenden eine vierstellige physische Standorthierarchie: Regal → Fach →
Ebene → Position (Bin). Der bestehende `location_type`-Enum enthielt zwischen `SHELF` und `BIN`
keine maschinenlesbare Stufe für die physische Ebene innerhalb eines Faches, was die
Barcode-Auflösung und Abfragefilterung auf Ebenenebene verhinderte.

Der `location_type`-Enum wird um den Wert `LAYER` erweitert, der zwischen `SHELF` und `BIN`
eingeordnet ist. `LAYER` bezeichnet die physische Ebene innerhalb eines Fachs (Regal → Fach →
Ebene → Position). Die rekursive Eltern-Kind-Struktur von `Location` bleibt unverändert; `LAYER`
ist ausschließlich ein Typwert ohne neue Tabellenstruktur.

Der vollständige `location_type`-Enum lautet damit: `WAREHOUSE`, `ZONE`, `AISLE`, `RACK`,
`SHELF`, `LAYER`, `BIN`, `VIRTUAL`, `CUSTOMER_SITE`.

Lizenzbeschränkung: Der neue Enum-Wert lebt vollständig im Open-Source-Backend; keine
Closed-Source-Abhängigkeit entsteht.

## Changelog
- 2026-05-03: Erstentscheidung.
- 2026-05-04: `LAYER`-Enum-Wert zwischen `SHELF` und `BIN` ergänzt, um die 4-stufige physische
  Lagerhierarchie (Regal → Fach → Ebene → Position) maschinenlesbar zu verankern. UC-0008,
  UC-0009 als auslösende Use Cases. Siehe Amendment 2026-05-04.
