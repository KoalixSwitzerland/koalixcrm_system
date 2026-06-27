# UC-0010: Wareneingang mit Lieferschein und Lagerplatzvorschlag

**ID:** UC-0010
**Bezug:** ADR-0002, ADR-0003, ADR-0009, ADR-0011, ADR-0012
**Lizenzseite:** Open-Source-Backend (Datenmodell, GoodsReceipt-Aggregat, Lagerplatzvorschlag-Logik, Bewegungsbuchung und API); Closed-Source-Frontend (Lieferschein-Ingestion-UI, Scan-Maske, Lagerplatz-Override-Dialog)

**Warum:** Ohne eine strukturierte Lieferschein-Erfassung mit positionsgenauer Scan-Bestätigung und einem systemgestützten Lagerplatzvorschlag entstehen fehlerhafte Einlagerungen: Ware landet auf falschen Stellplätzen, Chargen werden nicht erfasst und `StockMovement`-Events fehlen im Audit-Trail. Die Verknüpfung jeder Buchung mit einem `GoodsReceipt`-Dokument stellt die Rückverfolgbarkeit vom physischen Eingang bis zum Lagerort sicher.

---

## Akteure

- **Primär:** Projektleiter oder Wareneingangs-Verantwortlicher (eingeloggter Benutzer mit Schreibrecht auf Wareneingang und Lagerbewegungen im aktiven Workspace)
- **System:** KoalixCRM-Backend (DRF), KoalixCRM-Frontend (Next.js/Refine)

## Vorbedingungen

- Der Benutzer ist authentifiziert und hat einen aktiven Workspace.
- Eine Lieferung mit Lieferschein ist physisch eingetroffen.
- Der Lieferschein-Payload liegt als strukturiertes Datenobjekt vor (Format und Quelle sind Integrationsdetails und nicht Teil dieses Use Cases).
- Alle im Lieferschein aufgeführten Produkte existieren im Produktkatalog des aktiven Workspace.

## Auslöser

Eine Lieferung mit Lieferschein trifft ein; der Wareneingangs-Verantwortliche erfasst den Eingang im System.

---

## Hauptablauf

```plantuml
@startuml UC-0010-Hauptablauf
actor "Wareneingangs-\nVerantwortlicher" as WV
participant "Frontend\n(Next.js)" as FE
participant "Backend\n(DRF)" as BE
database "Datenbank" as DB

WV -> FE : Lieferschein-Payload übermitteln\n(strukturierte Positionsliste)
FE -> BE : POST /api/stock/goods-receipts/\n{supplier?, positions: [{product_id, qty_expected,\nuom, batch_number?, expiry_date?}, …]}
BE -> BE : Alle product_id gegen Workspace-Katalog\nvalidieren (ADR-0003: Product + GTIN)
BE -> DB : INSERT GoodsReceipt {status=IN_PROGRESS,\nWorkspace-scoped};\nINSERT GoodsReceiptPosition je Position
DB --> BE : GoodsReceipt-ID
BE --> FE : 201 Created — GoodsReceipt\n{id, status=IN_PROGRESS, positions[]}
FE -> WV : GoodsReceipt anzeigen;\nPositionen-Liste mit Soll-Mengen

loop Für jede Position
    WV -> FE : Produkt-QR-Code der gelieferten Einheit scannen
    FE -> BE : POST /api/stock/scan/\n{identifier: "<GTIN>", identifier_type: "AUTO"}
    BE -> BE : Scan gegen Product.gtin auflösen;\nGoodsReceiptPosition-Treffer prüfen\n(product_id + GoodsReceipt-ID)
    BE -> DB : Lagerplatzvorschlag berechnen:\n1. Stellplatz mit gleichem Produkt + Charge bevorzugen;\n2. sonst: freier Stellplatz nächster passender Größe\n(ADR-0009: Location-Hierarchie + is_active)
    DB --> BE : Vorgeschlagener Location-Knoten + Breadcrumb
    BE --> FE : Scan-Treffer: Produkt bestätigt,\nLagerplatzvorschlag (Location-ID + Breadcrumb)
    FE -> WV : Produkt + Lagerplatz-Vorschlag anzeigen
    WV -> FE : Vorgeschlagenen Lagerplatz bestätigen\nODER anderen Lagerplatz auswählen
    FE -> BE : POST /api/stock/movements/\n{event_type: OBJECT_EVENT,\nbusiness_step: receiving,\nproduct: <id>, batch?: <id_oder_neu>,\ndestination_location: <location_id>,\nqty: <qty_expected>,\ndocument_type: GoodsReceipt,\ndocument_id: <GoodsReceipt.id>,\nidempotency_key: <UUID>}
    BE -> DB : INSERT StockMovement\n{business_step=receiving,\ndestination_location=<gewählter Lagerplatz>,\ndocument_id=GoodsReceipt.id,\noccurred_at=jetzt};\nsynchrones UPDATE StockBalance\n(qty_on_hand += qty_expected; ADR-0011);\nggf. INSERT Batch bei neuem Chargeneintrag\n(ADR-0012);\nGoodsReceiptPosition.status = CONFIRMED
    DB --> BE : StockMovement-ID
    BE --> FE : 201 Created — Buchung bestätigt;\nGoodsReceiptPosition-Status aktualisiert
    FE -> WV : Position als eingebucht markieren
end

BE -> DB : Alle GoodsReceiptPositions CONFIRMED?\n→ UPDATE GoodsReceipt.status = COMPLETED
DB --> BE : OK
BE --> FE : GoodsReceipt.status = COMPLETED
FE -> WV : Wareneingang abgeschlossen
@enduml
```

