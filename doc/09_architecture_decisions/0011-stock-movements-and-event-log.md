# ADR-0011: Lager- und Lebenszyklus-Ereignis-Log

## Status
Accepted

## Context

`OnHandRecord` (ADR-0009) und `StockBalance` (ADR-0010) beschreiben den aktuellen Bestand.
Jede Änderung an diesen Werten — Wareneingang, Einlagerung, Kommissionierung, Versand,
Transfer, Schwund, Retoure, Mietausgabe, Mietrückgabe — muss lückenlos nachvollziehbar sein.
Darüber hinaus trägt jede individuell verfolgte Einheit (`SerialUnit`, ADR-0012) einen
vollständigen Lebenszyklus: Inbetriebnahme, Inspektion, Reparatur, Firmware-Aktualisierung,
Außerbetriebnahme. Diese Lebenszyklus-Berührungen verändern keine Menge, müssen aber
denselben unveränderlichen Event-Log nutzen, damit die vollständige Einheitenhistorie aus
einer einzigen Quelle der Wahrheit abfragbar ist. Regulatorische Anforderungen
(Lebensmittelsicherheit, Pharmalieferkette, DSGVO-Audit, ISO-Kalibriernachweis) und
operative Anforderungen (Differenzklärung, Inventurabgleich, Servicehistorie) verlangen
einen unveränderlichen Ereignis-Log. GS1 EPCIS 2.0 (Electronic Product Code Information
Services) definiert ein branchenweites Ereignismodell für genau diesen Anwendungsfall;
seine Core Business Vocabulary (CBV) spezifiziert kanonische Ereignistypen und
Businessstep-Codes — einschließlich mengenloser Lifecycle-Events. Das Backend muss als
PyPI-Wheel installierbar bleiben; eine vollständige EPCIS-Repository-Implementierung ist
deshalb nicht Gegenstand dieses ADR, aber das interne Ereignismodell orientiert sich an der
EPCIS-2.0-Struktur, um einen späteren Export zu ermöglichen.

## Decision

Jede Lagerbewegung und jeder Lebenszyklus-Touch einer Einheit wird als unveränderlicher
`StockMovement`-Datensatz gespeichert. Zeilen werden nach dem Schreiben nicht mehr geändert
oder gelöscht; Korrekturen erfolgen durch kompensierende Gegenbuchungen mit eigenem
`StockMovement`-Datensatz. Das `event_type`-Enum orientiert sich an GS1 EPCIS 2.0 CBV:
`OBJECT_EVENT`, `AGGREGATION_EVENT`, `TRANSACTION_EVENT`, `TRANSFORMATION_EVENT`,
`ASSOCIATION_EVENT`. Ein ergänzendes `business_step`-Feld trägt den EPCIS-CBV-Businessstep-Code;
der gültige Wertebereich umfasst: `shipping`, `receiving`, `packing`, `picking`, `accepting`,
`inspecting`, `scrapping`, `returning`, `rental_out`, `rental_return`, `adjustment`,
`commissioning`, `installing`, `removing`, `repairing`, `decommissioning`, `inventorying`. Das Feld `qty`
ist nullable: ein Ereignis mit `qty = null` ist ein gültiges Lebenszyklus-Event (z. B. eine
Inspektion, die einen Messwert aufzeichnet, ohne eine Menge zu verschieben); ein Ereignis mit
`qty != null` ist eine Mengenbewegung. `StockMovement`-Events mit `qty != null` aktualisieren
`StockBalance`-Felder (ADR-0010) synchron im selben Datenbank-Transaktion; asynchrone
Aktualisierung via Celery wird abgelehnt, da ATP-Korrektheit finanziell tragend ist und
Überverkäufe nicht tolerierbar sind. PostgreSQL verarbeitet dieses Muster bei korrekter
Indexierung auf `(workspace, product, location)` mit tausenden Transaktionen pro Sekunde.
`OnHandRecord`-Zeilen werden durch Mengenbewegungen erstellt, aktualisiert oder auf null
gesetzt; Lebenszyklus-Events ohne Menge berühren `OnHandRecord` nicht. Jeder
`StockMovement`-Datensatz trägt einen `idempotency_key`, der Doppelbuchungen durch
Netzwerk-Retries verhindert.

