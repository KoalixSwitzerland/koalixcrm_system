# ADR-0010: Lagerbestandszustände und Reservierungen

## Status
Accepted

## Context

`OnHandRecord` aus ADR-0009 hält die physische Menge pro Produkt × Standort × Charge/Serie. Für
Handels-, Fertigungs- und Mietbetrieb reicht eine einzelne Mengenspalte nicht aus: Ein
Verkaufsauftrag reserviert Material, das physisch noch im Regal liegt; ein eingehender
Einkaufsauftrag erhöht die bestellbare Menge, bevor die Ware eintrifft; Quarantäne-Einheiten
dürfen nicht kommissioniert werden. Ohne explizite Zustandsmengen führen einfache Bestandsabfragen
zu Über-Kommissionierung. Gleichzeitig erfordert Available-to-Promise (ATP) eine reproduzierbare
Formel, die für alle Mandanten einheitlich gilt. Das Reservierungsmodell muss zudem
`CommercialDocument`-Referenzen (Verkaufsauftrag, Transferauftrag, Angebot) tragen, ohne das
Commercial-Document-Modell direkt zu importieren.

## Decision

Lagermengen werden in fünf virtuelle Zustandsfelder auf einem `StockBalance`-Aggregatsdatensatz
aufgeteilt: `qty_on_hand`, `qty_booked`, `qty_reserved_for_document`, `qty_ordered`,
`qty_in_transit`, `qty_quarantine`. Eine Reservierung wird als eigene `StockReservation`-Entität
modelliert, die einen generischen Dokumenten-FK trägt (`document_type` + `document_id` via
Django Generic Relations). Die ATP-Formel lautet:
`ATP = qty_on_hand − qty_booked − qty_reserved_for_document + qty_ordered`.
`qty_in_transit` und `qty_quarantine` gehen nicht in ATP ein, da sie nicht disponierbar sind.
`StockBalance` ist ein denormalisierter Aggregatssatz, der durch `StockMovement`-Events
(ADR-0011) aktuell gehalten wird und nicht direkt beschrieben wird.

## Why

Mehrere additive Mengenfelder auf einem Aggregatssatz — statt eines polymorphen Zustandsenums
auf der atomaren `OnHandRecord`-Zeile — ermöglichen simultane SQL-Aggregatsfunktionen (`SUM`,
`FILTER`) ohne Self-Join, halten ATP-Berechnungen in einer einzelnen Tabellenzeile lesbar und
machen das Zustandsmodell für den DRF-Endpunkt in einer einzigen Serializer-Darstellung
darstellbar.

## Alternatives Considered

- **Einzelne polymorphe `state`-Spalte auf `OnHandRecord` (z. B. `state ∈ {on_hand, reserved,
  quarantine, …}`)** — abgelehnt: eine physische Einheit existiert mit genau einer Zeile, die
  aber gleichzeitig teilweise reserviert und teilweise frei sein kann; ein einzelner Zustand
  kann diese partielle Aufteilung nicht ausdrücken, ohne zusätzliche Mengenspalten einzuführen,
  was die Lösung auf dasselbe Ergebnis wie der gewählte Ansatz reduziert.
- **Mengenfelder direkt auf `OnHandRecord` statt auf separatem `StockBalance`** — abgelehnt:
  `OnHandRecord` ist die granulare Bewegungsebene (Charge × Standort × Eigentümer); ATP-Abfragen
  müssen über viele Zeilen aggregieren; ein Aggregatssatz hält lesintensive Pfade
  performant ohne Denormalisierungsrisiko auf jeder atomaren Zeile.
- **Reservierungen als Flags auf `CommercialDocument` ohne eigene Tabelle** — abgelehnt: eine
  Reservierung muss Menge, Einheit, Produkt, Standort und Dokumentreferenz tragen; diese Struktur
  ist zu reichhaltig für ein Flag auf dem Dokument und gehört als eigenständige Entität ins
  Lagerdatenmodell.