---

## Alternativablauf A: Scan stimmt nicht mit Lieferscheinposition überein

- Das Backend löst den gescannten Barcode auf ein Produkt auf, das in keiner offenen `GoodsReceiptPosition` des aktuellen `GoodsReceipt` auftaucht.
- Das Backend antwortet mit HTTP 422 und nennt das aufgelöste Produkt sowie die Produkte der offenen Positionen.
- Das Frontend zeigt den Konflikt; der Benutzer kann eine andere Position manuell auswählen oder den Scan wiederholen.

## Alternativablauf B: Produkt nicht im Katalog

- Das Backend findet für die im Lieferschein-Payload angegebene `product_id` keinen passenden `Product`-Eintrag im aktiven Workspace.
- Das Backend antwortet auf die initiale `POST /api/stock/goods-receipts/`-Anfrage mit HTTP 422 und benennt die fehlende `product_id`.
- Das Backend legt kein `GoodsReceipt`-Aggregat an; der Benutzer muss den fehlenden Katalogeintrag zuerst anlegen.

## Alternativablauf C: Vorgeschlagener Lagerplatz vom Benutzer abgelehnt

- Der Benutzer wählt einen anderen Stellplatz als den vom System vorgeschlagenen.
- Das Backend akzeptiert den benutzergewählten `Location`-Knoten, sofern dieser `is_active = true` trägt und zum aktiven Workspace gehört.
- Das Backend bucht den `StockMovement` auf den benutzergewählten Stellplatz.

## Alternativablauf D: Lieferung unvollständig

- Der Benutzer beendet den Erfassungsvorgang, ohne alle `GoodsReceiptPosition`-Einträge zu bestätigen.
- Das `GoodsReceipt` verbleibt im Status `IN_PROGRESS`.
- Bereits bestätigte Positionen bleiben als `StockMovement`-Einträge unveränderlich im Log; nur die offenen Positionen fehlen.
- Der Benutzer setzt den Eingang zu einem späteren Zeitpunkt fort, indem er das bestehende `GoodsReceipt` öffnet.

## Alternativablauf E: Mehrere Chargen auf einer Position