Der Event-Log unterliegt einer mandantenweiten konfigurierbaren **Aufbewahrungsuntergrenze**
(`retention_floor`) pro Objektklasse. Die Standardaufbewahrung ist „für immer" (kein
automatisches Löschen). Jede künftige Lösch-Werkzeug-Implementierung MUSS das Löschen von
`StockMovement`-Zeilen verweigern, deren `occurred_at` die konfigurierte Untergrenze noch nicht
unterschritten hat. Die Untergrenze ist eine additive Schutzsperre — kein Löschauslöser;
das System löscht niemals automatisch. Sie ermöglicht branchenspezifische gesetzliche
Mindestaufbewahrungsfristen (Pharma ≥ 10 Jahre, Lebensmittel, Automotive) als erzwingbare
Untergrenze durch den Mandanten zu konfigurieren.

## Why

Unveränderliche Ereigniszeilen mit kompensierenden Gegenbuchungen — statt Mutationen auf
`OnHandRecord` — gewährleisten, dass der vollständige Buchungsweg jederzeit aus dem
`StockMovement`-Log rekonstruierbar ist, ohne Backup-Wiederherstellungen oder Audit-Trigger
auf Mutable-Tabellen zu benötigen. Mengenlose Lebenszyklus-Events in derselben Tabelle zu
führen — statt in einer separaten Lebenszyklus-Ereignistabelle — erhält die vollständige
Einheitenhistorie in einer einzigen Abfrageebene und verhindert das Auseinanderdriften zweier
Zeithorizonte (ADR-0015). Die EPCIS-2.0-orientierte Struktur ermöglicht einen späteren Export
in externe EPCIS-Repositories (Handelspartner, Regulatoren) ohne Remodellierung; EPCIS 2.0
unterstützt mengenlose Lifecycle-Events nativ.

## Alternatives Considered

- **Mutable `OnHandRecord`-Zeilen ohne Event-Log** — abgelehnt: Bestandsänderungen sind
  nicht nachvollziehbar; Differenzklärung erfordert manuelle Snapshot-Vergleiche;
  regulatorische Audit-Anforderungen (Pharma, Lebensmittel) sind nicht erfüllbar.
- **Audit-Trigger auf der Datenbank-Ebene (z. B. PostgreSQL `BEFORE UPDATE`-Trigger, der
  alte Zeilen in eine Shadow-Tabelle kopiert)** — abgelehnt: Trigger sind datenbankspezifisch
  (PostgreSQL only), schwer testbar, erzeugen kein strukturiertes Ereignisobjekt mit
  Geschäftssemantik (Businessstep, Grund, Belegbezug) und sind im PyPI-Paket nicht portabel.
- **Vollständige EPCIS-2.0-Repository-Implementierung mit allen EPCIS-XML/JSON-LD-Endpunkten** —
  abgelehnt: erhöht die Implementierungs- und Testkomplexität stark; das Backend muss als
  PyPI-Wheel für alle KoalixCRM-Installationen lauffähig sein, nicht als spezialisiertes
  EPCIS-Repository; ein EPCIS-Export-Endpunkt kann als separates ADR nachgerüstet werden.
- **Ereignistypen als freier String statt Enum** — abgelehnt: typsichere Enum-Werte
  ermöglichen Abfragen und Trigger ohne String-Matching; freie Strings öffnen den Log für
  Inkonsistenzen.

## Consequences

### Positive
- Der `StockMovement`-Log ist das unveränderliche Fundament für Differenzklärung,
  Inventurprotokoll, Rückverfolgbarkeit (ADR-0012) und Renditeverfolgung (ADR-0013).
- EPCIS-orientierte Feldnamen (`event_type`, `business_step`, `read_point`, `biz_location`)
  erlauben einen EPCIS-2.0-Export ohne Remodellierung.
- Der `idempotency_key` verhindert Doppelbuchungen bei Netzwerk-Retries; der Log bleibt
  konsistent, auch wenn Clients einen Event mehrfach senden.
- Kompensierende Gegenbuchungen sind selbst Events im Log und deshalb vollständig auditierbar.

