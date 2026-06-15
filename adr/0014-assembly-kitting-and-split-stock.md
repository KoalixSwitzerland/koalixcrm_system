# ADR-0014: Montage/Kitting und geteilter Bestand

## Status
Accepted

## Context

ADR-0006 definiert `BillOfMaterials` und `BomItem` für Produkte mit `kind = MANUFACTURED_GOOD`
nach ISA-95 Part 2. Wird ein Fertigungs- oder Montageauftrag ausgelöst, müssen die
Komponenten-`OnHandRecord`-Mengen aus ADR-0009 partiell gebunden werden: Ein Teil des
Komponentenbestands ist für den laufenden Auftrag committet, der Rest bleibt frei disponierbar.
ADR-0003 definiert `kind = KIT` für Produkte, die als zusammengestelltes Bündel verkauft werden,
ohne eigene Fertigung; Kits erfordern eine „Explosion" der Stückliste in Einzelkomponenten bei
der Auftragserfüllung oder alternativ eine physische Vorkommissionierung. Diese beiden
Anwendungsfälle — Fertigungscommitment und Kit-Handling — sind eng genug verwandt, um in einem
ADR behandelt zu werden.

## Decision

Für Fertigungs- und Montageaufträge wird eine `ProductionOrder`-Entität eingeführt, die einen
`BillOfMaterials`-Eintrag referenziert und via `ProductionOrderComponent`-Zeilen die geplante
und tatsächlich entnommene Menge je `BomItem` trägt. Die Bestandsbindung von Komponenten erfolgt
über `StockReservation`-Einträge (ADR-0010, `reservation_type = RESERVED_FOR_DOCUMENT`) mit
Dokumentverweis auf den `ProductionOrder`. Das fertige Produkt entsteht als `StockMovement`
vom Typ `TRANSFORMATION_EVENT` (ADR-0011): Quell-`OnHandRecord`-Zeilen der Komponenten werden
verbraucht, eine neue `OnHandRecord`-Zeile des Fertigprodukts entsteht. Bei Fertigungsabschluss
MUSS die Applikationsschicht zusätzlich zum `TRANSFORMATION_EVENT` ein oder mehrere
`AGGREGATION_EVENT`-Einträge emittieren, die das As-Built-BOM erfassen: Eltern-`SerialUnit`
(Fertigprodukt) ← Kind-Komponenten-Lots/Serien. Diese `AGGREGATION_EVENT`-Zeilen sind die
unveränderliche Quelle der Wahrheit für die As-Built-Provenienz und ermöglichen
Vorwärts-/Rückwärtsverfolgung bei Rückrufen. Für `kind = KIT` (ADR-0003) gelten zwei
Betriebsmodi, die pro Produkt konfigurierbar sind: `EXPLODE_ON_PICK` (Stückliste wird zum
Pickezeitpunkt aus der vorberechneten Explosions-Snapshot-Tabelle `BillOfMaterialsExplosion`
gelesen; keine physische Vorkommissionierung) und `PREASSEMBLE` (Kit wird vorab physisch
zusammengestellt; ein Kit-`OnHandRecord` entsteht). Der Betriebsmodus wird über ein neues
`kit_mode`-Feld auf `Product` (additiv, ADR-0003) gesteuert.

Eine Celery-Task berechnet die abgeflachte BOM-Explosion bei Anlage oder Änderung eines
`BillOfMaterials`-Eintrags und speichert das Ergebnis in der `BillOfMaterialsExplosion`-
Snapshot-Tabelle. Der Pick-Zeitpfad liest die Snapshot-Tabelle. Weicht die Snapshot-Version
von der aktuellen BOM-Version ab, explodiert die Applikationsschicht synchron neu (dieser Fall
ist selten). Weiche Tiefengrenze: 10 BOM-Ebenen (System warnt und empfiehlt `PREASSEMBLE`);
harte Tiefengrenze: 20 Ebenen (System verweigert die Aufnahme und erzwingt `PREASSEMBLE`).

## Why

