# ADR-0006: Beschaffung und Stücklisten

## Status
Accepted

## Context

KoalixCRM unterstützt Mandanten, die Handelsware beschaffen oder Fertigprodukte herstellen.
Für beide Szenarien fehlt bislang ein strukturiertes Modell: Lieferantenverknüpfungen sind nicht
im System abgebildet, und Stücklisten für gefertigte Produkte existieren nicht. Ein separates
Lieferantenmodell würde die in ADR-0001 eingeführte `Party`-Struktur duplizieren. Der
Katalog-Backbone (ADR-0003) definiert das `kind`-Enum mit `MANUFACTURED_GOOD` als Ankerpunkt für
Stücklisten.

## Decision

Der Lieferanten-Link nutzt `Party` aus ADR-0001 mit `PartyRole.role_type = 'supplier'`; es wird
kein separates Lieferantenmodell eingeführt. `ProductSupply` verknüpft ein `Product` (ADR-0003)
mit einem Lieferanten-`Party`. `BillOfMaterials` und `BomItem` modellieren die Stückliste für
`kind = MANUFACTURED_GOOD` nach ISA-95 Part 2. Der `Routing`-Namensraum (Arbeitspläne für
Fertigungssteuerung) ist nicht Teil dieses Modells.

## Why

Die Wiederverwendung von `Party` als Lieferantenrepräsentation — statt eines neuen Modells —
vermeidet doppelte Kontaktmodellierung und hält `ProductSupply` konsistent mit dem in ADR-0001
etablierten Rollenmodell; ISA-95 Part 2 liefert eine branchenstandard-konforme Struktur für
`BillOfMaterials`/`BomItem`, ohne ein proprietäres Schema einzuführen.

## Alternatives Considered

- **Separates `Supplier`-Modell** — abgelehnt: dupliziert die in ADR-0001 bereits definierte
  Partei- und Rollenstruktur; Lieferanten sind `Party`-Einträge mit `PartyRole.role_type =
  'supplier'`.
- **Stücklisten als verschachtelte JSON-Struktur in `Product`** — abgelehnt: keine
  referentielle Integrität auf Komponentenebene, kein direkter Bezug zu Einheiten und
  Ausschusswerten, keine Unterstützung für Alternativkomponenten.

## Consequences

### Positive
- `ProductSupply.supplier` zeigt direkt auf eine `Party`; Lieferantenkontakt, Adresse und
  Bankverbindung sind ohne Join auf ein separates Modell abrufbar.
- `BomItem` unterstützt Menge, Ausschuss-Prozentsatz und Alternativkomponenten nach ISA-95 Part 2.
- Die ISA-95-Strukturierung ermöglicht einen späteren Export in MRP-/ERP-Systeme ohne
  Remodellierung.

### Negative
- `BillOfMaterials` ist nur für `kind = MANUFACTURED_GOOD` sinnvoll; die Applikationsschicht
  muss diesen Invarianten durchsetzen, da das Datenbankschema kein strukturelles Hindernis für
  andere `kind`-Werte bietet.
- Der `Routing`-Namensraum (Arbeitspläne) ist nicht modelliert; Fertigungssteuerung über
  KoalixCRM bleibt eingeschränkt, bis ein separates ADR diesen Bereich adressiert.

---

## Beschaffungs- und Fertigungsentitäten

**`ProductSupply`** (workspace-scoped) verknüpft ein `Product` (ADR-0003) mit einem
Lieferanten-`Party` (ADR-0001, `PartyRole.role_type = 'supplier'`) und trägt: Lieferanten-SKU,
Lieferzeit, Mindestbestellmenge (MOQ), Einkaufspreis.

**`BillOfMaterials`** (workspace-scoped) repräsentiert eine Stückliste für `Product` mit
`kind = MANUFACTURED_GOOD`.

**`BomItem`** (workspace-scoped) hält Komponenten-Produkt (FK auf `Product`), Menge,
Ausschuss-Prozentsatz und Alternativkomponenten. Die Strukturierung folgt ISA-95 Part 2.

---

## Workspace-Scoping-Matrix

| Entität              | Scoping   |
|----------------------|-----------|
| `ProductSupply`      | workspace |
| `BillOfMaterials`    | workspace |
| `BomItem`            | workspace |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend und dem geschlossenen
Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke.

---

## Standards-Verankerung

| Standard      | Verwendung im Modell                                         |
|---------------|--------------------------------------------------------------|
| ISA-95 Part 2 | Struktur von `BillOfMaterials` / `BomItem`                   |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** `ProductSupply.supplier` referenziert `Party`
(nicht ein separates Lieferantenmodell). `PartyRole.role_type = 'supplier'` identifiziert den
Lieferanten. `customer_group_transform` referenziert `PartyGroup` (ADR-0001).
Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus.

**ADR-0002 (Admin-UI-Framework):** Beschaffungs- und Stücklistendaten sind über DRF-Endpunkte
exponiert; keine direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `ProductSupply` und `BillOfMaterials` tragen einen FK
auf `Product`. `BillOfMaterials` gilt ausschließlich für `Product` mit `kind = MANUFACTURED_GOOD`
(ADR-0003, `kind`-Enum). `BomItem` referenziert Komponenten-`Product`-Einträge.

**ADR-0019 (Produkt-`kind`-Invarianten und Gating abhängiger Objekte):** ADR-0019 ist die
autoritäre Quelle für alle `kind`-Gating-Regeln abhängiger Entitäten einschließlich
`BillOfMaterials`. Der zulässige BOM-Geltungsbereich ist dort auf
`kind ∈ {MANUFACTURED_GOOD, KIT}` erweitert (siehe Nachtrag oben); die Durchsetzung erfolgt
über `ProductKindPolicy`.

## Nachtrag 2026-06-27 — BOM-Geltungsbereich auf KIT erweitert (siehe ADR-0019)

Die ursprüngliche Formulierung „`BillOfMaterials` gilt ausschließlich für `kind = MANUFACTURED_GOOD`"
ist durch ADR-0019 (Produkt-`kind`-Invarianten und Gating abhängiger Objekte) abgelöst; ADR-0019
erweitert den zulässigen BOM-Geltungsbereich auf `kind ∈ {MANUFACTURED_GOOD, KIT}`, weil
ADR-0014 für `ProductionOrder`-Einträge eine Stückliste auch für KIT-Produkte vorschreibt.
Der kanonische Regelsatz für `kind`-Gating aller abhängigen Objekte sowie dessen Durchsetzung
über `ProductKindPolicy` liegen vollständig in ADR-0019. Der ursprüngliche Entscheidungstext
dieses ADR bleibt unverändert als historischer Beleg erhalten.

---

## Changelog
- 2026-06-27: Nachtrag ergänzt: BOM-Geltungsbereich auf `kind ∈ {MANUFACTURED_GOOD, KIT}`
  erweitert via ADR-0019; ADR-0019 als autoritäre `kind`-Gating-Quelle in Abhängigkeiten
  eingetragen.
- 2026-05-03: Erstentscheidung. Herausgelöst aus dem vormaligen omnibus ADR-0003
  (Produkt-Katalog-Domänenmodell).