### Negative
- Der Log wächst monoton. PostgreSQL declarative Range-Partitionierung nach `occurred_at`
  (monatliche Partitionen, eingerichtet vor dem ersten Datenlauf) hält Abfragen performant.
  Archivierung älterer Partitionen in Cold Storage ist eine separate operative Policy;
  Auslöser ≈ 36 aktive Partitionen oder anhaltender Datenbankgrößendruck.
- Synchrone `StockBalance`-Aktualisierung erhöht den Schreib-Overhead pro Event; dieser
  Overhead ist akzeptiert, da ATP-Korrektheit finanziell tragend ist.
- Ein vollständiger Bestandsstand ergibt sich nur durch Aggregation über den Log oder durch
  Lesen des `StockBalance`-Aggregatssatzes; direktes Lesen einzelner `StockMovement`-Zeilen
  liefert keine fertigen Mengen.
- Lebenszyklus-Events ohne Menge (z. B. `commissioning`, `inspecting`) erweitern den Log um
  Zeilen, die keine Mengenbewegung darstellen; Abfragen, die ausschließlich Mengenbewegungen
  benötigen, müssen `qty IS NOT NULL` als Filter setzen.

---

## Entitäten

**`StockMovement`** (workspace-scoped) — Ein unveränderlicher Lagerbewegungsdatensatz.

Felder:

| Feld                  | Typ / Quelle                                       | Beschreibung                                                      |
|-----------------------|----------------------------------------------------|-------------------------------------------------------------------|
| `event_type`          | Enum (EPCIS CBV)                                   | `OBJECT_EVENT`, `AGGREGATION_EVENT`, `TRANSACTION_EVENT`, `TRANSFORMATION_EVENT`, `ASSOCIATION_EVENT` |
| `business_step`       | String (EPCIS CBV URI oder Kurzcode)               | `shipping`, `receiving`, `packing`, `picking`, `accepting`, `inspecting`, `scrapping`, `returning`, `rental_out`, `rental_return`, `adjustment`, `commissioning`, `installing`, `removing`, `repairing`, `decommissioning`, `inventorying` |
| `occurred_at`         | Datetime                                           | Zeitstempel des physischen Ereignisses                            |
| `recorded_at`         | Datetime (auto)                                    | Zeitstempel der Datenbankeinbuchung                               |
| `source_location`     | FK → `Location` (ADR-0009, nullable)               | Quellstandort                                                     |
| `destination_location`| FK → `Location` (ADR-0009, nullable)               | Zielstandort                                                      |
| `product`             | FK → `Product` (ADR-0003)                          | —                                                                 |
| `variant`             | FK → `ProductVariant` (ADR-0003, obligatorisch)    | Nachtrag 2026-07-04 (OQ-0019): obligatorisch, vollzieht die ADR-0021-Ripple-Liste nach |
| `batch`               | FK → `Batch` (ADR-0012, nullable)                  | —                                                                 |
| `serial_unit`         | FK → `SerialUnit` (ADR-0012, nullable)             | —                                                                 |
| `handling_unit`       | FK → `HandlingUnit` (ADR-0009, nullable)           | —                                                                 |
| `parent_serial_unit`  | FK → `SerialUnit` (ADR-0012, nullable)              | Nachtrag 2026-07-04 (OQ-0020): Eltern-Einheit bei `AGGREGATION_EVENT`, gesetzt wenn Fertigprodukt-`tracking_mode = SERIAL` |
| `parent_batch`        | FK → `Batch` (ADR-0012, nullable)                   | Nachtrag 2026-07-04 (OQ-0020): Eltern-Charge bei `AGGREGATION_EVENT`, gesetzt wenn Fertigprodukt-`tracking_mode = BATCH` |
| `aggregation_group`   | UUID (nullable; obligatorisch bei `event_type = AGGREGATION_EVENT`) | Nachtrag 2026-07-04 (OQ-0020): gemeinsamer Gruppierungsschlüssel aller Kind-Zeilen desselben Fertigungsabschlusses, unabhängig von `tracking_mode` |
| `qty`                 | Dezimal (nullable)                                 | Menge; null bei mengenlosem Lebenszyklus-Event (z. B. Inspektion, Kalibrierung) |
| `uom`                 | FK → `core.Unit`                                   | —                                                                 |
| `reason_code`         | FK → `MovementReasonCode` (global)                 | Buchungsgrund (Inventurkorrektur, Ausschuss, …)                   |
| `document_type`       | Django ContentType (nullable)                      | Generische Belegverknüpfung                                       |
| `document_id`         | PositiveIntegerField (nullable)                    | Generische Belegverknüpfung                                       |
| `owner_type`          | Enum: `OWN`, `CUSTOMER_CONSIGNMENT`, `RENTAL`      | Eigentümerschaft nach der Bewegung                                |
| `owner_party`         | FK → `Party` (ADR-0001, nullable)                  | Eigentümer bei Fremdbestand                                       |
| `disposition`         | Enum (EPCIS CBV Disposition, nullable)             | `in_progress`, `reserved`, `in_transit`, `in_possession`, `returned`, `destroyed`; null bei Mengenbewegungen ohne Dispositionssemantik; unveränderlich nach Schreiben |
| `idempotency_key`     | UUID (workspace-eindeutig)                         | Verhindert Doppelbuchungen bei Retry                              |
| `compensates`         | FK → `StockMovement` (nullable)                    | Verweis auf den originialen Event bei Gegenbuchung                |
| `created_by`          | FK → User (nullable)                               | Auslösender Benutzer oder Systemprozess                           |