Die Abbildung der Bestandsbindung über `StockReservation` (ADR-0010) — statt eines separaten
Commitment-Felds auf `OnHandRecord` — hält das ATP-Modell konsistent: committeter Bestand
ist `qty_reserved_for_document` und geht korrekt in die ATP-Formel ein, ohne ein siebtes
Mengensegment einzuführen. Die Verwendung von `TRANSFORMATION_EVENT` (EPCIS 2.0, ADR-0011)
für die Fertigstellung ist semantisch korrekt: Inputprodukte werden in ein Outputprodukt
transformiert, was dem EPCIS-Kanonmodell entspricht und später für GS1-Export nutzbar ist.
`AGGREGATION_EVENT`-Einträge für das As-Built-BOM — statt einer separaten As-Built-Tabelle —
halten die Fertigungs- und Komponentenprovenienz im selben Event-Log wie alle anderen
Bewegungen; Rückrufverfolgung traversiert denselben Log ohne Tabellenwechsel. Die
vorberechnete `BillOfMaterialsExplosion`-Snapshot-Tabelle entkoppelt die Explosion von der
Pick-Latenz; harte und weiche Tiefengrenzen verhindern, dass rekursive BOM-Bäume die
synchrone Explosionslogik unbegrenzt blockieren.

## Alternatives Considered

- **Komponentenbindung als separates `ComponentCommitment`-Feld auf `OnHandRecord`** —
  abgelehnt: verdoppelt das Mengensegmentierungsmodell aus ADR-0010 (`StockBalance`); ATP
  müsste ein achtes Feld berücksichtigen; die Semantik überschneidet sich vollständig mit
  `StockReservation`.
- **Kit-Explosion ausschließlich in der Applikationsschicht ohne Lagerspiegelung** — abgelehnt:
  Kit-Kommissionierung ohne Lagerbuchung hinterlässt keine `StockMovement`-Events;
  Differenzklärung und Audit-Trail fehlen; Bestandsabweichungen sind nicht nachvollziehbar.
- **Kit als separates Modell außerhalb von `Product`** — abgelehnt: ADR-0003 definiert
  `kind = KIT` als Enum-Wert auf `Product`; ein separates Modell würde die Backbone-Struktur
  duplizieren und die Preislogik (ADR-0005) und Klassifizierung (ADR-0004) für Kits
  entkoppeln.

## Consequences

### Positive
- Komponentenbindung via `StockReservation` (ADR-0010) hält die ATP-Formel unverändert;
  kein neues Mengensegment, kein neues ATP-Modell.
- `TRANSFORMATION_EVENT` in ADR-0011 ist der EPCIS-konforme Typ für Stücklistenauflösungen;
  Fertigungsevents sind ohne Remodellierung EPCIS-exportierbar.
- `PREASSEMBLE`-Kits erscheinen als eigene `OnHandRecord`-Zeilen; Lagerortabfragen zeigen
  physisch vorkommissionierte Bündel direkt an.
- `EXPLODE_ON_PICK`-Kits erzeugen keinen Vorlagerbestand; der Lagerbestand bleibt in
  Einzelkomponenten, was Überbestand vermeidet.

### Negative
- `EXPLODE_ON_PICK`-Kits lesen die `BillOfMaterialsExplosion`-Snapshot-Tabelle; liegt kein
  gültiger Snapshot vor oder weicht die Version ab, explodiert die Applikationsschicht synchron
  nach. Die Celery-Task zum Vorberechnen des Snapshots muss zuverlässig laufen; ein Ausfall
  erhöht die Pick-Latenz im Fehlerfall.
- `ProductionOrder` ist eine neue Entität ohne vollständiges Fertigungssteuerungsmodell;
  Arbeitspläne (Routing), Maschinenbelegung und Personalplanung sind weiterhin nicht
  modelliert (bekannte Lücke aus ADR-0006).
- Mehrstufige Stücklisten (Baugruppe enthält Unterbaugruppen) erfordern rekursive
  `ProductionOrder`-Ketten; das Datenmodell unterstützt diese Rekursion, aber die
  Applikationslogik muss die Auflösung und Buchungsreihenfolge explizit steuern.
- Das Emittieren von `AGGREGATION_EVENT`-Zeilen bei jedem Fertigungsabschluss erhöht die
  Anzahl der Event-Log-Einträge pro Produktionsvorgang; bei hoher Variantenanzahl und großen
  BOMs wächst der Log schneller.

---

## Entitäten

