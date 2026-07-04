# ADR-0008: Digital Product Passport — JSONB-Vorhalter

## Status
Accepted

## Context

Die EU-Verordnung über einen digitalen Produktpass (DPP) verpflichtet Hersteller und
Importeure schrittweise ab 2026 zur maschinenlesbaren Offenlegung von Produkt-Lebenszyklusdaten
(Material, Reparierbarkeit, Kohlenstoff-Fußabdruck, Recyclingfähigkeit). Die delegierten
Rechtsakte der EU für die jeweiligen Produktkategorien sind noch nicht abgeschlossen; das
konkrete JSONB-Schema ist deshalb noch nicht festlegbar. Wird die Spalte erst eingeführt, wenn
die Rechtsakte vollständig vorliegen, entsteht eine spätere Remodellierung aller betroffenen
`Product`-Einträge. Der Katalog-Backbone ist in ADR-0003 definiert.

## Decision

`ProductPassport` (workspace-scoped) hält einen JSONB-Block pro `Product` (ADR-0003) als
strukturellen Vorhalter für EU-DPP-Daten. Strukturierte DPP-Felder — sofern die delegierten
EU-Rechtsakte sie als maschinenlesbare Pflichtangaben festlegen — entstehen als Projektion
(materialisierte Ansicht oder berechneter Serialisierer) über den Ereignis-Log (ADR-0011),
die Produktstammdaten (ADR-0003) und die Klassifizierungsdaten (ADR-0004). Der JSONB-Block
ist ausschließlich für unstrukturierte regulatorische Metadaten reserviert, die kein
strukturiertes Zuhause in einem anderen Modell haben (z. B. Freitext-Chemikaliendeklarationen).
Die endgültige Struktur des JSONB-Blocks wird in einem Folge-ADR festgelegt, sobald die
delegierten EU-Rechtsakte für die relevanten Produktkategorien abgeschlossen sind.

## Why

Der JSONB-Vorhalter verhindert eine spätere Schema-Migration auf allen bestehenden
`Product`-Einträgen, wenn die EU-DPP-Anforderungen verbindlich werden; gleichzeitig erzwingt er
keine verfrühte Strukturentscheidung, bevor die Rechtsakte feststehen. Strukturierte Felder aus
dem Ereignis-Log und den Produktstammdaten als Projektion abzubilden — statt sie dupliziert in
`passport_data` zu speichern — stellt sicher, dass DPP-Exporte immer aus einer einzigen Quelle
der Wahrheit lesen und nicht mit dem tatsächlichen Ereignis-Log divergieren können.

## Alternatives Considered

- **DPP-Spalte erst einführen, wenn die EU-Rechtsakte vollständig sind** — abgelehnt: erzwingt
  eine spätere additive Schema-Migration auf allen bestehenden `Product`-Einträgen; je nach
  Datenmenge aufwändig und riskant.
- **DPP-Daten in einem separaten, vollständig strukturierten Modell von Beginn an** — abgelehnt:
  das JSONB-Schema ist noch nicht festlegbar, da die delegierten EU-Rechtsakte je Produktkategorie
  noch nicht abgeschlossen sind; ein fixes Schema jetzt würde mit hoher Wahrscheinlichkeit eine
  Remodellierung erfordern.

## Consequences

### Positive
- Keine spätere Schema-Migration auf allen `Product`-Einträgen, wenn DPP-Pflichten verbindlich
  werden.
- Der JSONB-Block ist flexibel genug, um verschiedene Kategorien-Schemata parallel aufzunehmen,
  bis ein strukturiertes ADR die endgültige Form festlegt.
- GS1 GTIN in `Product.gtin` (ADR-0003) dient bereits als globaler Produktidentifikator für
  DPP-Export und GDSN-Integration.

### Negative
- Der JSONB-Block bietet bis zur Strukturierung durch ein Folge-ADR keine referentielle
  Integrität und keine automatische Validierung.
- Abfragen auf DPP-Inhalte vor der Strukturierung erfordern JSONB-Pfad-Operatoren und sind
  nicht über B-Tree-Indizes optimierbar.
- Die Projektionslogik (welche Ereignis-Log-Felder in die DPP-Ansicht eingehen) muss
  konsistent mit dem Folge-ADR gehalten werden; eine Änderung der EPCIS-Businessstep-Semantik
  in ADR-0011 kann die Projektion beeinflussen.

---

## Compliance-Vorhalter-Entität

**`ProductPassport`** (workspace-scoped) — 1:1-FK auf `Product` (ADR-0003). Trägt einen
JSONB-Block als Vorhalter für unstrukturierte regulatorische Metadaten (z. B.
Freitext-Chemikaliendeklarationen), die kein strukturiertes Zuhause in einem anderen Modell
haben. Strukturierte DPP-Felder entstehen als Projektion über den Ereignis-Log (ADR-0011),
die Produktstammdaten (ADR-0003) und die Klassifizierungsdaten (ADR-0004); sie werden nicht
redundant in `passport_data` gespeichert. Die abschließende Festlegung, welche Felder
strukturiert und welche als JSONB-Freitext geführt werden, erfolgt in einem Folge-ADR nach
Abschluss der delegierten EU-Rechtsakte je Produktkategorie.

---

## Workspace-Scoping-Matrix

| Entität           | Scoping   |
|-------------------|-----------|
| `ProductPassport` | workspace |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend und dem geschlossenen
Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke.

---

## Standards-Verankerung

| Standard                                      | Verwendung im Modell                                                            |
|-----------------------------------------------|---------------------------------------------------------------------------------|
| EU Digital Product Passport (Verordnung 2024/1781 und delegierte Rechtsakte) | `ProductPassport` JSONB-Vorhalter; Struktur durch Folge-ADR |
| GS1 GTIN                                      | `Product.gtin` (ADR-0003) als globaler Identifikator für DPP-Export und GDSN   |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** `ProductPassport` erbt den
`WorkspaceScopedModel`-Mechanismus.

**ADR-0002 (Admin-UI-Framework):** `ProductPassport`-Daten sind über DRF-Endpunkte exponiert;
keine direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `ProductPassport` trägt einen 1:1-FK auf `Product`.
`Product.gtin` (ADR-0003) dient als globaler Identifikator für DPP-Export.

**ADR-0011 (Lager- und Lebenszyklus-Ereignis-Log):** Strukturierte DPP-Felder entstehen als
Projektion über den Ereignis-Log; `StockMovement`-Events mit Lifecycle-Businesssteps
(z. B. `commissioning`, `inspecting`, `decommissioning`) liefern die Datengrundlage.

**ADR-0015 (Geräte-Lebenszyklus-Historie):** `ProductPassport` und die Lebenszyklus-Historie
(ADR-0015) lesen aus denselben Ereignisquellen; die DPP-Projektion ist eine Teilmenge der
in ADR-0015 definierten Unit-History-Abfrage.

## Changelog
- 2026-05-03: Erstentscheidung. Herausgelöst aus dem vormaligen omnibus ADR-0003
  (Produkt-Katalog-Domänenmodell).
- 2026-05-03: `passport_data`-Semantik präzisiert: strukturierte DPP-Felder entstehen als
  Projektion über Ereignis-Log (ADR-0011) und Produktstammdaten; JSONB-Block ist auf
  unstrukturierte Freitext-Metadaten ohne strukturiertes Zuhause beschränkt.