- **Separate ATP-Tabelle mit Trigger-Aktualisierung** — abgelehnt: ein separater Aggregatssatz
  (`StockBalance`), der über den Event-Log (ADR-0011) konsistent gehalten wird, hat identische
  Semantik, ist aber expliziter auditierbar und unabhängig von Datenbankebene-Triggern.

## Consequences

### Positive
- ATP-Formel ist in einer Tabellenzeile direkt lesbar und per Index auf
  `(workspace, product, variant, location)` effizient abfragbar.
- `StockReservation` trägt eine generische Dokumentreferenz; das Stock-Modul importiert keine
  direkte FK-Abhängigkeit auf `contracts`- oder `purchasing`-Modelle, was das PyPI-Paket
  modular hält.
- Zustandsübergänge (z. B. `booked → reserved_for_document`) werden über `StockMovement`
  (ADR-0011) protokolliert; der Audit-Trail ist vollständig.
- `qty_quarantine` verhindert Über-Kommissionierung aus gesperrten Chargen (ADR-0012)
  ohne zusätzliche Applikationslogik im Pick-Pfad.

### Negative
- `StockBalance` ist ein Aggregatssatz; er kann bei fehlender Event-Verarbeitung
  (z. B. ausgefallener Celery-Worker) kurzzeitig von `OnHandRecord`-Summen abweichen.
  Eine Reconciliation-Task muss als betrieblicher Prozess vorgesehen werden.
- Die ATP-Formel gilt global für alle Mandanten; individuelle ATP-Anpassungen
  (z. B. Sicherheitsbestand in ATP einrechnen) erfordern ein eigenes Folge-ADR.
- `qty_in_transit` setzt voraus, dass `StockMovement`-Events (ADR-0011) das
  Versandereignis vor dem Eingangsbestätigung korrekt sequenziert; fehlerhafte
  Sequenzierung ergibt temporär falsche Transit-Mengen.

---

## Entitäten

**`StockBalance`** (workspace-scoped) — Denormalisierter Aggregatssatz pro
`(workspace, product, variant, location)`.
Felder: FK auf `Product` (ADR-0003), FK auf `ProductVariant` (ADR-0003, nullable), FK auf
`Location` (ADR-0009), `qty_on_hand`, `qty_booked`, `qty_reserved_for_document`,
`qty_ordered`, `qty_in_transit`, `qty_quarantine` (alle Dezimal, gleiche `uom`), `uom`
(FK `core.Unit`). Der zusammengesetzte Unique-Constraint lautet:
`(workspace, product, variant, location)`.

**ATP-Formel:**
`ATP = qty_on_hand − qty_booked − qty_reserved_for_document + qty_ordered`

`qty_in_transit` und `qty_quarantine` gehen nicht in ATP ein.

**`StockReservation`** (workspace-scoped) — Eine aktive Reservierung einer Menge oder
Einzeleinheit, einschließlich zeitfenstergebundener Mietreservierungen.
Felder: FK auf `Product` (ADR-0003), FK auf `ProductVariant` (ADR-0003, nullable), FK auf
`Location` (ADR-0009, nullable — leer bedeutet: jeder Standort dieses Mandanten), FK auf
`Batch` (ADR-0012, nullable), FK auf `SerialUnit` (ADR-0012, nullable — gesetzt bei
`kind = RENTAL` und `kind = PROJECT_HOLD`),
`kind` (Enum: `SALE`, `RENTAL`, `PROJECT_HOLD`) — Reservierungsart,
`reservation_type` (Enum: `BOOKED`, `RESERVED_FOR_DOCUMENT`),
`reservation_status` (Enum: `PROVISIONAL`, `CONFIRMED`) — Verbindlichkeitsstufe der Reservierung,
`document_type` (Django ContentType), `document_id` (PositiveIntegerField) — generische
Dokumentreferenz für Verkaufsauftrag, Angebot, Transferauftrag, Einkaufsauftrag,
`qty_reserved` (Dezimal, nullable bei serialnummerngebundenen Reservierungen),
`uom` (FK `core.Unit`),
`rental_start` (Datetime, nullable — Mietbeginn bei `kind = RENTAL`),
`rental_end` (Datetime, nullable — Mietende bei `kind = RENTAL`),
`expires_at` (nullable Datetime),
`status` (Enum: `ACTIVE`, `FULFILLED`, `CANCELLED`, `EXPIRED`).

