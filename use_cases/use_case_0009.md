# UC-0009: Komponentenentnahme mit Bestandsbestätigung (Ad-hoc-Zykluszählung)

**ID:** UC-0009
**Bezug:** ADR-0009, ADR-0010, ADR-0011, ADR-0012
**Lizenzseite:** Open-Source-Backend (Datenmodell, Bewegungslogik, Zykluszählungs-Korrektur und API); Closed-Source-Frontend (Scan-Maske, Mengeneingabe-UI, Bestätigungsdialog)

**Warum:** Bei der Ad-hoc-Entnahme von Komponenten entstehen Bestandsabweichungen, wenn die tatsächlich entnommene Menge nicht sofort gebucht wird und der Restbestand am Stellplatz nicht verifiziert wird. Ohne eine kombinierte Picking-Buchung und sofortige Zykluszählung bei Abweichung veraltert der `OnHandRecord` schnell, was zu Fehlkommissionierungen bei nachfolgenden Aufträgen führt.

---

## Akteure

- **Primär:** Lagermitarbeiter (eingeloggter Benutzer mit Schreibrecht auf Lagerbewegungen im aktiven Workspace)
- **System:** KoalixCRM-Backend (DRF), KoalixCRM-Frontend (Next.js/Refine)

## Vorbedingungen

- Der Lagermitarbeiter ist authentifiziert und hat einen aktiven Workspace.
- Das zu entnehmende `Product` existiert im aktiven Workspace mit mindestens einem `OnHandRecord` an einem identifizierten `Location`-Knoten.
- Der Barcode des Produkts (GTIN aus `Product.gtin`) oder der Stellplatz-Barcode (`Location.external_ref`) ist physisch auf dem Produkt bzw. am Stellplatz vorhanden.

## Auslöser

Der Lagermitarbeiter entnimmt eine Komponente aus dem Lager und möchte die Entnahme im System erfassen.

---

## Hauptablauf

```plantuml
@startuml UC-0009-Hauptablauf
actor "Lagermitarbeiter" as LM
participant "Frontend\n(Next.js)" as FE
participant "Backend\n(DRF)" as BE
database "Datenbank" as DB

LM -> FE : Produkt-Barcode (GTIN) ODER\nStellplatz-Barcode (Location.external_ref) scannen
FE -> BE : POST /api/stock/scan/\n{identifier: "<gescannter Wert>", identifier_type: "AUTO"}
BE -> BE : Identifier-Auflösung:\nGTIN → Product.gtin (ADR-0003);\nexternal_ref → Location.external_ref (ADR-0009);\nglobal_uid → SerialUnit.global_uid (ADR-0012)
BE -> DB : Passenden OnHandRecord(s)\nfür aufgelösten Identifier + Workspace abfragen
DB --> BE : Product + Location + qty_on_hand (je OnHandRecord-Zeile)
BE --> FE : 200 OK — aufgelöster Identifier-Typ,\nProdukt, Stellplatz, aktuelle qty_on_hand
FE -> LM : Produkt und Stellplatz anzeigen;\nEingabefeld für entnommene Menge einblenden
LM -> FE : Entnommene Menge eingeben
FE -> BE : POST /api/stock/movements/\n{event_type: OBJECT_EVENT,\nbusiness_step: picking,\nproduct: <id>, source_location: <location_id>,\nqty: <entnommen>, uom: <id>,\nidempotency_key: <UUID>}
BE -> DB : INSERT StockMovement\n{business_step=picking, qty=<entnommen>,\nsource_location=<Stellplatz>,\noccurred_at=jetzt, disposition=null};\nsynchrones UPDATE StockBalance\n(qty_on_hand -= <entnommen>; ADR-0011)
DB --> BE : StockMovement-ID, neue qty_on_hand
BE --> FE : 201 Created — Picking-Event gespeichert;\nneue qty_on_hand am Stellplatz
FE -> LM : Bestätigung der Entnahme;\nEingabefeld für Restmenge am Stellplatz anzeigen
LM -> FE : Tatsächliche Restmenge am Stellplatz eingeben
FE -> BE : POST /api/stock/cycle-count/\n{location: <id>, product: <id>,\nconfirmed_qty: <Restmenge>,\nidempotency_key: <UUID>}
BE -> DB : qty_on_hand nach Picking-Event lesen\n(erwartete Restmenge = qty_on_hand_nach_picking)
DB --> BE : Erwartete Restmenge

alt Restmenge stimmt überein (confirmed_qty == erwartete_Restmenge)
    BE --> FE : 200 OK — keine Korrektur erforderlich;\nAudit-Trail vollständig (1 StockMovement)
    FE -> LM : Bestandsbestätigung: kein Unterschied
else Restmenge weicht ab (confirmed_qty ≠ erwartete_Restmenge)
    BE -> DB : INSERT StockMovement\n{business_step=inventorying,\nreason_code=cycle_count_discrepancy,\nqty=<delta: confirmed_qty - erwartete_Restmenge>,\nsource_location=<Stellplatz>,\noccurred_at=jetzt, disposition=null};\nsynchrones UPDATE StockBalance (ADR-0011)
    DB --> BE : Korrektur-StockMovement-ID
    BE --> FE : 201 Created — Zykluszählung mit\nKorrekturbuchung gespeichert;\n2 StockMovement-Zeilen im Audit-Trail
    FE -> LM : Bestandsbestätigung: Abweichung\nbegründet und korrigiert anzeigen
end
@enduml
```

