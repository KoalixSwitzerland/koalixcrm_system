# ADR-0007: Dienstleistungsprofil

## Status
Accepted

## Context

KoalixCRM unterstützt Mandanten, die neben Waren auch Dienstleistungen verkaufen und abrechnen.
Dienstleistungen unterscheiden sich strukturell von physischen Produkten: Sie haben kein Gewicht,
keinen Lagerbestand und keine Stückliste, wohl aber ein Abrechnungsmodell (Festpreis, Stundensatz,
Abonnement, Staffelpreis), eine Standarddauer, eine Leistungsbeschreibung und eine
SLA-Referenz. Der Katalog-Backbone (ADR-0003) definiert `kind = SERVICE` als Ankerpunkt für
dienstleistungsspezifische Metadaten.

## Decision

`ServiceProfile` steht in einer 1:1-Beziehung zu `Product` (ADR-0003) und gilt ausschließlich
für Einträge mit `kind = SERVICE`. `ServiceProfile` trägt: Abrechnungsmodell
(`fixed | hourly | subscription | tiered`), Standarddauer, Leistungsbeschreibung und
SLA-Referenz. Kein separates Dienstleistungsmodell außerhalb von `Product` wird eingeführt.

## Why

Eine 1:1-Erweiterung via `ServiceProfile` — statt einer Dienstleistungs-Unterklasse von `Product`
oder einem vollständig separaten Modell — hält die Preislogik (ADR-0005), Klassifizierung
(ADR-0004) und Backbone-Identifikatoren (ADR-0003) für Dienstleistungen und Waren einheitlich
anwendbar, ohne Dienstleistungen in den gemeinsamen Listenabfragen strukturell zu bevorzugen oder
zu benachteiligen.

## Alternatives Considered

- **Separate `Service`-Modellhierarchie neben `Product`** — abgelehnt: dupliziert
  Preislogik, Klassifizierung und Identifikatoren; Dienstleistungen und Waren erscheinen nicht
  in einer einheitlichen Katalogliste.
- **Dienstleistungsfelder direkt auf `Product` als nullable Spalten** — abgelehnt: nullable
  Felder ohne strukturelle Beschränkung auf `kind = SERVICE` sind semantisch mehrdeutig und
  erzwingen kein konsistentes Pflichtfeldflag für Dienstleistungen.

## Consequences

### Positive
- Dienstleistungen nutzen dieselben Preislisten (ADR-0005), dasselbe Klassifizierungssystem
  (ADR-0004) und dieselben Backbone-Identifikatoren (ADR-0003) wie physische Produkte.
- `ServiceProfile` ist schmal und enthält ausschließlich dienstleistungsspezifische Metadaten;
  der Backbone bleibt unbelastet.
- Das Abrechnungsmodell-Enum (`fixed | hourly | subscription | tiered`) ist erweiterbar, ohne
  die Backbone-Struktur zu ändern.

### Negative
- Die Applikationsschicht muss sicherstellen, dass `ServiceProfile`-Einträge ausschließlich
  für `Product` mit `kind = SERVICE` angelegt werden; das Datenbankschema bietet dafür kein
  strukturelles Hindernis.
- `ServiceProfile` deckt keine komplexe Projektstrukturierung (Phasen, Meilensteine, Ressourcen)
  ab; diese Anforderungen erfordern ein separates ADR.

---

## Dienstleistungsentitäten

**`ServiceProfile`** (workspace-scoped) — 1:1-FK auf `Product` (ADR-0003, `kind = SERVICE`).
Felder: Abrechnungsmodell (`fixed | hourly | subscription | tiered`), Standarddauer,
Leistungsbeschreibung, SLA-Referenz.

---

## Workspace-Scoping-Matrix

| Entität          | Scoping   |
|------------------|-----------|
| `ServiceProfile` | workspace |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend und dem geschlossenen
Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** `ServiceProfile` erbt den
`WorkspaceScopedModel`-Mechanismus.

**ADR-0002 (Admin-UI-Framework):** `ServiceProfile`-Daten sind über DRF-Endpunkte exponiert;
keine direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `ServiceProfile` trägt einen 1:1-FK auf `Product` und
gilt ausschließlich für `Product` mit `kind = SERVICE` (ADR-0003, `kind`-Enum).

**ADR-0004 (Klassifizierung & Attribute):** Dienstleistungen sind als `Product` klassifiziert
und nutzen dasselbe `AttributeSet`-System.

**ADR-0005 (Preise & Einheiten):** `ProductPrice` und `PriceList` gelten für Dienstleistungen
identisch wie für Waren.

## Changelog
- 2026-05-03: Erstentscheidung. Herausgelöst aus dem vormaligen omnibus ADR-0003
  (Produkt-Katalog-Domänenmodell).