---

## Zustandsdefinitionen

| Zustand                   | Semantik                                                                             | In ATP |
|---------------------------|--------------------------------------------------------------------------------------|--------|
| `on_hand`                 | Physisch verfügbar und nicht gebunden                                                | Ja (positiv) |
| `booked`                  | Reserviert ohne Belegbezug (manuelle Sperre, z. B. Inventursperre)                  | Nein (wird abgezogen) |
| `reserved_for_document`   | Reserviert für einen kommerziellen Beleg (Auftrag, Angebot, Transferauftrag)         | Nein (wird abgezogen) |
| `ordered`                 | Eingehend aus Einkaufsauftrag; noch nicht physisch im Lager                          | Ja (positiv) |
| `in_transit`              | In Bewegung zwischen Standorten (unterwegs, noch nicht eingebucht)                   | Nein |
| `quarantine`              | QC-Sperre; darf nicht kommissioniert oder verbucht werden                            | Nein |

---

## Workspace-Scoping-Matrix

| Entität            | Scoping   | Begründung                                                              |
|--------------------|-----------|-------------------------------------------------------------------------|
| `StockBalance`     | workspace | Mengen sind Mandantendaten; ATP-Berechnung ist mandantenspezifisch      |
| `StockReservation` | workspace | Reservierungen binden mandantenspezifische Bestände an Belege           |
| Zustands-Enum-Werte | global   | Zustandsdefinitionen sind plattformweit stabil; als Enum im Code        |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Die generische Dokumentreferenz (`document_type` + `document_id`)
vermeidet eine direkte Importabhängigkeit auf `contracts`-Modelle; das Stock-Modul ist damit
unabhängig von geschlossenen oder mandantenspezifischen Erweiterungen des Belegmodells
installierbar. Das REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige
Kommunikationsbrücke zum Frontend.

---

## Standards-Verankerung

| Standard      | Verwendung im Modell                                                                       |
|---------------|--------------------------------------------------------------------------------------------|
| WMS-Kanonmuster | Zustandsdefinitionen (`booked`, `reserved_for_document`, `ordered`, `in_transit`, `quarantine`) folgen dem WMS-Standard-Vokabular |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben
`WorkspaceScopedModel`.

**ADR-0002 (Admin-UI-Framework):** `StockBalance` und `StockReservation` sind über
DRF-Endpunkte exponiert; keine direkte Modell-Referenz im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** `StockBalance` und `StockReservation` tragen FK auf
`Product` und `ProductVariant`. Die Aussage aus ADR-0003 — „`ProductVariant` übernimmt die
SKU-Identifikation sauber" — bedeutet, dass ATP-Abfragen wahlweise auf `Product`-Ebene oder
`ProductVariant`-Ebene aggregieren können.

**ADR-0009 (Lager-Domänen-Backbone):** `StockBalance` referenziert `Location`; `StockBalance`
wird durch `StockMovement`-Events (ADR-0011) aktuell gehalten. `StockReservation` referenziert
`Location` und `Batch`.

**ADR-0011 (Lagerbewegungen und Ereignis-Log):** `StockMovement`-Events aktualisieren
`StockBalance`-Felder; Zustandsübergänge (z. B. `on_hand → reserved_for_document`) werden
als Events protokolliert.

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `StockReservation` trägt optionalen
FK auf `Batch` für chargengebundene Reservierungen.

## Amendments

### Amendment 2026-05-04 — OQ-0011: Zeitfensterbasierte Verfügbarkeitsfunktion für Miet-ATP

Die skalare ATP-Formel gilt für mengenbasierte Produkte ohne Zeitbindung. Für Produkte mit
`tracking_mode = SERIAL` in Mietanwendungen gilt eine zweite, orthogonale Verfügbarkeitsfrage:
„Ist `SerialUnit` X im Zeitfenster `[start, end]` frei von aktiven `StockReservation`-Einträgen
(`kind ∈ {RENTAL, PROJECT_HOLD}`, `status = ACTIVE`) und aktiven `RentalAssignment`-Einträgen?"