- Eine `GoodsReceiptPosition` enthält mehrere physische Chargen (unterschiedliche `batch_number`).
- Der Benutzer scannt und bestätigt jede Charge separat als Teillieferung der Position.
- Das Backend legt für jede Charge einen eigenen `StockMovement`-Eintrag und ggf. einen `Batch`-Datensatz an.
- Sobald die Summe der eingebuchten Mengen der Soll-Menge der Position entspricht, wird die Position auf `CONFIRMED` gesetzt.

---

## Nachbedingungen

- Ein `GoodsReceipt`-Aggregat mit Status `COMPLETED` (oder `IN_PROGRESS` bei Teillieferung) existiert im aktiven Workspace.
- Für jede bestätigte `GoodsReceiptPosition` existiert mindestens ein unveränderlicher `StockMovement`-Eintrag mit `business_step = receiving` und `document_id = GoodsReceipt.id`.
- `StockBalance.qty_on_hand` am jeweiligen Einlagerungsstellplatz spiegelt die eingebuchte Menge wider.
- Für jede neu erfasste Charge existiert ein `Batch`-Datensatz mit den übermittelten Feldern (ADR-0012: „Chargennummer, Ablaufdatum, Mindesthaltbarkeitsdatum, Produktionsdatum").
- Kein `StockMovement`-Eintrag dieses Use Cases ist nach dem Schreiben veränderbar (ADR-0011: „Zeilen werden nach dem Schreiben nicht mehr geändert oder gelöscht").

---

## Behavioral Acceptance Criteria

### BAC-1: Lieferschein-Ingestion über REST

- [ ] Das Backend nimmt einen strukturierten Lieferschein-Payload als JSON-Objekt entgegen und legt daraus ein `GoodsReceipt`-Aggregat mit Status `IN_PROGRESS` an (ADR-0002: „Django keeps the existing DRF endpoints in each `*_api_py/` package and serves them under `/api/`").
- [ ] Der Endpunkt akzeptiert ausschließlich strukturierte Positionsdaten (Produkt-ID, Soll-Menge, Maßeinheit, optional Charge); Bild- oder OCR-Daten sind nicht Teil des Payload-Schemas.

### BAC-2: GoodsReceipt-Statusübergänge

- [ ] Ein neu angelegtes `GoodsReceipt` trägt den Status `IN_PROGRESS`.
- [ ] Sobald alle `GoodsReceiptPosition`-Einträge den Status `CONFIRMED` tragen, setzt das Backend `GoodsReceipt.status = COMPLETED` synchron im selben Transaktion.
- [ ] Ein `GoodsReceipt` mit mindestens einer nicht bestätigten Position verbleibt im Status `IN_PROGRESS`.

### BAC-3: Scan-Auflösung pro Position

- [ ] Das Backend löst den gescannten GTIN-Barcode gegen `Product.gtin` auf und ordnet den Treffer der passenden offenen `GoodsReceiptPosition` zu (ADR-0003: „`Product` trägt `gtin` (GS1 GTIN)").
- [ ] Das Backend antwortet mit HTTP 422 und einer Fehlerbeschreibung, wenn der Scan kein Produkt einer offenen Position trifft.

### BAC-4: Lagerplatzvorschlag-Regel

- [ ] Das Backend schlägt als ersten Kandidaten einen `Location`-Knoten vor, der bereits `OnHandRecord`-Zeilen für dasselbe Produkt und dieselbe Charge trägt und dessen `is_active = true` gilt.
- [ ] Existiert kein solcher Stellplatz, schlägt das Backend den nächsten freien Stellplatz mit passender Kapazität vor (Kriterium: `is_active = true`, kein bestehender `OnHandRecord` mit `qty_on_hand > 0` für ein anderes Produkt, gleiche Hierarchieebene `BIN`).
- [ ] Das Backend liefert den Vorschlag als `Location`-ID mit vollständigem Breadcrumb-Array.

### BAC-5: Manuelle Override-Möglichkeit

- [ ] Der Benutzer wählt einen anderen als den vorgeschlagenen Stellplatz; das Backend akzeptiert diesen, sofern er `is_active = true` trägt und zum aktiven Workspace gehört.
- [ ] Das Backend bucht den `StockMovement` auf den benutzergewählten Stellplatz, nicht auf den vorgeschlagenen.

### BAC-6: Buchung pro Position als eigener StockMovement

- [ ] Für jede bestätigte `GoodsReceiptPosition` schreibt das Backend einen eigenen `StockMovement`-Eintrag mit `event_type = OBJECT_EVENT`, `business_step = receiving` (ADR-0011: `business_step`-Wertebereich enthält `receiving`).
- [ ] Das Backend aktualisiert `StockBalance.qty_on_hand` am Einlagerungsstellplatz synchron im selben Transaktion (ADR-0011: „`StockMovement`-Events mit `qty != null` aktualisieren `StockBalance`-Felder synchron im selben Datenbank-Transaktion").

### BAC-7: Verknüpfung über document_id

- [ ] Jeder `StockMovement`-Eintrag dieses Use Cases trägt `document_type = GoodsReceipt` und `document_id = <GoodsReceipt.id>` (ADR-0011: „Generische Belegverknüpfung" via `document_type` + `document_id`).
- [ ] Eine Abfrage aller `StockMovement`-Einträge mit `document_id = <GoodsReceipt.id>` liefert genau die Buchungen dieses Wareneingangs.

### BAC-8: Workspace-Scope

- [ ] Das Backend legt `GoodsReceipt`, `GoodsReceiptPosition` und alle `StockMovement`-Einträge im aktiven Workspace an (ADR-0001: „Tenant-owned data inherits `WorkspaceScopedModel` and is filtered by `request.active_workspace`").
- [ ] Alle Produkt- und Lagerplatz-Lookups sind auf den aktiven Workspace beschränkt.

### BAC-9: Lizenzgrenze — Endpoint-Schnittstelle

- [ ] Der Backend-Endpunkt `POST /api/stock/goods-receipts/` nimmt ausschließlich strukturierte JSON-Daten entgegen; keine Bildverarbeitung, kein OCR-Processing und keine Dokumenten-Parsing-Logik erfolgen im Open-Source-Backend.
- [ ] Die Lieferschein-Ingestion aus Bildern oder PDFs ist nicht Teil des Open-Source-Backends; dieser Integrationsweg obliegt dem Closed-Source-Frontend oder einem separaten Integrationsdienst.

---

## Architectural gaps surfaced

Die Put-Away-Strategie (Regel zur Lagerplatzvergabe beim Wareneingang) ist architektonisch nicht in einem bestehenden ADR als Service-Algorithmus oder konfigurierbares Produktattribut festgelegt. Dieser Use Case beschreibt die Wunschregel (bestehender Stellplatz mit gleichem Produkt + Charge zuerst, sonst freier Stellplatz nächster passender Größe); ob diese Logik als eigenständiger Algorithmus im Backend-Service, als konfigurierbare Strategie pro Produkt oder als mandantenweite Konfiguration implementiert wird, entscheidet `kxcrm-architect`. Diese Lücke ist in `open_questions.md` als OQ-0015 erfasst.

Das `GoodsReceipt`-Aggregat ist kein definiertes Datenmodell in einem bestehenden ADR. Das vorliegende ADR (ADR-0017 als Kandidat) ist noch nicht geschrieben; `kxcrm-architect` entscheidet über das Datenmodell. Diese Lücke ist in `open_questions.md` als OQ-0018 erfasst.

Der Identifier-Auflösungsmechanismus für Scan-Operationen ist auch in diesem Use Case relevant (vgl. UC-0009) und in keinem bestehenden ADR als eigenständige Identifier-Registry spezifiziert. Diese Lücke ist gemeinsam mit UC-0009 in `open_questions.md` als OQ-0016 erfasst.