---

## Alternativablauf A: Scan nicht auflösbar

- Das Backend findet für den gescannten Identifier keinen Treffer im aktiven Workspace (weder über `Product.gtin` noch über `Location.external_ref` noch über `SerialUnit.global_uid`).
- Das Backend antwortet mit HTTP 404 und benennt, welcher Identifier-Typ erkannt wurde (GTIN-Format erkannt, aber kein passendes Produkt; `external_ref`-Format erkannt, aber kein passender Stellplatz; unbekanntes Format).
- Das Frontend zeigt eine Fehlermeldung mit dem erkannten Identifier-Typ, damit der Lagermitarbeiter den Scan überprüfen oder manuell suchen kann.

## Alternativablauf B: Mengeneingabe überschreitet verfügbaren Bestand

- Der Lagermitarbeiter gibt eine entnommene Menge ein, die größer ist als die aktuelle `qty_on_hand` am Stellplatz.
- Das Backend gibt eine Warnung mit HTTP 422 zurück und nennt die verfügbare Menge.
- Das Frontend zeigt die Warnung; der Lagermitarbeiter bestätigt die Eingabe explizit (erzwungene Korrektur) oder passt die Menge an.
- Wird die Eingabe explizit bestätigt, bucht das Backend die Entnahme dennoch und setzt `qty_on_hand` auf 0 (kein Negativbestand); ein `StockMovement` mit `reason_code = quantity_exceeded` wird gespeichert.

## Alternativablauf C: Restmengeneingabe negativ oder leer

- Der Lagermitarbeiter gibt eine negative Zahl oder keinen Wert als Restmenge ein.
- Das Backend lehnt die Anfrage mit HTTP 400 ab.
- Das Frontend zeigt eine Eingabevalidierungsfehlermeldung; die Zykluszählung wird nicht gespeichert.

---

## Nachbedingungen