`StockMovement`-Zeilen sind nach dem Schreiben unveränderlich (`immutable`-Constraint auf
Applikationsebene; `UPDATE`- und `DELETE`-Rechte werden dem Applikationsbenutzer entzogen).

**`MovementReasonCode`** (global) — Lookup-Tabelle für Buchungsgründe.
Felder: `code` (plattformweiter eindeutiger String), `label_de`, `label_en`,
`applies_to_business_steps` (Array von `business_step`-Werten). Wird als Fixture ausgeliefert;
Mandanten können per Admin eigene Codes hinzufügen (workspace-scoped Erweiterung via
`MovementReasonCodeExtension`).

---

## Workspace-Scoping-Matrix

| Entität                        | Scoping   | Begründung                                                           |
|--------------------------------|-----------|----------------------------------------------------------------------|
| `StockMovement`                | workspace | Bewegungen sind Mandantendaten; Audit-Trail ist mandantenspezifisch  |
| `MovementReasonCode` (Basis)   | global    | Standardcodes sind plattformweit stabil; als Fixture auslieferbar    |
| `MovementReasonCode` (Erweiterung) | workspace | Mandantenspezifische Codes ergänzen den globalen Satz            |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Die EPCIS-2.0-Orientierung nutzt einen offenen GS1-Standard;
keine kommerziellen Lizenzen oder Quantalq-proprietären Abhängigkeiten sind erforderlich.
Das REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige Kommunikationsbrücke zum
Frontend.

---

## Standards-Verankerung

| Standard              | Verwendung im Modell                                                                          |
|-----------------------|-----------------------------------------------------------------------------------------------|
| GS1 EPCIS 2.0         | `event_type`-Enum (ObjectEvent, AggregationEvent, TransactionEvent, TransformationEvent, AssociationEvent); `business_step`-CBV-Codes; native Unterstützung mengenloser Lifecycle-Events |
| GS1 EPCIS CBV         | `business_step`-Werte: Lager (`shipping`, `receiving`, `packing`, `picking`, `accepting`, `scrapping`, `returning`); Lifecycle (`commissioning`, `installing`, `removing`, `inspecting`, `repairing`, `decommissioning`); Inventur (`inventorying`) |
| GS1 EPCIS CBV Disposition | `disposition`-Feld: `in_progress`, `reserved`, `in_transit`, `in_possession`, `returned`, `destroyed`; Diskriminator zwischen geplanten und physisch vollzogenen Mietbewegungen |
| WMS-Kanonmuster       | Bewegungstypen: Putaway, Pick, Pack, Ship, Transfer, Adjustment, Scrap, Return                |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben
`WorkspaceScopedModel`. `StockMovement.owner_party` referenziert `Party`.

**ADR-0002 (Admin-UI-Framework):** `StockMovement` ist über einen read-only DRF-Endpunkt
exponiert; Schreibzugriff erfolgt ausschließlich über dedizierte Bewegungs-Endpunkte
(nicht via generisches PATCH/PUT auf den Log).

