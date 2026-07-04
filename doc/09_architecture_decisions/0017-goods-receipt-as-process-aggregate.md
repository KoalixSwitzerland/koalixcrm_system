# ADR-0017: GoodsReceipt als Prozess-Aggregat

## Status
Accepted

## Context

UC-0010 (Wareneingang mit Lieferschein und Lagerplatzvorschlag) beschreibt einen mehrstufigen
Wareneingangs-Prozess: Ein Lieferant schickt Ware mit einem Lieferschein; ein Lagermitarbeiter
scannt Positionen, vergleicht gescannte Mengen mit erwarteten Mengen, identifiziert Abweichungen
und schließt den Vorgang ab, sobald alle Positionen bestätigt sind. Wareneingang kann über
mehrere Schritte oder Sitzungen verteilt ablaufen (Teillieferung). Kein bestehendes ADR
(ADR-0009 bis ADR-0015) enthält ein Aggregat, das den DRAFT/IN_PROGRESS/COMPLETED-Zustand
eines Wareneingangs, die pro-Position erwartete Menge, den pro-Position Abweichungsstatus und
die Verknüpfung zur erzeugten `StockMovement`-Buchung aufnimmt. ADR-0014 zeigt mit
`ProductionOrder` das Präzedenzprinzip: ein mehrstufiger Lagervorgang gehört in ein
strukturiertes Aggregat, nicht in rekonstruierte Log-Abfragen.

## Decision

`GoodsReceipt` wird als eigenständiges Prozess-Aggregat im Open-Source-Backend eingeführt.

**Header-Entität `GoodsReceipt`** (workspace-scoped):