**`ProductionOrder`** (workspace-scoped) — Ein Fertigungs- oder Montageauftrag.
Felder: FK auf `Product` (ADR-0003, `kind = MANUFACTURED_GOOD` oder `kind = KIT`), FK auf
`BillOfMaterials` (ADR-0006), `planned_qty` (Dezimal), `uom` (FK `core.Unit`), `status`
(Enum: `DRAFT`, `RELEASED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`), `planned_start`
(Datetime, nullable), `completed_at` (Datetime, nullable), `document_type` (Django ContentType,
nullable — Auftragsreferenz), `document_id` (PositiveIntegerField, nullable).

**`ProductionOrderComponent`** (workspace-scoped) — Eine Komponentenzeile eines
`ProductionOrder`.
Felder: FK auf `ProductionOrder`, FK auf `BomItem` (ADR-0006), FK auf `Product` (ADR-0003,
Komponente), FK auf `ProductVariant` (ADR-0003, nullable), FK auf `Batch` (ADR-0012, nullable),
`planned_qty` (Dezimal), `actual_qty` (Dezimal), `uom` (FK `core.Unit`), FK auf `StockReservation`
(ADR-0010, nullable — aktive Komponentenreservierung).

**`Product.kit_mode`** (additives Feld auf `Product` aus ADR-0003, gilt nur für `kind = KIT`) —
Enum: `EXPLODE_ON_PICK` (virtuelle Explosion zum Pickezeitpunkt aus Snapshot), `PREASSEMBLE`
(physische Vorkommissionierung). Defaultwert `EXPLODE_ON_PICK`. Wird von der Applikationsschicht
nur für `kind = KIT` ausgewertet.

**`BillOfMaterialsExplosion`** (workspace-scoped) — Vorberechnete, abgeflachte BOM-Explosion.
Felder: FK auf `BillOfMaterials` (ADR-0006), `bom_version` (Integer — Versionszähler des
BOM-Eintrags zum Zeitpunkt der Vorberechnung), `depth` (Integer — Tiefe im BOM-Baum),
FK auf `BomItem` (ADR-0006, Blatt-Komponente), `effective_qty` (Dezimal — Menge für
1 Einheit des Endprodukts nach Tiefenauflösung), `uom` (FK `core.Unit`). Die Tabelle wird
durch eine Celery-Task bei Anlage oder Änderung des `BillOfMaterials`-Eintrags vollständig
neu berechnet. Pick-Zeitpfad prüft, ob `bom_version` dem aktuellen `BillOfMaterials`-Stand
entspricht; bei Abweichung erfolgt synchrone Neuberechnung.

---

## Bestandsfluss bei Fertigungsabschluss

1. `ProductionOrder`-Status wechselt zu `IN_PROGRESS`.
2. `StockReservation`-Einträge (ADR-0010) für alle Komponenten werden mit
   `reservation_type = RESERVED_FOR_DOCUMENT` und FK auf `ProductionOrder` angelegt.
3. Physische Entnahme: `StockMovement` vom Typ `OBJECT_EVENT`, `business_step = 'picking'`,
   Mengen der Komponenten-`OnHandRecord`-Zeilen werden reduziert.
4. Fertigstellung: `StockMovement` vom Typ `TRANSFORMATION_EVENT` — alle Quell-Events der
   Komponentenentnahmen sind im selben EPCIS-Transformationsereignis gebündelt; eine neue
   `OnHandRecord`-Zeile des Fertigprodukts entsteht.
5. As-Built-Erfassung: Die Applikationsschicht emittiert ein oder mehrere `StockMovement`-Zeilen
   vom Typ `AGGREGATION_EVENT`, die den Eltern-`SerialUnit`-FK (Fertigprodukt) mit den
   Kind-Komponenten-Lots/Serien verknüpfen. Diese Zeilen sind die Quelle der Wahrheit für
   As-Built-Provenienz und Rückrufverfolgung.
6. `StockReservation`-Einträge werden als `FULFILLED` geschlossen; `StockBalance`-Felder
   (ADR-0010) werden aktualisiert.

---

## Workspace-Scoping-Matrix