**ADR-0003 (Produkt-Katalog-Backbone):** `StockMovement` referenziert `Product` und
`ProductVariant`. ADR-0003 definiert: „`ProductVariant` übernimmt die SKU-Identifikation sauber."

**ADR-0009 (Lager-Domänen-Backbone):** `StockMovement` referenziert `Location` (Quelle und
Ziel) und `HandlingUnit`.

**ADR-0010 (Lagerbestandszustände und Reservierungen):** `StockMovement`-Events aktualisieren
`StockBalance`-Felder; jede Bewegung überführt Mengen zwischen den virtuellen Zuständen
(`on_hand`, `in_transit`, `quarantine` etc.).

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `StockMovement` trägt FK auf `Batch`
und `SerialUnit`; Vorwärts- und Rückwärtsverfolgung traversiert den Movement-Log.

**ADR-0013 (Miet- und Kundenstandbestand):** `business_step`-Werte `rental_out` und
`rental_return` sind Sonderformen des `OBJECT_EVENT`-Typs für Mietbewegungen.

**ADR-0014 (Montage/Kitting und geteilter Bestand):** `TRANSFORMATION_EVENT` ist der
EPCIS-Typ für Stücklistenauflösungen und Kit-Explosionen. `AGGREGATION_EVENT` erfasst das
As-Built-BOM bei Fertigungsabschluss.

**ADR-0015 (Geräte-Lebenszyklus-Historie):** Die Lebenszyklus-Businessstep-Werte
(`commissioning`, `installing`, `removing`, `inspecting`, `repairing`, `decommissioning`)
sind die Ereignisgrundlage für die in ADR-0015 definierte Unit-History-Abfrage.

**ADR-0021 (Produkt-Variantengranularität):** `StockMovement.variant` ist obligatorisch
(Amendment 2026-07-04) und vollzieht die in ADR-0021 begonnene Umstellung der
Lager-/Serien-/Reservierungsdomäne auf `ProductVariant` als autoritativen Schlüssel nach.

## Amendments

### Amendment 2026-05-04 — OQ-0010: `disposition`-Feld und Soft-Reservierungs-Semantik

`StockMovement` erhält ein neues Feld `disposition` (Enum, nullable). Die Wertemengen orientiert
sich am GS1 EPCIS 2.0 CBV `Disposition`-Vokabular. Der definierte Wertebereich umfasst:
`in_progress`, `reserved`, `in_transit`, `in_possession`, `returned`, `destroyed`.

**Semantische Unterscheidung planend vs. physisch:**

Eine Soft-Reservierung — erzeugt beim Speichern einer Mietangebotsposition (ADR-0010) — schreibt
einen `StockMovement`-Datensatz mit `business_step = rental_out`, `qty = null` und
`disposition = reserved`. Dieser Eintrag dokumentiert die Planungsabsicht ohne physische
Mengenbewegung. Die physische Übergabe der Einheit an den Kunden schreibt einen zweiten
`StockMovement`-Datensatz mit `business_step = rental_out`, `qty = null` und
`disposition = in_possession`. Der Wechsel von `reserved` zu `in_possession` unterscheidet im
Log eindeutig, ob eine Mietausgabe geplant oder vollzogen ist. Kein separater
`planned_rental_out`-Businessstep wird eingeführt; das `disposition`-Feld ist der alleinige
Diskriminator. Bei Mietrückgabe trägt der abschließende `StockMovement`-Eintrag
`business_step = rental_return` und `disposition = returned`. Kompensierende Gegenbuchungen
(Freigabe einer Reservierung nach Angebotsrückzug, ADR-0010) tragen `disposition = null` und
`business_step = adjustment`; das Feld `compensates` verweist auf den ursprünglichen
Planungseintrag.

Das `disposition`-Feld ist nullable: Mengenbewegungen ohne Dispositionssemantik (z. B.
`receiving`, `shipping`, `adjustment`) tragen `disposition = null`. Partitionierung,
Aufbewahrungsuntergrenze und synchrone `StockBalance`-Aktualisierung bleiben unverändert.
Die Unveränderlichkeitsregel gilt für das `disposition`-Feld gleichermaßen: kein UPDATE nach
dem Schreiben; Zustandsänderungen entstehen als neue `StockMovement`-Zeilen.