| Feld              | Typ                                                         | Beschreibung                                      |
|-------------------|-------------------------------------------------------------|---------------------------------------------------|
| `id`              | UUID                                                        | Primärschlüssel                                   |
| `workspace`       | FK → Workspace                                              | Mandantenbindung (ADR-0001)                       |
| `supplier_party`  | FK → `Party` (ADR-0001)                                     | Lieferant                                         |
| `external_doc_ref`| String (nullable)                                           | Lieferscheinnummer des Lieferanten                |
| `received_at`     | Datetime                                                    | Zeitpunkt des physischen Wareneingangs            |
| `status`          | Enum: `DRAFT`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`      | Prozesszustand                                    |
| `notes`           | Text (nullable)                                             | Freitext-Notizen                                  |
| `created_by`      | FK → User (nullable)                                        | Auslösender Benutzer oder Systemprozess           |
| `created_at`      | Datetime (auto)                                             | Zeitstempel der Datenbankanlage                   |

**Kind-Entität `GoodsReceiptLine`** (workspace-scoped, FK auf `GoodsReceipt`):

| Feld              | Typ                                                         | Beschreibung                                             |
|-------------------|-------------------------------------------------------------|----------------------------------------------------------|
| `goods_receipt`   | FK → `GoodsReceipt`                                         | Eltern-Aggregat                                          |
| `product`         | FK → `Product` (ADR-0003)                                   | Empfangenes Produkt                                      |
| `expected_qty`    | Dezimal                                                     | Erwartete Menge laut Lieferschein                        |
| `received_qty`    | Dezimal                                                     | Tatsächlich gezählte/gescannte Menge                     |
| `batch`           | FK → `Batch` (ADR-0012, nullable)                           | Charge (optional)                                        |
| `line_status`     | Enum: `PENDING`, `CONFIRMED`, `MISMATCHED`                  | Positions-Übereinstimmungsstatus                         |
| `notes`           | Text (nullable)                                             | Freitext-Notizen zur Position                            |

**Zustandsübergang COMPLETED:**

Beim Übergang von `IN_PROGRESS` auf `COMPLETED` schreibt die Applikationsschicht für jede
`GoodsReceiptLine` mit `received_qty > 0` exakt einen `StockMovement`-Datensatz mit
`business_step = receiving`, `qty = received_qty`, `document_type = 'goods_receipt'`,
`document_id = <GoodsReceipt.id>` und `destination_location = <zugewiesener Lagerplatz aus
dem Einlagerungs-Schritt>`. Diese Buchungen erfolgen synchron im selben Datenbank-Transaktion,
konsistent mit der Regel: „`StockMovement`-Events mit `qty != null` aktualisieren
`StockBalance`-Felder (ADR-0010) synchron im selben Datenbank-Transaktion" (ADR-0011).

**Ingestion:**

`POST /api/v1/goods-receipts` akzeptiert einen strukturierten JSON-Payload mit Header-Feldern
und einer Positions-Liste. OCR-, EDI- und Lieferantenportal-Integrationen sind externe Adapter,
die diesen Payload erzeugen. Das Backend akzeptiert keine Binärdokumente direkt; der
OCR-Anbieter bleibt austauschbar (ADR-0002).

**Geltungsbereich dieses ADR:**

Die Put-Away-Strategie — welcher Algorithmus einen Lagerplatz für `destination_location`
vorschlägt — ist nicht Gegenstand dieses ADR. OQ-0015 bleibt offen; das Aggregat ist die
notwendige Voraussetzung, damit die Put-Away-Strategie in einem Folge-ADR definiert werden
kann.

## Why

Ein strukturiertes Aggregat statt eines reinen Movement-Tag-Musters (mehrere `StockMovement`-Zeilen
mit gemeinsamer `document_id`) ist die einzige Möglichkeit, `expected_qty` vor dem Schreiben des
`StockMovement` zu persistieren, den DRAFT/IN_PROGRESS-Zustand zwischen Sitzungen zu halten und
`MISMATCHED`-Positionen nach Abschluss des Vorgangs abfragbar zu machen. Das Prinzip entspricht
dem in ADR-0014 für `ProductionOrder` etablierten Muster: mehrstufige Lagervorgänge mit
Commitment-Semantik werden als eigenständige Aggregate modelliert, nicht aus Log-Abfragen
rekonstruiert.

## Alternatives Considered

- **Movement-Tag-Muster: mehrere `StockMovement`-Zeilen mit gemeinsamer `document_id`** —
  abgelehnt: (a) `expected_qty` lässt sich nicht persistieren, bevor der `StockMovement`
  geschrieben wird; (b) die Zustände `DRAFT` und `IN_PROGRESS` sind in einem unveränderlichen
  Log nicht darstellbar; (c) der per-Position `MISMATCHED`-Status verliert seine Bedeutung, sobald
  der `StockMovement`-Eintrag geschrieben oder weggelassen wurde — eine nachträgliche
  Rekonstruktion der Abweichung aus dem Log ist fehleranfällig.
- **Erweiterung von `StockReservation` (ADR-0010) um Wareneingangs-Felder** — abgelehnt:
  `StockReservation` modelliert Ausgangs- und Halte-Vorgänge; eine Erweiterung um eingehende
  Vorgänge vermischt zwei semantisch entgegengesetzte Prozesse und erhöht die Komplexität
  des Reservierungsmodells ohne architektonischen Gewinn.
- **JSONB-Felder auf `StockMovement` für erwartete Mengen und Abweichungsstatus** — abgelehnt:
  nicht strukturiert abfragbar; kein Typ-Sicherheits-Vorteil; verletzt die EPCIS-orientierte
  Feldstruktur aus ADR-0011.

## Consequences

### Positive
- `expected_qty` ist vor dem Buchungsschritt persistiert; Abweichungen sind auf Positions-Ebene
  (`MISMATCHED`) direkt abfragbar, ohne den `StockMovement`-Log traversieren zu müssen.
- DRAFT und IN_PROGRESS ermöglichen mehrsitzige Wareneingänge (Teillieferungen) ohne
  Datenverlust zwischen Sitzungen.
- Die Verknüpfung `document_type = 'goods_receipt'` / `document_id` im `StockMovement`
  (ADR-0011) liefert die bidirektionale Rückverfolgbarkeit: vom Aggregat zum Log und umgekehrt.
- OCR/EDI-Adapter sind externe Adaptor-Klassen; das Backend bleibt vendor-neutral (ADR-0002).
- Das Aggregat ist die notwendige Voraussetzung für OQ-0015 (Put-Away-Strategie); sobald die
  Strategie entschieden ist, kann der `destination_location`-Vorschlag in das Aggregat
  eingebettet werden ohne Schema-Änderung am Log.

### Negative
- Eine zusätzliche Tabelle (`GoodsReceipt` + `GoodsReceiptLine`) erhöht den Datenbankfußabdruck
  und erfordert eigene Migrations-Skripte.
- Die synchrone Buchungslogik beim COMPLETED-Übergang ist eine transaktionale Aufgabe; bei
  sehr langen Positions-Listen (> 1 000 Positionen) kann die Transaktion lang werden. Mandanten
  mit solch großen Lieferungen müssen den Vorgang in mehrere `GoodsReceipt`-Aggregate aufteilen.
- Der CANCELLED-Zustand erzeugt keine kompensierende `StockMovement`-Buchung, wenn kein
  COMPLETED vorausgegangen ist; ein partiell gebuchter und dann stornierter `GoodsReceipt`
  erfordert manuelle Korrekturbuchungen (adjustment-Events via ADR-0011).

---

## Workspace-Scoping-Matrix

| Entität               | Scoping   | Begründung                                                                     |
|-----------------------|-----------|--------------------------------------------------------------------------------|
| `GoodsReceipt`        | workspace | Wareneingangsvorgänge sind mandantenspezifische Geschäftsdaten                 |
| `GoodsReceiptLine`    | workspace | Kind-Entität des workspace-scopeten Aggregats                                  |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Aggregat lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel
und Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt und baut auf
keiner kommerziellen Abhängigkeit auf. OCR- und EDI-Adapter sind externe Integrationspunkte;
der OCR-Anbieter bleibt durch die Adapter-Grenze austauschbar, konsistent mit ADR-0002. Das
REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige Kommunikationsbrücke zum
Frontend.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** `GoodsReceipt.supplier_party` referenziert
`Party`. Workspace-scoped Entitäten erben `WorkspaceScopedModel`.

**ADR-0002 (Admin-UI-Framework):** OCR/EDI-Integrationen sind externe Adapter, die den
`POST /api/v1/goods-receipts`-Endpunkt aufrufen; das Backend bleibt vendor-neutral.

**ADR-0003 (Produkt-Katalog-Backbone):** `GoodsReceiptLine.product` referenziert `Product`.

**ADR-0009 (Lager-Domänen-Backbone):** `destination_location` im resultierenden
`StockMovement` referenziert `Location` (ADR-0009); der `LAYER`-Enum-Wert (ADR-0009,
Amendment 2026-05-04) ist als Zielstandort zulässig.

**ADR-0010 (Lagerbestandszustände und Reservierungen):** Die synchronen
`StockMovement`-Buchungen beim COMPLETED-Übergang aktualisieren `StockBalance`-Felder
synchron im selben Datenbank-Transaktion (ADR-0011-Invariante).

**ADR-0011 (Lager- und Lebenszyklus-Ereignis-Log):** Jede `GoodsReceiptLine` mit
`received_qty > 0` erzeugt beim COMPLETED-Übergang exakt einen `StockMovement` mit
`business_step = receiving`. Die Invariante gilt: „`StockMovement`-Events mit `qty != null`
aktualisieren `StockBalance`-Felder (ADR-0010) synchron im selben Datenbank-Transaktion."

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummernverfolgung):** `GoodsReceiptLine.batch`
trägt einen optionalen FK auf `Batch`.

**ADR-0014 (Montage/Kitting und geteilter Bestand):** `GoodsReceipt` folgt dem in ADR-0014
mit `ProductionOrder` etablierten Aggregat-Muster für mehrstufige Lagervorgänge.

**ADR-0021 (Produkt-Variantengranularität):** `GoodsReceiptLine.variant` FK →
`ProductVariant`, obligatorisch (Nachtrag 2026-07-04); vollzieht die ADR-0021-Ripple-Liste für
das Wareneingangs-Aggregat nach.

## Changelog
- 2026-05-04: Erstentscheidung. OQ-0018 geschlossen. UC-0010 als auslösender Use Case; UC-0008, UC-0009 als referenzierende Use Cases.
- 2026-07-04: Nachtrag — OQ-0021 geschlossen: `GoodsReceiptLine.product` (FK → `Product`)
  wird zu `GoodsReceiptLine.variant` (FK → `ProductVariant`, obligatorisch); vollzieht die
  ADR-0021-Schlüsselung nach, die für `StockMovement.variant` bereits obligatorisch gilt
  (ADR-0011, Nachtrag/Amendment 2026-07-04). Siehe Nachtrag 2026-07-04.

---

## Nachtrag (2026-07-04): `GoodsReceiptLine` → `ProductVariant` als autoritativer Schlüssel (ADR-0021, OQ-0021)

ADR-0021 legt fest, dass `sku`, `gtin`, `mpn` und alle Lagertatsachen auf `ProductVariant`
geschlüsselt sind; `OnHandRecord`, `Batch`, `SerialUnit`, `StockBalance`, `StockReservation`,
`StockMovement` und `ProductionOrderComponent` sind entsprechend umgestellt (ADR-0021,
Ripple-Listen). Die ursprüngliche Feldliste dieses ADR führte `GoodsReceiptLine.product` als
FK auf `Product`; keine der beiden ADR-0021-Ripple-Listen nannte ADR-0017. Da der beim
`COMPLETED`-Übergang erzeugte `StockMovement`-Datensatz (siehe §Zustandsübergang COMPLETED)
`StockMovement.variant` befüllen muss (ADR-0011, obligatorisch seit Nachtrag 2026-07-04,
OQ-0019), kann eine `GoodsReceiptLine` ohne konkrete `ProductVariant` die Buchung nicht
eindeutig erzeugen, sobald ein `Product` mehr als eine `ProductVariant` trägt. UC-0010 setzt in
seinem Payload-Beispiel (`POST /api/stock/goods-receipts/`) bereits `product_variant_id` je
Position voraus, was der bisherigen `Product`-FK-Feldliste widerspricht.

**Änderung:** `GoodsReceiptLine.product` (FK → `Product`) wird zu `GoodsReceiptLine.variant`
(FK → `ProductVariant`, obligatorisch). Der Bezug zum abstrakten Katalogobjekt bleibt über den
FK-Pfad `GoodsReceiptLine.variant → ProductVariant → Product` erreichbar; kein direktes
`Product`-Feld verbleibt auf `GoodsReceiptLine`.

**Chargenbezug (`GoodsReceiptLine.batch`):** `Batch` trägt seit ADR-0012 (Amendment
2026-07-04) einen obligatorischen FK auf `ProductVariant`. Die Applikationsschicht erzwingt,
dass eine gesetzte `GoodsReceiptLine.batch` zur selben `ProductVariant` gehört wie
`GoodsReceiptLine.variant` (`GoodsReceiptLine.batch.variant_id == GoodsReceiptLine.variant_id`);
kein Datenbank-Constraint. Damit bleibt die Chargenzuordnung entlang derselben
Variantenschlüsselung konsistent, die `Batch` bereits durchgängig trägt.

**Auswirkung auf die COMPLETED-Buchung:** Der beim `COMPLETED`-Übergang erzeugte
`StockMovement`-Datensatz übernimmt `variant` direkt von `GoodsReceiptLine.variant`; keine
Variantenauflösung am Buchungspunkt ist erforderlich (im Unterschied zur dreistufigen
Komponenten-Variantenauflösung aus ADR-0006/ADR-0011/ADR-0014, die für nicht explizit
gewählte BOM-Komponenten gilt — der Wareneingang kennt die empfangene Variante immer explizit).

Damit ist OQ-0021 geschlossen; siehe ADR-0021 §Ripple-Liste.