**Abfragevertrag (Query Contract):**

Die zeitfensterbasierte Verfügbarkeitsprüfung wird durch zwei Funktionen spezifiziert:

- `is_free(serial_unit, start, end) -> bool` — gibt `true` zurück, wenn keine aktive
  `StockReservation` (Amendment OQ-0013) und kein aktives `RentalAssignment` den Zeitraum
  `[start, end]` für die gegebene `SerialUnit` schneidet. Zwei Zeitfenster `[a, b]` und
  `[c, d]` schneiden sich, wenn `a < d` und `c < b` gilt (halboffene Intervalle, Grenzwerte
  exklusiv).
- `free_windows(product, start, end) -> list[(serial_unit, [(from, to)])]` — gibt für jede
  `SerialUnit` dieses Produkts die Liste der freien Teilfenster innerhalb `[start, end]`
  zurück, jeweils ausgeschnitten aus dem Anfragefenster durch alle Belegungsblöcke der Einheit.

**Implementierungsstrategie:**

Beide Funktionen werden als berechnete Datenbankabfrage über die kanonische
`StockReservation`-Tabelle (Amendment OQ-0013) realisiert. Eine materialisierte
`UnitAvailabilityWindow`-Tabelle wird nicht eingeführt; falls Abfragemessungen eine
materialisierte Sicht fordern, ist das ein separates operatives Entscheidungsproblem. Die
Abfrage filtert auf `status = ACTIVE` und `kind ∈ {RENTAL, PROJECT_HOLD}` und prüft
Überschneidung mit dem angefragten Zeitfenster. Parallel laufende `RentalAssignment`-Einträge
mit `status ∈ {ACTIVE, OVERDUE}` werden durch dieselbe Abfrage mit einbezogen (Vereinigung
beider Quellen, ohne Doppelzählung — möglich, weil bei Angebotsannahme die
`StockReservation.kind = RENTAL` in ein `RentalAssignment` überführt wird und die
`StockReservation` dabei auf `status = FULFILLED` gesetzt wird; Amendment OQ-0012 und OQ-0013).

Der dedizierte API-Endpunkt `GET /api/products/{id}/serial-units/availability/?start=&end=`
liefert für jede `SerialUnit` des Produkts den Belegungsstatus für den angefragten Zeitraum
(frei / belegt + frühestes Rückgabedatum). Dieser Endpunkt ist die einzige autorisierte
Schnittstelle für zeitfensterbasierte Verfügbarkeitsabfragen; das Frontend (ADR-0002) ruft
ihn vor dem Speichern einer Mietangebotsposition ab.

---

### Amendment 2026-05-04 — OQ-0012: Kopplung Angebots-Lebenszyklus an `StockReservation`-Übergänge

**Zustandsmaschine Angebot → Reservierung:**

| Angebotsstatus          | `StockReservation.status` | `StockReservation.reservation_status` | Semantik                                              |
|-------------------------|---------------------------|---------------------------------------|-------------------------------------------------------|
| `DRAFT`                 | `ACTIVE`                  | `PROVISIONAL`                         | Soft-Block; durch SENT einer anderen Reservierung verdrängbar |
| `SENT`                  | `ACTIVE`                  | `PROVISIONAL`                         | Fester Block (FIFO); konkurrierende Angebote werden abgewiesen |
| `ACCEPTED`              | `ACTIVE`                  | `CONFIRMED`                           | Unveränderlich bis zur physischen Übergabe (RentalAssignment) |
| `REJECTED`              | `CANCELLED`               | —                                     | Reservierung wird synchron freigegeben                |
| `EXPIRED`               | `CANCELLED`               | —                                     | Reservierung wird synchron freigegeben                |
| `CANCELLED`             | `CANCELLED`               | —                                     | Reservierung wird synchron freigegeben                |

