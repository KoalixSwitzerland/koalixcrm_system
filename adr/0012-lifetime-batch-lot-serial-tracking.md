# ADR-0012: Lebenszeit, Charge, Los und Seriennummernverfolgung

## Status
Accepted

## Context

KoalixCRM unterstützt Produktkategorien, bei denen jede physische Einheit oder jede
hergestellte Gruppe ein eigenes Lebenszeit-Profil trägt: Lebensmittel mit Mindesthaltbarkeitsdatum
(MHD), Batterien mit Produktionsdatum und Garantieablauf, Pharmazeutika mit Chargennummer und
regulatorisch vorgeschriebenem Ablaufdatum, Industriekomponenten mit Seriennummer und
Revisionshistorie. ADR-0003 definiert `Product.tracking_mode` als Enum `NONE`, `BATCH`, `SERIAL`
(eingeführt in ADR-0009); dieses ADR legt die Entitäten fest, die für `BATCH`- und
`SERIAL`-Tracking entstehen. FEFO-Kommissionierung (First-Expired-First-Out) setzt voraus, dass
das Ablaufdatum indexierbar auf der Datenbankebene gespeichert ist. Rückverfolgbarkeit
(Traceability) erfordert, dass sowohl der Vorwärtspfad (welche Endprodukte enthielten diese
Charge?) als auch der Rückwärtspfad (welche Rohstoffchargen stecken in diesem Endprodukt?) aus
dem `StockMovement`-Log (ADR-0011) traversierbar sind.

## Decision

Für `Product.tracking_mode = BATCH` wird eine `Batch`-Entität eingeführt, die Chargennummer,
Ablaufdatum, Mindesthaltbarkeitsdatum, Produktionsdatum und Lieferantencharge trägt. Für
`Product.tracking_mode = SERIAL` wird eine `SerialUnit`-Entität eingeführt, die eine eindeutige
Seriennummer, einen ISO/IEC-15459-kompatiblen eindeutigen Identifikator, Produktionsdatum,
Garantieablauf und aktuellen Betriebszustand trägt. FEFO-Pickreihenfolge basiert direkt auf dem
indizierten `expiry_date`-Feld von `Batch`; die Applikationsschicht wählt die Charge mit dem
frühesten nicht-abgelaufenen `expiry_date`. Traceability-Abfragen traversieren den
`StockMovement`-Log (ADR-0011) über FK-Verweise auf `Batch` oder `SerialUnit`.

## Why

Separate `Batch`- und `SerialUnit`-Entitäten mit eigenem Primärschlüssel — statt Chargen- und
Serieninformationen als Felder direkt auf `OnHandRecord` — ermöglichen, dass mehrere
`OnHandRecord`-Zeilen an verschiedenen Standorten auf dieselbe `Batch` verweisen (eine Charge kann
auf mehrere Standorte aufgeteilt sein), Ablaufdaten B-Tree-indizierbar sind ohne String-Parsing,
und der `StockMovement`-Log eine strukturierte FK-Verknüpfung für Traceability-Traversierung trägt
statt Text-Matching auf Chargennummern.

## Alternatives Considered

- **Chargennummer als Textfeld auf `OnHandRecord` ohne eigene Entität** — abgelehnt: kein
  Ablaufdatum in strukturierter Form speicherbar; FEFO-Sortierung erfordert String-zu-Datum-
  Konversion; keine FK-Verknüpfung im Movement-Log für strukturierte Traceability; eine Charge,
  die auf mehrere Standorte aufgeteilt ist, erscheint als mehrere nicht verbundene Datensätze.
- **Chargen- und Serieninformationen in einem gemeinsamen Polymorphmodell `TrackingUnit`** —
  abgelehnt: Chargen- und Serienattribute überschneiden sich kaum (`expiry_date` gehört zur
  Charge, `condition_state` gehört zur Serieneinheit); ein gemeinsames Modell erzeugt viele
  nullable Spalten und verliert die strukturelle Trennung zwischen los-basierter und
  einheitenbasierter Verfolgung.
- **Ablaufdatum im JSONB-Block von `ProductPassport` (ADR-0008)** — abgelehnt:
  `ProductPassport` trägt produktkatalog-bezogene Compliance-Daten; Ablaufdaten sind
  chargenspezifisch und lagerrelevant, nicht produktspezifisch; JSONB-Felder sind für
  FEFO-Sortierung nicht effizient indizierbar.

## Consequences