Das `disposition`-Feld wird in die `StockMovement`-Feldetabelle aufgenommen (zwischen
`owner_party` und `idempotency_key`).

| Feld          | Typ / Quelle                                   | Beschreibung                                                                |
|---------------|------------------------------------------------|-----------------------------------------------------------------------------|
| `disposition` | Enum (EPCIS CBV Disposition, nullable)         | `in_progress`, `reserved`, `in_transit`, `in_possession`, `returned`, `destroyed`; null bei Mengenbewegungen ohne Dispositionssemantik |

**CBV-Businessstep-Erweiterung:** Kein neuer `business_step`-Wert wird eingeführt. Die
bestehenden Werte `rental_out` (mit `disposition = reserved` für Planung, `disposition =
in_possession` für physische Übergabe) und `rental_return` (mit `disposition = returned`) decken
alle Mietphasen vollständig ab. Die Standards-Verankerungstabelle wird um das
`disposition`-CBV-Vokabular erweitert.

### Amendment 2026-05-04 — OQ-0017: `inventorying` als gültiger `business_step`-Wert

UC-0009 (Komponentenentnahme mit Bestandsbestätigung) und UC-0008 (Lagerbestand per Barcode
lokalisieren) nutzen Ad-hoc-Zykluszählungen und bestätigte Bestandskontrollen an einem
Standort. Der bisherige `business_step`-Wertebereich enthielt keinen Wert für
Inventurvorgänge.

**Semantik `inventorying`:** Ein `StockMovement` mit `business_step = inventorying` ist ein
Verifikationsdatensatz für eine Ad-hoc-Zykluszählung oder bestätigte Bestandskontrolle an
einem Standort. Er mutiert den Saldo **nicht** — die Invariante „`StockMovement`-Events mit
`qty != null` aktualisieren `StockBalance`-Felder (ADR-0010) synchron im selben
Datenbank-Transaktion" gilt nicht für `inventorying`-Events. Ein `inventorying`-Event trägt
`qty = null` (mengenloses Verifikationsereignis) und gilt damit als gültiges Lebenszyklus-Event
im Sinne der bestehenden Regel.

**Diskrepanz-Behandlung:** Weicht die gezählte Menge (`counted_qty`) vom Wert in
`OnHandRecord.qty_on_hand` ab, schreibt die Applikationsschicht einen zweiten, separaten
`StockMovement`-Datensatz mit `business_step = adjustment`, `reason_code =
cycle_count_discrepancy` und `qty = <Delta>`. Dieser `adjustment`-Event ist der einzige
Datensatz, der `OnHandRecord` und `StockBalance` synchron im selben Datenbank-Transaktion
aktualisiert. Die Unveränderlichkeitsregel bleibt gewahrt: kein UPDATE auf bestehenden
`StockMovement`-Zeilen; Korrekturen entstehen als neue Zeilen.

**Treffer ohne Diskrepanz:** Stimmt die Zählung überein, wird ausschließlich der
`inventorying`-Event geschrieben (kein `adjustment`-Event).

Lizenzbeschränkung: Kein neuer Closed-Source-Baustein. `inventorying` ist ein
projektspezifischer Businessstep; GS1 EPCIS 2.0 CBV enthält diesen Wert nicht als
Standardcode.

### Amendment 2026-07-04 — OQ-0019: `variant` wird obligatorisch (ADR-0021-Ripple abgeschlossen)