| Entität                       | Scoping   | Begründung                                                          |
|-------------------------------|-----------|---------------------------------------------------------------------|
| `ProductionOrder`             | workspace | Fertigungsaufträge sind Mandantendaten                              |
| `ProductionOrderComponent`    | workspace | Komponentenzeilen sind Teil des Mandantenauftrags                   |
| `BillOfMaterialsExplosion`    | workspace | Explosions-Snapshot ist mandantenspezifisch                         |
| `Product.kit_mode`            | (gehört zu `Product`, workspace-scoped, ADR-0003) | —           |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Das
REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige Kommunikationsbrücke zum Frontend.

---

## Standards-Verankerung

| Standard       | Verwendung im Modell                                                                         |
|----------------|----------------------------------------------------------------------------------------------|
| ISA-95 Part 2  | `ProductionOrder` referenziert `BillOfMaterials` / `BomItem` (ADR-0006) nach ISA-95 Part 2 |
| GS1 EPCIS 2.0  | `TRANSFORMATION_EVENT` für Fertigstellungsbuchung (ADR-0011)                                |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben
`WorkspaceScopedModel`.

**ADR-0002 (Admin-UI-Framework):** `ProductionOrder` und `ProductionOrderComponent` sind über
DRF-Endpunkte exponiert; keine direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `ProductionOrder` referenziert `Product`. ADR-0003
definiert: „Das `kind`-Enum klassifiziert das Produkt als `SERVICE`, `TRADING_GOOD`,
`MANUFACTURED_GOOD`, `KIT` oder `RAW_MATERIAL`." `ProductionOrder` gilt für
`MANUFACTURED_GOOD` und `KIT`. `Product.kit_mode` ist eine additive Erweiterung orthogonal
zum `kind`-Enum.

**ADR-0006 (Beschaffung und Stücklisten):** `ProductionOrder` referenziert `BillOfMaterials`;
`ProductionOrderComponent` referenziert `BomItem`. ADR-0006 definiert: „`BomItem` unterstützt
Menge, Ausschuss-Prozentsatz und Alternativkomponenten nach ISA-95 Part 2."

**ADR-0009 (Lager-Domänen-Backbone):** `ProductionOrderComponent` liest und verbraucht
`OnHandRecord`-Zeilen der Komponenten.

**ADR-0010 (Lagerbestandszustände und Reservierungen):** Komponentenbindung nutzt
`StockReservation` mit `reservation_type = RESERVED_FOR_DOCUMENT`; ATP bleibt konsistent.

**ADR-0011 (Lager- und Lebenszyklus-Ereignis-Log):** Komponentenentnahme als `OBJECT_EVENT`;
Fertigstellung als `TRANSFORMATION_EVENT`; As-Built-BOM als `AGGREGATION_EVENT`. ADR-0011
definiert: „Jede Lagerbewegung und jeder Lebenszyklus-Touch einer Einheit wird als
unveränderlicher `StockMovement`-Datensatz gespeichert."

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `ProductionOrderComponent` trägt
optionalen FK auf `Batch` für chargengebundene Komponentenzuordnung; Traceability-Traversierung
(ADR-0012) traversiert `TRANSFORMATION_EVENT`- und `AGGREGATION_EVENT`-Zeilen für
mehrstufige BOM-Rückverfolgung.

**ADR-0015 (Geräte-Lebenszyklus-Historie):** `AGGREGATION_EVENT`-Einträge, die bei
Fertigungsabschluss emittiert werden, sind die Datengrundlage für die As-Built-BOM-Abfrage
und die Komponentengraph-Traversierung, die ADR-0015 als Lebenszyklus-Abfragepfad definiert.

## Changelog
- 2026-05-03: Erstentscheidung.
- 2026-05-03: OQ-0007 geschlossen: Celery-Task berechnet abgeflachte BOM-Explosion in
  `BillOfMaterialsExplosion`-Snapshot vor; Pick-Zeitpfad liest Snapshot; weiche Tiefengrenze
  10 (Warnung + `PREASSEMBLE`-Empfehlung), harte Tiefengrenze 20 (Ablehnung). Bei
  Fertigungsabschluss MUSS die Applikationsschicht `AGGREGATION_EVENT`-Zeilen für das
  As-Built-BOM emittieren (Eltern-SerialUnit ← Kind-Lots/Serien).