### Positive
- FEFO-Picklisten entstehen durch eine einfache `ORDER BY batch.expiry_date ASC`-Abfrage
  auf den `OnHandRecord`-Zeilen mit `Batch`-Join; kein Nachverarbeitungsschritt erforderlich.
- Traceability-Abfragen (Vorwärts: welche Lieferungen enthielten Charge X?) traversieren
  `StockMovement`-Zeilen per FK auf `Batch` direkt; keine Log-Parsing erforderlich.
- `SerialUnit.condition_state` (Enum: `NEW`, `USED`, `DAMAGED`, `IN_REPAIR`) ist die
  Datenbankgrundlage für Mietflotten-Zustandsverfolgung (ADR-0013).
- `Batch.supplier_lot_number` erhält die Originalchargenbezeichnung des Lieferanten für
  Rückrufverwaltung und Lieferanten-Audit.

### Negative
- Bei `tracking_mode = SERIAL` entsteht pro physischer Einheit eine `SerialUnit`-Zeile;
  für Zigtausende seriennummernpflichtiger Einheiten ist die Tabellengröße entsprechend groß.
  Dekommissionierte Einheiten werden per Soft-Delete (`decommissioned_at` gesetzt) dauerhaft
  in derselben Tabelle gehalten; ein partieller Index auf `decommissioned_at IS NULL` hält
  Hot-Path-Abfragen schnell. Keine Archivtabelle; Traceability über Jahrzehnte bleibt in
  einer einzigen Tabelle abfragbar.
- Die Applikationsschicht muss erzwingen, dass `OnHandRecord`-Zeilen für ein Produkt mit
  `tracking_mode = BATCH` immer einen `Batch`-FK tragen und für `tracking_mode = SERIAL`
  immer einen `SerialUnit`-FK; das Datenbankschema allein bietet keine strukturelle Garantie.
- Chargenübergreifende Mengenabfragen (z. B. Gesamtbestand eines Produkts über alle Chargen)
  erfordern explizite GROUP-BY-Aggregation; einfaches Lesen einer einzelnen Zeile liefert
  keine Gesamtmenge.

---

## Entitäten

**`Batch`** (workspace-scoped) — Eine Produkt-Charge oder ein Los.
Felder: FK auf `Product` (ADR-0003), `batch_number` (workspace- und produkteindeutig),
`supplier_lot_number` (nullable), `production_date` (Date, nullable), `expiry_date`
(Date, nullable), `best_before_date` (Date, nullable), `received_at` (Datetime),
`quarantine` (Boolean — wenn `true`, verhindert Kommissionierung; spiegelt sich in
`StockBalance.qty_quarantine` aus ADR-0010), `notes`. Index auf `(workspace, product, expiry_date)`
für FEFO-Abfragen.

**`SerialUnit`** (workspace-scoped) — Eine individuell verfolgte physische Einheit.
Felder: FK auf `Product` (ADR-0003), FK auf `ProductVariant` (ADR-0003, nullable), `serial_number`
(workspace- und produkteindeutig), `global_uid` (nullable — ISO/IEC-15459-kompatibler
eindeutiger Identifikator, z. B. GS1 SGTIN oder GIAI), `production_date` (Date, nullable),
`warranty_expiry` (Date, nullable), `condition_state` (Enum: `NEW`, `USED`, `DAMAGED`,
`IN_REPAIR`), FK auf `Batch` (nullable — Serieneinheit kann einer Charge angehören),
`decommissioned_at` (Datetime, nullable), `notes`.

---

## FEFO-Pickreihenfolge

Die Applikationsschicht wählt beim Pick `OnHandRecord`-Zeilen in der Reihenfolge:
1. Chargen mit `Batch.quarantine = false`,
2. aufsteigend nach `Batch.expiry_date` (nulls last),
3. bei gleichem Ablaufdatum aufsteigend nach `Batch.production_date`.

FIFO-Reihenfolge (fallback ohne Ablaufdatum) ersetzt Kriterium 2 durch aufsteigend nach
`Batch.received_at`.

---

## Aufbewahrungsuntergrenze für `SerialUnit`-Einträge

Dekommissionierte `SerialUnit`-Zeilen (`decommissioned_at` gesetzt) unterliegen der
mandantenweiten konfigurierbaren Aufbewahrungsuntergrenze (`retention_floor`). Die
Standardaufbewahrung ist „für immer" (kein automatisches Löschen). Jede künftige
Lösch-Werkzeug-Implementierung MUSS das Löschen von `SerialUnit`-Zeilen verweigern,
deren `decommissioned_at` die konfigurierte Untergrenze noch nicht unterschritten hat.
Die Untergrenze ist eine additive Schutzsperre — kein Löschauslöser; das System löscht
niemals automatisch. Sie ermöglicht branchenspezifische gesetzliche
Mindestaufbewahrungsfristen (Pharma ≥ 10 Jahre, Lebensmittel, Automotive) durch den
Mandanten als erzwingbare Untergrenze zu konfigurieren.