Die Ripple-Liste der ADR-0021-Amendments (Abschnitt „Ripple-Liste Lager-/Serien-/
Reservierungsdomäne") schliesst `OnHandRecord`, `Batch`, `SerialUnit`, `StockBalance` und
`StockReservation` als durchgängig auf `ProductVariant` geschlüsselt ein, führt `StockMovement`
jedoch nicht auf. `StockMovement.variant` bleibt bislang nullable, obwohl jedes `Product`
(ADR-0021: „jedes `Product` besitzt ≥1 `ProductVariant`") mindestens eine `ProductVariant` trägt
und `StockMovement.product` bereits obligatorisch ist.

**Korrekte Aussage:** `StockMovement.variant` trägt einen obligatorischen FK auf
`ProductVariant`. Jede Zeile — Mengenbewegung wie mengenloses Lebenszyklus-Event — referenziert
neben dem `product`-FK die konkrete `ProductVariant`, für die das Ereignis gilt. Dies schliesst
die Lücke, die die Ripple-Liste offen liess: Die Schlüsselungskette (`OnHandRecord`, `Batch`,
`SerialUnit`, `StockBalance`, `StockReservation`, `StockMovement`) ist damit vollständig auf
`ProductVariant` konsistent.

**Auflösung der Komponenten-Variante bei BOM-Buchungen (OQ-0019, gemeinsam mit ADR-0006 und
ADR-0014):** `BomItem` (ADR-0006) bleibt Product-gekeyt; die konkrete Komponenten-`ProductVariant`
für einen `StockMovement`-Eintrag wird ausschliesslich am Buchungspunkt aufgelöst, nach dieser
Reihenfolge:

1. Eine im Request explizit angegebene `ProductVariant` (z. B. Auswahl im Kommissionier- oder
   Reservierungsformular).
2. `BomItem.default_component_variant` (ADR-0006, Nachtrag 2026-07-04), sofern gesetzt.
3. Die einzige `ProductVariant` des Komponenten-`Product`, sofern dieses genau eine trägt.

Lässt sich keine dieser drei Stufen auflösen (Komponenten-`Product` trägt mehr als eine
`ProductVariant`, kein Standardwert gesetzt, keine explizite Angabe im Request), weist das
Backend die Buchung mit HTTP 422 ab und benennt die zur Auswahl stehenden `ProductVariant`-Werte.

### Migrationsbedeutung

Die v2.0.0-Migration (REQ-0007) legt die Standardvariante an, auf die bestehende
`StockMovement`-Zeilen bei der Umstellung von nullable auf obligatorisches `variant`-Feld
verweisen.

### Amendment 2026-07-04 — OQ-0020: Eltern-Anker für `AGGREGATION_EVENT` bei nicht-serialisiertem Fertigprodukt

ADR-0014 beschreibt die As-Built-Erfassung als „ein oder mehrere `AGGREGATION_EVENT`-Einträge,
die den Eltern-`SerialUnit`-FK (Fertigprodukt) mit den Kind-Komponenten-Lots/Serien verknüpfen".
`StockMovement` trägt jedoch je Zeile genau einen `serial_unit`- und einen `batch`-FK; kein Feld
verknüpft mehrere Kind-Zeilen eindeutig mit derselben Eltern-Einheit desselben
Fertigungsabschlusses, und für ein Fertigprodukt mit `tracking_mode ∈ {NONE, BATCH}` existiert
keine Eltern-`SerialUnit`, auf die die `AGGREGATION_EVENT`-Zeilen zeigen könnten.

**Korrekte Aussage:** `StockMovement` erhält drei neue Felder: `parent_serial_unit` (FK →
`SerialUnit`, nullable), `parent_batch` (FK → `Batch`, nullable) und `aggregation_group` (UUID,
nullable, obligatorisch bei `event_type = AGGREGATION_EVENT`). `aggregation_group` ist der
alleinige, tracking-mode-unabhängige Gruppierungsschlüssel: Alle `AGGREGATION_EVENT`-Zeilen
desselben Fertigungsabschlusses tragen denselben `aggregation_group`-Wert, unabhängig davon, ob
das Fertigprodukt `tracking_mode = SERIAL`, `BATCH` oder `NONE` trägt. Die Applikationsschicht
erzeugt genau einen `aggregation_group`-Wert je Fertigungsabschluss (nicht je `ProductionOrder`,
da ein `ProductionOrder` mehrere Teilabschlüsse mit je eigenem `actual_qty` haben kann).

Zusätzlich zum tracking-mode-unabhängigen `aggregation_group`-Schlüssel setzt die
Applikationsschicht — sofern vorhanden — den passenden diskreten Eltern-Anker:

- `tracking_mode = SERIAL`: `parent_serial_unit` trägt die Fertigprodukt-`SerialUnit` jeder
  Kind-Zeile; direkte Abfragen „welche Kind-Zeilen gehören zu dieser Fertigprodukt-Einheit"
  laufen ohne Umweg über `aggregation_group`.
- `tracking_mode = BATCH`: `parent_batch` trägt die Fertigprodukt-`Batch` jeder Kind-Zeile,
  analog zu `parent_serial_unit`.
- `tracking_mode = NONE`: Weder `parent_serial_unit` noch `parent_batch` sind gesetzt;
  `aggregation_group` ist der einzige Anker. As-Built-Traceability für diesen Fall liefert die
  Menge der Kind-Komponenten-Lots/Serien des Fertigungsabschlusses, jedoch keine Verknüpfung zu
  einer einzelnen Fertigprodukt-Einheit (es existiert keine).

Diese Struktur bleibt vollständig innerhalb der bestehenden `StockMovement`-Tabelle — keine
separate As-Built-Aggregatentität wird eingeführt; ADR-0014 hat diese Alternative bereits
explizit abgelehnt („statt einer separaten As-Built-Tabelle"), und diese Ablehnung bleibt
unverändert bestehen. `aggregation_group` ist bei einem späteren EPCIS-2.0-Export als Grundlage
für `parentID` nutzbar: Bei `tracking_mode = SERIAL`/`BATCH` liefert die Fertigprodukt-Einheit
selbst die EPCIS-`parentID`; bei `tracking_mode = NONE` mint die Exportschicht eine interne
URN aus `aggregation_group`, da EPCIS 2.0 keinen physischen Trägeridentifikator für diesen Fall
vorschreibt.

### Migrationsbedeutung

Bestehende `AGGREGATION_EVENT`-Zeilen erhalten rückwirkend einen `aggregation_group`-Wert je
`document_id` (`ProductionOrder`)-Gruppe der v2.0.0-Migration (REQ-0007); mehrere historische
Teilabschlüsse desselben `ProductionOrder`, die migrationsseitig nicht unterscheidbar sind,
erhalten denselben `aggregation_group`-Wert (dokumentierte Migrationsungenauigkeit).

## Changelog
- 2026-07-04: OQ-0019 geschlossen (gemeinsam mit ADR-0006 und ADR-0014): `variant` wird
  obligatorisch, schliesst die ADR-0021-Ripple-Liste ab; Komponenten-Variantenauflösung bei
  Buchung dreistufig festgelegt. OQ-0020 geschlossen (gemeinsam mit ADR-0014):
  `parent_serial_unit`, `parent_batch` und `aggregation_group` als tracking-mode-unabhängiger
  As-Built-Eltern-Anker eingeführt. Siehe Amendments 2026-07-04.
- 2026-05-03: Erstentscheidung.
- 2026-05-03: Geltungsbereich auf Lebenszyklus-Events erweitert (mengenlose Ereignisse,
  neue CBV-Businessstep-Codes). OQ-0004 geschlossen: PostgreSQL declarative Range-Partitionierung
  nach `occurred_at`, monatlich, vor Datenlauf eingerichtet. OQ-0005 geschlossen: synchrone
  `StockBalance`-Aktualisierung im selben Datenbank-Transaktion festgelegt. Workspace-weite
  konfigurierbare Aufbewahrungsuntergrenze (`retention_floor`) als additive Schutzsperre
  eingeführt. Titel auf „Lager- und Lebenszyklus-Ereignis-Log" aktualisiert.
- 2026-05-04: OQ-0010 geschlossen: `disposition`-Feld (EPCIS CBV Enum, nullable) eingeführt; Soft-Reservierungs-Semantik (`disposition=reserved` vs. `disposition=in_possession`) als Diskriminator für geplante vs. physische Mietausgabe festgelegt; kein separater `planned_rental_out`-Businessstep. Siehe Amendment 2026-05-04.
- 2026-05-04: OQ-0017 geschlossen: `inventorying` als projektspezifischer `business_step`-Wert ergänzt; Semantik: Verifikationsdatensatz ohne Saldo-Mutation; Diskrepanz-Behandlung über separaten `adjustment`-Event mit `reason_code = cycle_count_discrepancy`. UC-0008, UC-0009 als auslösende Use Cases. Siehe Amendment 2026-05-04 (OQ-0017).