`StockReservation` erhält ein neues Feld `reservation_status` (Enum: `PROVISIONAL`,
`CONFIRMED`). Dieses Feld ist additiv; das bestehende Feld `status` (Enum: `ACTIVE`,
`FULFILLED`, `CANCELLED`, `EXPIRED`) bleibt unverändert.

**Konkurrenzregel bei gleichzeitigen Angeboten:**

Zwei oder mehr Angebote können nicht gleichzeitig eine `StockReservation` mit
`reservation_status = PROVISIONAL` und `status = ACTIVE` für dieselbe `SerialUnit` und einen
überschneidenden Zeitraum halten, sobald das erste der konkurrierenden Angebote den Status
`SENT` erreicht. Die Applikationsschicht erzwingt: Der erste eingehende `SENT`-Übergang auf
eine `SerialUnit`/Zeitfenster-Kombination gewinnt (FIFO nach `occurred_at` des
Angebots-Statuswechsels). Jeder nachfolgende `SENT`-Versuch auf dieselbe Einheit und denselben
überschneidenden Zeitraum wird mit HTTP 409 abgewiesen; die Fehlermeldung benennt die
kollidierendeReservierungs-ID und schlägt freie Alternativeinheiten aus `free_windows()` vor.
Ein Angebot im Status `DRAFT` kann eine `PROVISIONAL`-Reservierung halten; das SENT eines
konkurrierenden Angebots auf dieselbe Einheit/denselben Zeitraum setzt die
`DRAFT`-Reservierung nicht zurück — die `DRAFT`-Reservierung bleibt als schwächerer Block
erhalten, wird aber beim nächsten eigenen `SENT`-Versuch mit HTTP 409 abgewiesen. Dieses
Modell implementiert „first-SENT-wins" als einfachste korrekte Semantik.

**Freigabe bei Angebotsrückzug:**

Sobald ein Angebot `REJECTED`, `EXPIRED` oder `CANCELLED` wird, setzt die Applikationsschicht
alle zugehörigen `StockReservation`-Einträge im selben Datenbank-Transaktion auf
`status = CANCELLED`. Die Applikationsschicht emittiert außerdem einen kompensierenden
`StockMovement`-Eintrag mit `business_step = adjustment`, `disposition = null` und
`compensates = FK auf den ursprünglichen Planungs-StockMovement` (ADR-0011). Die freigegebene
`SerialUnit` erscheint im Verfügbarkeitskalender für alle anderen Angebote sofort nach dem
Commit als frei.

---

### Amendment 2026-05-04 — OQ-0013: Vereinheitlichung `StockReservation` und `RentalAssignment`

`StockReservation` ist die einzige autoritative Quelle für die Belegung einer `SerialUnit` in
einem Zeitfenster. `RentalAssignment` bleibt als Spezialisierung erhalten und trägt
mietvertragsspezifische Felder (`rental_start`, `return_due_date`, `returned_at`,
`condition_at_return`); es ist jedoch nicht die Belegungsquelle für den Verfügbarkeitskalender.

**`StockReservation` erhält ein neues Pflichtfeld `kind`:**

| `kind`           | Semantik                                                                   |
|------------------|----------------------------------------------------------------------------|
| `SALE`           | Reservierung für einen Verkaufsauftrag oder ein Verkaufsangebot            |
| `RENTAL`         | Reservierung für ein Mietangebot oder einen Mietvertrag                    |
| `PROJECT_HOLD`   | Manuelle Sperre durch einen Projektverantwortlichen ohne Belegbezug        |

Eine `StockReservation` mit `kind = RENTAL` entsteht synchron beim Speichern einer
Mietangebotsposition (Amendment OQ-0012, `reservation_status = PROVISIONAL`). Bei physischer
Übergabe (Mietbeginn) erzeugt die Applikationsschicht einen `RentalAssignment`-Datensatz,
setzt `StockReservation.status = FULFILLED` und emittiert einen `StockMovement` mit
`disposition = in_possession`. Der Zeitraum zwischen Angebotsannahme (`reservation_status =
CONFIRMED`) und physischer Übergabe ist durch die `StockReservation` mit `kind = RENTAL`,
`status = ACTIVE` und `reservation_status = CONFIRMED` gedeckt — kein zusätzlicher
`RentalAssignment`-Eintrag existiert für diesen Zeitraum.