- Für die entnommene Menge existiert im `StockMovement`-Log genau ein unveränderlicher Eintrag mit `business_step = picking`, `qty = <entnommen>`, dem aufgelösten Produkt und dem Quell-Stellplatz.
- Hat der Lagermitarbeiter eine abweichende Restmenge bestätigt, existiert ein zweiter unveränderlicher `StockMovement`-Eintrag mit `business_step = inventorying`, `reason_code = cycle_count_discrepancy` und `qty = <delta>`.
- Hat der Lagermitarbeiter keine Abweichung festgestellt, existiert kein zweiter `StockMovement`-Eintrag für diesen Vorgang.
- `StockBalance.qty_on_hand` am betroffenen Stellplatz spiegelt den korrigierten Bestand nach beiden Buchungen wider.
- Beide `StockMovement`-Zeilen sind nach dem Schreiben unveränderlich (ADR-0011: „Zeilen werden nach dem Schreiben nicht mehr geändert oder gelöscht").

---

## Behavioral Acceptance Criteria

### BAC-1: Scan-Variante GTIN (Produkt-Barcode)

- [ ] Das Backend löst einen Scan, dessen Wert dem Muster einer GS1-GTIN entspricht, gegen `Product.gtin` auf und gibt das passende Produkt mit all seinen `OnHandRecord`-Standorten zurück.
- [ ] Die Antwort benennt `identifier_type = "GTIN"`, damit das Frontend den aufgelösten Typ anzeigen kann.

### BAC-2: Scan-Variante Location.external_ref (Stellplatz-Barcode)

- [ ] Das Backend löst einen Scan, der `Location.external_ref` entspricht, gegen den `Location`-Datensatz des aktiven Workspace auf und gibt alle am Stellplatz befindlichen Produkte mit ihrer `qty_on_hand` zurück (ADR-0009: „`external_ref` (optionaler AS/RS-Adressstring oder Barcode-Klartext)").
- [ ] Die Antwort benennt `identifier_type = "LOCATION_EXTERNAL_REF"`.

### BAC-3: Scan-Variante SerialUnit.global_uid

- [ ] Das Backend löst einen Scan, der dem Format `SerialUnit.global_uid` entspricht, gegen den `SerialUnit`-Datensatz auf und gibt das zugehörige Produkt und den aktuellen Standort zurück (ADR-0012: „`global_uid` (nullable — ISO/IEC-15459-kompatibler eindeutiger Identifikator)").
- [ ] Die Antwort benennt `identifier_type = "SERIAL_GLOBAL_UID"`.

### BAC-4: Buchung des Picking-Events

- [ ] Das Backend schreibt nach der Mengeneingabe einen `StockMovement`-Eintrag mit `event_type = OBJECT_EVENT`, `business_step = picking`, `qty = <entnommene Menge>` und `source_location = <Stellplatz>` (ADR-0011: `business_step`-Wertebereich enthält `picking`).
- [ ] Das Backend aktualisiert `StockBalance.qty_on_hand` am Stellplatz synchron im selben Datenbank-Transaktion (ADR-0011: „`StockMovement`-Events mit `qty != null` aktualisieren `StockBalance`-Felder synchron im selben Datenbank-Transaktion").
- [ ] Der `StockMovement`-Eintrag trägt einen eindeutigen `idempotency_key`; ein zweiter POST mit demselben Key erzeugt keine doppelte Buchung.

### BAC-5: Buchung des Korrektur-Events bei Abweichung

- [ ] Weicht die bestätigte Restmenge von der erwarteten Restmenge ab, schreibt das Backend einen zweiten `StockMovement`-Eintrag mit `business_step = inventorying`, `reason_code = cycle_count_discrepancy` und `qty = confirmed_qty − erwartete_Restmenge`.
- [ ] Das Backend korrigiert `StockBalance.qty_on_hand` um den Delta-Wert synchron im selben Transaktion.

### BAC-6: KEIN zusätzliches Event bei Übereinstimmung

- [ ] Stimmt die bestätigte Restmenge exakt mit der erwarteten Restmenge überein, schreibt das Backend keinen zweiten `StockMovement`-Eintrag.
- [ ] Das Backend antwortet in diesem Fall mit HTTP 200 (nicht 201), um die Abwesenheit einer Korrekturbuchung anzuzeigen.

### BAC-7: Audit-Trail (Unveränderlichkeit)

- [ ] Alle `StockMovement`-Zeilen dieses Use Cases sind nach dem Schreiben unveränderlich; kein `UPDATE` oder `DELETE` auf bereits gespeicherte Zeilen ist zulässig (ADR-0011: „Zeilen werden nach dem Schreiben nicht mehr geändert oder gelöscht; Korrekturen erfolgen durch kompensierende Gegenbuchungen").
- [ ] Beide `StockMovement`-Einträge (Picking + ggf. Korrektur) sind über die `StockMovement`-Log-Abfrage für denselben `product`/`location`-Kontext gemeinsam abrufbar.

### BAC-8: Workspace-Scope

- [ ] Das Backend löst Scans ausschließlich gegen `Product`, `Location` und `SerialUnit`-Einträge des aktiven Workspace auf (ADR-0001: „Tenant-owned data inherits `WorkspaceScopedModel` and is filtered by `request.active_workspace`").
- [ ] Alle erzeugten `StockMovement`-Einträge gehören dem aktiven Workspace an.

---

## Architectural gaps surfaced

Der Scan-Auflösungsmechanismus (Identifier Registry) über mehrere Identifier-Typen hinweg (GTIN, `external_ref`, `global_uid`) ist als eigenständige Komponente nicht in einem bestehenden ADR spezifiziert. Die vorliegende Auflösungslogik ist in diesem Use Case als Applikationsverhalten beschrieben; die architektonische Entscheidung, ob eine dedizierte Identifier-Registry-Entität oder eine reine Suchfunktion im Endpunkt angemessen ist, obliegt `kxcrm-architect`. Diese Lücke ist in `open_questions.md` als OQ-0016 erfasst.

Der `business_step`-Wert `inventorying` ist im Wertebereich von ADR-0011 nicht aufgeführt. Die geplante Erweiterung des Wertebereichs ist in diesem Use Case als Voraussetzung beschrieben; die formale Aufnahme erfolgt durch `kxcrm-architect` als ADR-0011-Amendment. Diese Lücke ist in `open_questions.md` als OQ-0017 erfasst.