---

## Traceability-Traversierung

**Vorwärtspfad** (von Charge X zu Endprodukten): Alle `StockMovement`-Zeilen mit
`business_step = 'picking'` oder `TRANSFORMATION_EVENT` mit `batch = X` → referenzierte
Dokument-IDs → Lieferscheine, Aufträge.

**Rückwärtspfad** (von Endprodukt zu Rohstoffchargen): `StockMovement`-Zeilen mit
`event_type = TRANSFORMATION_EVENT`, `destination_location` enthält das Endprodukt → alle
Quell-`batch`-FKs desselben Transformations-Events.

---

## Workspace-Scoping-Matrix

| Entität      | Scoping   | Begründung                                                                    |
|--------------|-----------|-------------------------------------------------------------------------------|
| `Batch`      | workspace | Chargen sind mandantenspezifische Produktionsdaten                            |
| `SerialUnit` | workspace | Serieneinheiten sind mandantenspezifische Anlagendaten                        |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. ISO/IEC 15459 ist ein öffentlich zugänglicher Standard; die
Nutzung des `global_uid`-Felds zur Speicherung von ISO/IEC-15459-konformen Identifikatoren
erfordert keine kommerzielle Lizenz. Das REST-API-Integrationsprotokoll (ADR-0002) bleibt die
einzige Kommunikationsbrücke zum Frontend.

---

## Standards-Verankerung

| Standard          | Verwendung im Modell                                                           |
|-------------------|--------------------------------------------------------------------------------|
| GS1 GTIN          | `Product.gtin` (ADR-0003) — Chargen und Serien referenzieren `Product`        |
| ISO/IEC 15459     | `SerialUnit.global_uid` als eindeutiger Einheitenidentifikator                 |
| FEFO / FIFO       | Pick-Reihenfolge basiert auf `Batch.expiry_date` bzw. `Batch.received_at`      |
| GS1 EPCIS 2.0     | Traceability-Traversierung nutzt `StockMovement`-Log (ADR-0011)                |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben
`WorkspaceScopedModel`.

**ADR-0003 (Produkt-Katalog-Backbone):** `Batch` und `SerialUnit` tragen FK auf `Product`.
Das `tracking_mode`-Enum (eingeführt in ADR-0009 als additive Erweiterung von `Product` aus
ADR-0003) steuert, ob `Batch`- oder `SerialUnit`-Einträge für ein Produkt angelegt werden.

**ADR-0009 (Lager-Domänen-Backbone):** `OnHandRecord` trägt FK auf `Batch` und `SerialUnit`.
`Product.tracking_mode` (ADR-0009) regelt die Verwendung dieser Entitäten.

**ADR-0010 (Lagerbestandszustände und Reservierungen):** `Batch.quarantine = true` spiegelt
sich in `StockBalance.qty_quarantine`; `StockReservation` trägt optionalen FK auf `Batch`.

**ADR-0011 (Lagerbewegungen und Ereignis-Log):** `StockMovement` trägt FK auf `Batch` und
`SerialUnit`; Traceability-Traversierung nutzt den Movement-Log.

**ADR-0013 (Miet- und Kundenstandbestand):** `SerialUnit.condition_state` ist die
Datenbankgrundlage für Zustandsverfolgung in der Mietflotte.

**ADR-0015 (Geräte-Lebenszyklus-Historie):** `SerialUnit` ist das Datenobjekt, dessen
vollständige Lebenszyklus-Historie ADR-0015 definiert. Die Soft-Delete-Strategie
(`decommissioned_at`) und die Aufbewahrungsuntergrenze ermöglichen die in ADR-0015 geforderte
jahrzehntlange Rückverfolgbarkeit.

## Changelog
- 2026-05-03: Erstentscheidung.
- 2026-05-03: OQ-0006 geschlossen: Dekommissionierte `SerialUnit`-Zeilen werden dauerhaft
  per Soft-Delete gehalten; keine Archivtabelle; partieller Index auf `decommissioned_at IS NULL`
  für Hot-Path-Performance. Workspace-weite konfigurierbare Aufbewahrungsuntergrenze
  (`retention_floor`) als additive Schutzsperre eingeführt.
