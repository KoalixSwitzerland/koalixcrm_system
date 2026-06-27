# ADR-0005: Preisgestaltung und Maßeinheiten

## Status
Accepted

## Context

Die bestehenden Modelle `ProductPrice` und `customer_group_transform` implementieren eine
zeitgebundene Preislogik mit Parteigruppen-Umrechnungsfaktoren und sind funktionsfähig. Sie
decken jedoch weder produktspezifische Einheitenkonversionen (z. B. Verkaufseinheit Stück ↔
Lagereinheit Karton à 12) noch eine explizite Preislistenstruktur für Kanal- oder
Kundensegmentierung ab. Der Katalog-Backbone ist in ADR-0003 definiert; das vorliegende ADR legt
fest, wie Preis- und Einheitenlogik auf `Product` aufsetzen.

## Decision

Die bestehenden Modelle `ProductPrice` und `customer_group_transform` bleiben unverändert und
werden von `Product` (ADR-0003, ehemals `ProductType`) referenziert. Ergänzend werden
`UnitOfMeasureConversion` für produktspezifische Einheitenumrechnungen und `PriceList` zur
expliziten Gruppierung von `ProductPrice`-Einträgen nach Kanal oder Kundensegment eingeführt.

## Why

Die Erweiterung der bestehenden, funktionsfähigen Preislogik um `UnitOfMeasureConversion` und
`PriceList` — statt einer Neuentwicklung — minimiert Migrationsaufwand und hält die bewährte
`customer_group_transform`-Semantik intakt, während Kanal-Preislisten und Einheitenkonversionen
ohne Modell-Umbau nachgerüstet werden.

## Alternatives Considered

- **Komplette Neuentwicklung der Preislogik** — abgelehnt: die bestehenden Modelle sind
  funktionsfähig; ein vollständiger Ersatz verursacht Datenmigrations- und Testaufwand ohne
  architektonischen Mehrwert.
- **Einheitenkonversionen global in `core.Unit` statt produktspezifisch** — abgelehnt: die
  Konversionsrate (z. B. Stück ↔ Karton) ist produktspezifisch und variiert je Hersteller und
  Verpackungsform; eine globale Konversionstabelle erfasst diese Varianz nicht korrekt.

## Consequences

### Positive
- Bestehende Vertrags- und Buchhaltungsmodule, die `ProductPrice` und `customer_group_transform`
  referenzieren, bleiben unverändert lauffähig.
- `PriceList` ermöglicht explizite Kanal- und Kundensegment-Preisgestaltung ohne Ad-hoc-Filterlogik
  in den API-Consumern.
- `UnitOfMeasureConversion` hält produktspezifische Umrechnungen datenbankgeführt statt
  hartkodiert.

### Negative
- `ProductPrice` und `customer_group_transform` tragen Legacy-Feldnamen; eine spätere
  Umbenennung erfordert Datenmigration.
- Die Kombination aus `ProductPrice`, `customer_group_transform` und `PriceList` ergibt drei
  Preisebenen; deren Vorrangregeln sind nachfolgend festgelegt und müssen in der API-Schicht
  implementiert werden.

---

## Drei-Ebenen-Preisvorrang

Die Preisauflösung für eine gegebene Kombination aus Produkt, Datum, Kundensegment und Einheit
folgt dieser geordneten Priorität (höchste zuerst):

1. **`ProductPrice` mit explizitem `PriceList`-FK** — kanalspezifischer oder
   kundensegmentspezifischer Preis; gilt, wenn die angeforderte `PriceList` mit der Anfrage
   übereinstimmt.
2. **`ProductPrice` ohne `PriceList`-FK** — workspace-weiter Standardpreis; gilt, wenn kein
   kanalspezifischer Preis vorhanden ist.
3. **`customer_group_transform`-Umrechnungsfaktor** — wendet den Parteigruppen-Faktor auf den
   unter 1 oder 2 ermittelten Basispreis an.

Alle drei Ebenen sind vorhanden; die Preisberechnungsfunktion ist eine reine Funktion ohne
Seiteneffekte.

---

## Preis- und Einheitenentitäten

**`ProductPrice`** (workspace-scoped, bestehend) — zeitgebundener Preis mit FK auf `Product`.

**`customer_group_transform`** (workspace-scoped, bestehend) — Umrechnungsfaktoren für
Parteigruppe, Einheit und Währung; referenziert `PartyGroup` (ADR-0001).

**`UnitOfMeasureConversion`** (workspace-scoped) — hält produktspezifische Umrechnungen
(z. B. Verkaufseinheit Stück ↔ Lagereinheit Karton à 12) mit FK auf `Product` und `core.Unit`.

**`PriceList`** (workspace-scoped) — gruppiert `ProductPrice`-Einträge nach Kanal oder
Kundensegment explizit.

---

## Workspace-Scoping-Matrix

| Entität                    | Scoping   |
|----------------------------|-----------|
| `ProductPrice`             | workspace |
| `customer_group_transform` | workspace |
| `UnitOfMeasureConversion`  | workspace |
| `PriceList`                | workspace |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend und dem geschlossenen
Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** `customer_group_transform` referenziert
`PartyGroup` (ehemals `CustomerGroup` gemäß ADR-0001).

**ADR-0002 (Admin-UI-Framework):** Preislisten und Einheitenkonversionen sind über DRF-Endpunkte
exponiert; keine direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `ProductPrice` und `UnitOfMeasureConversion` tragen
einen FK auf `Product`.

## Changelog
- 2026-05-03: Erstentscheidung. Herausgelöst aus dem vormaligen omnibus ADR-0003
  (Produkt-Katalog-Domänenmodell).
- 2026-05-03: Drei-Ebenen-Preisvorrang ratifiziert (zuvor dokumentierte Annahme in REQ-0012).