**Vermeidung von Doppelzählung:**

Der Verfügbarkeitskalender (`is_free()`, `free_windows()`) berücksichtigt ausschließlich
`StockReservation`-Einträge mit `status = ACTIVE`. Ein `RentalAssignment` mit `status =
ACTIVE` blockiert im Kalender nur dann, wenn keine zugehörige `StockReservation` mit
`status = ACTIVE` existiert (d. h. wenn der `RentalAssignment` direkt ohne vorherige
Reservierung entstanden ist — Legacy-Fall). Da im Normalablauf die `StockReservation` auf
`FULFILLED` gesetzt wird, sobald ein `RentalAssignment` angelegt wird, ist Doppelzählung
strukturell ausgeschlossen: die Reservierung existiert mit `status = ACTIVE` oder mit
`status = FULFILLED`, nie beides gleichzeitig für denselben Zeitraum.

**`StockReservation` erhält außerdem ein neues Feld `serial_unit`:**

FK auf `SerialUnit` (ADR-0012, nullable). Dieses Feld wird für `kind = RENTAL` und
`kind = PROJECT_HOLD` auf eine konkrete `SerialUnit` gesetzt; für `kind = SALE` ist es
optional (mengenbezogene Reservierungen ohne Serialnummernbindung tragen `serial_unit = null`).
Die Zeitfensterfelder `rental_start` und `rental_end` werden ebenfalls als nullable Felder
auf `StockReservation` eingeführt, damit die Überschneidungsprüfung (`is_free()`) direkt auf
der Reservierungstabelle ohne Join auf `RentalAssignment` durchführbar ist.

**Migrationsbedeutung:**

Bestehende `RentalAssignment`-Einträge ohne zugehörige `StockReservation` werden bei der
Datenmigration mit einer synthetischen `StockReservation` (`kind = RENTAL`, `status =
FULFILLED`, `reservation_status = CONFIRMED`) retroaktiv verknüpft, damit der
Verfügbarkeitskalender für historische Zeiträume konsistent ist. Diese Migration ist additiv
und löscht keine bestehenden `RentalAssignment`-Zeilen.

## Changelog
- 2026-05-03: Erstentscheidung.
- 2026-05-04: OQ-0011 geschlossen: zeitfensterbasierte Verfügbarkeitsfunktion `is_free(serial_unit, start, end)` und `free_windows(product, start, end)` als berechnete Abfrage über `StockReservation` definiert; dedizierter API-Endpunkt festgelegt. Siehe Amendment 2026-05-04.
- 2026-05-04: OQ-0012 geschlossen: Zustandsmaschine Angebot → `StockReservation` mit `reservation_status`-Feld (`PROVISIONAL`/`CONFIRMED`) und first-SENT-wins-Konkurrenzregel festgelegt. Siehe Amendment 2026-05-04.
- 2026-05-04: OQ-0013 geschlossen: `StockReservation` ist alleinige Belegungsquelle für den Verfügbarkeitskalender; `kind`-Feld (`SALE`, `RENTAL`, `PROJECT_HOLD`), `serial_unit`-FK und Zeitfensterfelder auf `StockReservation` eingeführt; `RentalAssignment` bleibt als Spezialisierung erhalten; Doppelzählungsregel festgelegt; Migrationsvorgehen beschrieben. Siehe Amendment 2026-05-04.
- 2026-07-04: Amendment — `StockBalance` und `StockReservation` FK wechselt von `Product` auf
  `ProductVariant` als autoritativer Schlüssel (ADR-0021); `free_windows()`-Signatur und
  Verfügbarkeits-Endpunkt entsprechend angepasst. Siehe Amendment 2026-07-04.

---

## Amendment 2026-07-04 — `StockBalance` und `StockReservation` → `ProductVariant` als autoritativer Schlüssel (ADR-0021)

ADR-0021 fixiert `ProductVariant` als autoritativen Granularitätsanker für Bestand; das
Amendment 2026-06-28 zu ADR-0009 verschiebt `OnHandRecord` bereits von `Product` auf
`ProductVariant`. `StockBalance` ist der denormalisierte Aggregatssatz über `OnHandRecord`
(„`StockBalance` ist ein denormalisierter Aggregatssatz, der durch `StockMovement`-Events
aktuell gehalten wird", oben) und `StockReservation` bindet Bestandsmengen oder Einzeleinheiten
an Belege; beide tragen bislang denselben dualen FK-Zustand (obligatorischer FK auf `Product`
neben nullable FK auf `ProductVariant`) wie das vor Amendment 2026-06-28 beschriebene
`OnHandRecord`. Da `StockBalance` aus `OnHandRecord`-Zeilen aggregiert und `StockReservation`
dieselbe Bestandseinheit reserviert, muss die Schlüsselung beider Entitäten der bereits
vollzogenen `OnHandRecord`-Umstellung folgen, sonst aggregiert `StockBalance` über eine andere
Granularität als die Zeilen, aus denen es gebildet wird.

### Korrekte Aussage — `StockBalance`

`StockBalance` trägt einen obligatorischen FK auf `ProductVariant` anstelle des bisherigen FK
auf `Product`. Das Produkt ist über den FK-Pfad `StockBalance → ProductVariant → Product`
erreichbar. Der zusammengesetzte Unique-Constraint lautet neu:
`(workspace, variant, location)`. Der Index für ATP-Abfragen wechselt von
`(workspace, product, variant, location)` auf `(workspace, variant, location)`.

Produktweite Bestandsübersichten (Summe über alle Varianten eines Produkts) erfordern eine
GROUP-BY-Aggregation über den FK-Pfad `StockBalance → ProductVariant → Product`; sie sind nicht
mehr über ein direktes FK-Feld auf `StockBalance` abfragbar. Dies entspricht der bereits in
ADR-0009 (Amendment 2026-06-28) für `OnHandRecord` festgelegten Konsequenz.

### Korrekte Aussage — `StockReservation`

`StockReservation` trägt einen obligatorischen FK auf `ProductVariant` anstelle des bisherigen
FK auf `Product`. Das Produkt ist über den FK-Pfad `StockReservation → ProductVariant → Product`
erreichbar. `location`, `batch` und `serial_unit` bleiben unverändert nullable FKs; `batch`
verweist nun auf ein variantengekeytes `Batch` (ADR-0012, Amendment 2026-07-04), `serial_unit`
auf eine variantengekeytes `SerialUnit` (ADR-0012, Amendment 2026-07-04).

### ATP, `is_free()` und `free_windows()`

Die ATP-Formel (`ATP = qty_on_hand − qty_booked − qty_reserved_for_document + qty_ordered`)
bleibt unverändert; sie wird pro `(workspace, variant, location)` statt pro
`(workspace, product, variant, location)` ausgewertet. `is_free(serial_unit, start, end)`
(Amendment 2026-05-04, OQ-0011) bleibt unverändert, da sie bereits auf `SerialUnit` operiert.
`free_windows(product, start, end)` wird zu `free_windows(variant, start, end)`: Da `SerialUnit`
(ADR-0012, Amendment 2026-07-04) auf `ProductVariant` schlüsselt, muss die Verfügbarkeitsabfrage
über die Seriennummern einer Variante iterieren, nicht über die eines abstrakten Produkts, das
mehrere Varianten mit unterschiedlichem `tracking_mode` tragen kann. Der API-Endpunkt wechselt
entsprechend von `GET /api/products/{id}/serial-units/availability/?start=&end=` auf
`GET /api/variants/{id}/serial-units/availability/?start=&end=`.

### Migrationsbedeutung

Die v2.0.0-Migration (REQ-0007) legt die Standardvariante an, auf die bestehende
`StockBalance`- und `StockReservation`-Zeilen bei der Umstellung von `Product` auf
`ProductVariant` verweisen.

ADR-0021 ist die autoritative Quelle für die Schlüsselung; das vorliegende Amendment
dokumentiert die Auswirkung auf ADR-0010.
