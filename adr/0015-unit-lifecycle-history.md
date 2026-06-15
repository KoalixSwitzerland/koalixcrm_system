# ADR-0015: Geräte-Lebenszyklus-Historie

## Status
Accepted

## Context

Individuell verfolgte physische Einheiten (`SerialUnit`, ADR-0012) durchlaufen über ihre
Nutzungszeit hinweg mehr als nur Lagerbewegungen: sie werden in Betrieb genommen,
installiert, deinstalliert, inspiziert, kalibriert, repariert, mit neuer Firmware bespielt
und schließlich außer Betrieb genommen. Werkstatt- und Servicebetriebe (ADR-0013,
`owner_type = CUSTOMER_OWNED`) sowie Mietflottenbetreiber (`owner_type = RENTAL`) benötigen
diese vollständige Einheitenhistorie als einheitliche Abfrageebene. Regulatorische Standards
(Pharmalieferkette, Lebensmittelsicherheit, ISO-Kalibriernachweis) verlangen einen
lückenlosen, unveränderlichen Nachweis über den gesamten Lebenszyklus einer Einheit über
mehrere Jahrzehnte. GS1 EPCIS 2.0 + CBV definiert kanonische Businessstep-Codes, die
Lifecycle-Ereignisse nativ abdecken (`commissioning`, `installing`, `removing`, `inspecting`,
`repairing`, `decommissioning`). Der strukturelle Rahmen der IEC 63278 Asset Administration
Shell (AAS), IEC 62890 und VDI 2770 ist für zukünftige industrielle Interoperabilität
relevant; keine dieser Normen ist heute für KoalixCRM verpflichtend, und kein Kunde hat
bisher eine vollständige AAS-Konformität beauftragt.

## Decision

Die per-`SerialUnit`-Zeitleiste im unveränderlichen `StockMovement`-Log (ADR-0011) ist das
System of Record für die vollständige Lebenszyklus-Historie einer Einheit. Geburt
(Inbetriebnahme), Lagerbewegungen, Installationen, Inspektionen, Reparaturen und
Außerbetriebnahme erscheinen als aufeinander folgende `StockMovement`-Zeilen, die über den
`serial_unit`-FK auf dieselbe `SerialUnit` verweisen. Mengenlose Lebenszyklus-Ereignisse
tragen `qty = null` (ADR-0011); Mengenereignisse tragen einen Wert in `qty`. Das As-Built-BOM
einer `SerialUnit` ergibt sich aus `AGGREGATION_EVENT`-Zeilen (ADR-0014), die beim
Fertigungsabschluss emittiert werden. Aktuell installierte Komponenten einer Einheit
ergeben sich aus der Traversierung des `AGGREGATION_EVENT`-Graphen: eine Komponente gilt
als installiert, wenn ein `business_step = 'installing'`-Ereignis auf sie verweist und kein
nachfolgendes `business_step = 'removing'`-Ereignis mit demselben `serial_unit`-FK existiert.

Das Design der Lebenszyklus-Felder und Abfragepfade orientiert sich an der AAS
`ServiceAndMaintenance`-Semantik (IEC 63278) als Gestaltungslinse — nicht als
verpflichtender Standard. Strukturelle Kompatibilität mit AAS auf Feldebene ermöglicht es,
einen Tier-1-AASX-Serialisierer (Datenexport in AASX-Format) nachzurüsten, ohne
Schema-Änderungen vorzunehmen. Tier 2 (schreibende AAS-API) und Tier 3 (vollständige
AAS-Konformität) liegen außerhalb des Geltungsbereichs dieses ADR und werden erst
umgesetzt, wenn ein zahlender Kunde diese Anforderung stellt.

## Why

Den `StockMovement`-Log als einzige Quelle der Lebenszyklus-Wahrheit zu verwenden — statt
einen parallelen Lebenszyklus-Event-Log einzuführen — verhindert das Auseinanderdriften
zweier Zeithorizonte und erlaubt, die vollständige Einheitenhistorie mit einer einzigen
Abfrage auf einer einzigen Tabelle zu rekonstruieren. EPCIS 2.0 unterstützt mengenlose
Lifecycle-Events nativ; die CBV-Businessstep-Codes für Lifecycle-Ereignisse sind bereits
in ADR-0011 verankert, sodass kein neues Ereignismodell erforderlich ist.

## Alternatives Considered

- **Separater Lebenszyklus-Ereignis-Log parallel zum `StockMovement`-Log** — abgelehnt:
  spaltet die Einheitenhistorie auf zwei Tabellen auf; Abfragen der vollständigen Zeitleiste
  erfordern UNION oder JOIN über zwei wachsende Tabellen; Konsistenzgarantien zwischen den
  Logs (Reihenfolge, Duplikate) müssen doppelt erzwungen werden.
- **Freitext-`notes`-JSONB auf `SerialUnit` für Lebenszyklus-Aufzeichnungen** — abgelehnt:
  nicht strukturiert abfragbar; nicht auditierbar (JSONB kann nachträglich überschrieben
  werden); kein maschinenlesbarer Businessstep-Code; keine EPCIS-Exportfähigkeit.
- **Vollständige AAS-Implementierung (Tier 2 und Tier 3) jetzt** — abgelehnt: die vollständige
  AAS-Konformität umfasst eine schreibende API, einen Submodel-Registry-Service und einen
  AASX-Serialisierer; der Implementierungsaufwand beträgt schätzungsweise 6–12 Monate für
  vollständige Tier-3-Konformität; kein aktueller Kunde hat diese Anforderung gestellt; der
  ROI rechtfertigt den Aufwand nicht. AAS-Readiness als Gestaltungslinse sichert die
  Optionalität ohne Vorleistungskosten.

## Consequences

### Positive
- Die vollständige Lebenszyklus-Historie einer `SerialUnit` — von der Inbetriebnahme bis zur
  Außerbetriebnahme — ist durch eine einzige Abfrage auf dem `StockMovement`-Log
  rekonstruierbar, gefiltert nach `serial_unit_id` und sortiert nach `occurred_at`.
- Das As-Built-BOM entsteht aus `AGGREGATION_EVENT`-Zeilen (ADR-0014) im selben Log; keine
  separate As-Built-Tabelle ist notwendig.
- Aktuell installierte Komponenten einer Einheit ergeben sich aus dem `AGGREGATION_EVENT`-Graph
  durch Traversierung der `installing`/`removing`-Paare; kein zusätzliches Zustandsfeld auf
  `SerialUnit` wird benötigt.
- AAS-Readiness als Gestaltungslinse ermöglicht einen späteren Tier-1-AASX-Serialisierer
  ohne Schema-Änderungen; die Option auf industrielle Interoperabilität bleibt offen.
- Die Aufbewahrungsuntergrenze (ADR-0011, ADR-0012) schützt die Lebenszyklus-Historie gegen
  vorzeitiges Löschen und erfüllt jahrzehntlange regulatorische Aufbewahrungspflichten.

### Negative
- Abfragen der vollständigen Einheitenhistorie traversieren den gesamten
  `StockMovement`-Log-Bereich für eine `SerialUnit`; bei sehr langen Lebensdauern (> 20 Jahre)
  und häufigen Ereignissen kann die Anzahl der Zeilen pro Einheit groß werden. PostgreSQL
  Range-Partitionierung nach `occurred_at` (ADR-0011) hält Abfragen performant.
- Die Graph-Traversierung für installierte Komponenten (Auswertung von
  `installing`/`removing`-Paaren) ist eine Applikationslogik-Aufgabe; das Datenbankschema
  allein liefert keinen aktuellen Installationsstatus ohne Auswertung des Logs.
- AAS-Readiness als Gestaltungslinse ist kein Konformitätsnachweis; Kunden, die eine
  vollständige AAS-Zertifizierung verlangen, erhalten diese nicht durch dieses ADR.

---

## Lebenszyklus-Abfragepfade

**Vollständige Einheitenhistorie:** Alle `StockMovement`-Zeilen mit
`serial_unit_id = <id>`, geordnet nach `occurred_at`. Ergibt die vollständige Zeitleiste
von Inbetriebnahme bis Außerbetriebnahme (oder aktueller Zeit bei aktiver Einheit).

**As-Built-BOM:** Alle `StockMovement`-Zeilen mit `event_type = AGGREGATION_EVENT` und
`serial_unit_id = <id>` (Elternrolle). Die Kind-FKs ergeben die Komponentenzusammensetzung
zum Fertigungszeitpunkt.

**Aktuell installierte Komponenten:** Traversierung des `AGGREGATION_EVENT`-Graphen:
für jede Kind-`SerialUnit`, die in einem `business_step = 'installing'`-Ereignis mit dem
Elternteil verknüpft ist, prüft die Abfrage, ob ein nachfolgendes
`business_step = 'removing'`-Ereignis mit demselben Kind existiert. Existiert kein solches
Entfernungsereignis, gilt die Komponente als aktuell installiert.

**Halter-Zeitleiste (`who_held_it_when`):** Projektion über alle `StockMovement`-Zeilen
mit `serial_unit_id = <id>` und `disposition ∈ {in_possession, returned}` (ADR-0011,
Amendment OQ-0010), sortiert aufsteigend nach `occurred_at`. Die Projektion gruppiert
aufeinanderfolgende Ereignisse nach `owner_party` zu geschlossenen Haltezeitfenstern.
Ein Haltezeitfenster beginnt mit dem ersten `StockMovement`-Eintrag, bei dem
`disposition = in_possession` und `owner_party = <Halter>`, und endet mit dem ersten
nachfolgenden Eintrag, bei dem `disposition = returned` für dieselbe Einheit. Das Ergebnis
ist eine geordnete Liste von Tupeln `(holder_party, from, to)`, wobei `to = null` für
die aktuell haltende Partei gilt (Einheit noch nicht zurückgegeben).

`who_held_it_when(serial_unit) -> list[(holder_party, from, to)]`

Diese Abfrage ist der dedizierte API-Endpunkt `GET /api/serial-units/{id}/holder-timeline/`
und liefert die vollständige Halterhistorie einer Einheit. Sie ergänzt den Pfad
„Vollständige Einheitenhistorie" ohne diesen zu ersetzen: die Halter-Zeitleiste ist eine
auf Dispositionssemantik gefilterte Projektion; die vollständige Einheitenhistorie enthält
alle Ereignistypen einschließlich Inspektionen, Reparaturen und Mengenbewegungen.

**Standort-Zeitleiste (`where_was_it_when`):** Projektion über alle `StockMovement`-Zeilen
mit `serial_unit_id = <id>`, geordnet aufsteigend nach `occurred_at`. Die Projektion
gruppiert aufeinanderfolgende Zeilen nach `destination_location`: ein neues Zeitfenster
beginnt, wenn der Wert von `destination_location` gegenüber dem vorangehenden Eintrag
wechselt. Das Ergebnis ist eine geordnete Liste von Tupeln `(location, from, to)`, wobei
`to = null` für den aktuellen Standort gilt (Einheit noch nicht weiter bewegt). Zeilen ohne
`destination_location` (mengenlose Lebenszyklus-Events wie `inspecting` an einem implizit
bekannten Ort) unterbrechen das laufende Zeitfenster nicht; die Projektion ignoriert
`destination_location = null`-Einträge für die Segmentierung.

`where_was_it_when(serial_unit) -> list[(location, from, to)]`

Diese Abfrage ist der dedizierte API-Endpunkt `GET /api/serial-units/{id}/location-timeline/`
und liefert die vollständige Standorthistorie einer Einheit (UC-0008, UC-0009, UC-0010). Sie
spiegelt strukturell den `who_held_it_when`-Pfad: beide sind Projektionen über denselben
`StockMovement`-Log, gefiltert nach verschiedenen Feldern. Die Standort-Zeitleiste ist eine
auf `destination_location`-Wechseln segmentierte Projektion; sie zeigt, wann eine Einheit
wo war, unabhängig davon, wer sie hielt.

---

## CBV-Businessstep-Vocabulary für Lifecycle-Events

Die folgenden Businessstep-Codes aus GS1 EPCIS 2.0 CBV sind die kanonischen Werte für
Lebenszyklus-Ereignisse und werden in der `business_step`-Spalte des `StockMovement`-Logs
(ADR-0011) gespeichert:

| `business_step`    | Semantik                                                                 |
|--------------------|--------------------------------------------------------------------------|
| `commissioning`    | Einheit wird erstmals in Betrieb genommen (Geburtsereignis)              |
| `installing`       | Einheit wird in eine übergeordnete Baugruppe eingebaut                   |
| `removing`         | Einheit wird aus einer übergeordneten Baugruppe ausgebaut                |
| `inspecting`       | Einheit wird geprüft oder kalibriert (mengenlos, `qty = null`)           |
| `repairing`        | Einheit befindet sich in oder verlässt den Reparaturprozess              |
| `decommissioning`  | Einheit wird dauerhaft außer Betrieb genommen                            |

---

## AAS-Readiness-Gestaltungslinse

Die `StockMovement`-Felder `event_type`, `business_step`, `serial_unit`, `occurred_at`,
`read_point` und `biz_location` bilden strukturell auf die AAS-Submodel `ServiceAndMaintenance`
(IEC 63278) ab. Ein Tier-1-AASX-Serialisierer kann diese Felder ohne Datenbankschema-Änderung
auf AASX-Dokumente projizieren. IEC 62890 (Lebenszyklusmanagement für Mess- und
Automatisierungstechnik) und VDI 2770 (Dokumentationsformat für Anlagen-Assets) sind als
zukünftig kompatible Standards gelistet; keine dieser Normen ist ein verpflichtender
Implementierungsrahmen für dieses ADR.

---

## Workspace-Scoping-Matrix

| Entität / Artefakt              | Scoping   | Begründung                                                                    |
|---------------------------------|-----------|-------------------------------------------------------------------------------|
| `StockMovement` (Lifecycle)     | workspace | Lebenszyklus-Events sind mandantenspezifische Anlagendaten (ADR-0011)         |
| `SerialUnit`                    | workspace | Einheiten sind mandantenspezifische Anlagendaten (ADR-0012)                   |
| Lebenszyklus-Abfragepfade       | workspace | Abfragen traversieren den mandantenspezifischen `StockMovement`-Log           |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel
und Docker-Image ausgeliefert wird. GS1 EPCIS 2.0 ist ein offener GS1-Standard; die
Businessstep-Codes sind frei verwendbar. IEC 63278, IEC 62890 und VDI 2770 werden nur als
Gestaltungslinse referenziert, nicht implementiert; keine kommerziellen Lizenzen dieser
Normengremien sind erforderlich, um das Datenbankschema in dieser Form zu betreiben. Das
REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige Kommunikationsbrücke zwischen
dem Open-Source-Backend und dem geschlossenen Next.js-Frontend.

---

## Standards-Verankerung

| Standard                         | Verwendung in diesem ADR                                                                |
|----------------------------------|-----------------------------------------------------------------------------------------|
| GS1 EPCIS 2.0 + CBV              | Kanonische Businessstep-Codes für Lifecycle-Events; `event_type`-Enum (ADR-0011)       |
| IEC 63278 Asset Administration Shell | Gestaltungslinse: strukturelle Kompatibilität für zukünftigen Tier-1-AASX-Export   |
| IEC 62890                        | Referenz: Lebenszyklusmanagement für Mess- und Automatisierungstechnik (future-fit)     |
| VDI 2770                         | Referenz: Dokumentationsformat für Anlagen-Assets (future-fit)                          |
| ISO/IEC 15459                    | `SerialUnit.global_uid` (ADR-0012) als eindeutiger Einheitenidentifikator               |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben
`WorkspaceScopedModel`. `owner_party`-Verweise auf `Party` bei Miet- und Kundengeführtem
Bestand (ADR-0013) sind Teil der Lebenszyklus-Kontextualisierung.

**ADR-0002 (Admin-UI-Framework):** Lebenszyklus-Abfragen sind über read-only DRF-Endpunkte
exponiert; Schreibzugriff erfolgt ausschließlich über dedizierte Ereignis-Endpunkte, nicht
via generisches PATCH/PUT (ADR-0002).

**ADR-0003 (Produkt-Katalog-Backbone):** Jede `SerialUnit` trägt einen FK auf `Product`
(ADR-0003). ADR-0003 definiert: „`ProductType` wird zu `Product` umbenannt und übernimmt
die Rolle des kanonischen Katalogobjekts." `lifecycle_status = EXTERNAL_ONLY` (ADR-0003)
ermöglicht Lebenszyklus-Verwaltung für Fremdeinheiten ohne vollständigen Katalogstammsatz.

**ADR-0008 (Digital Product Passport):** Strukturierte DPP-Felder entstehen als Projektion
über den Ereignis-Log; die Lebenszyklus-Abfragepfade aus diesem ADR liefern die Datengrundlage.
ADR-0008 definiert: „Strukturierte DPP-Felder entstehen als Projektion (materialisierte Ansicht
oder berechneter Serialisierer) über den Ereignis-Log (ADR-0011), die Produktstammdaten
(ADR-0003) und die Klassifizierungsdaten (ADR-0004)."

**ADR-0011 (Lager- und Lebenszyklus-Ereignis-Log):** Der `StockMovement`-Log ist das
unveränderliche Fundament der Lebenszyklus-Historie. ADR-0011 definiert: „Jede Lagerbewegung
und jeder Lebenszyklus-Touch einer Einheit wird als unveränderlicher `StockMovement`-Datensatz
gespeichert." Die Aufbewahrungsuntergrenze aus ADR-0011 schützt die Lebenszyklus-Historie
gegen vorzeitiges Löschen.

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `SerialUnit` ist das Datenobjekt,
dessen Lebenszyklus-Historie dieses ADR definiert. ADR-0012 definiert: „Für
`Product.tracking_mode = SERIAL` wird eine `SerialUnit`-Entität eingeführt, die eine
eindeutige Seriennummer, einen ISO/IEC-15459-kompatiblen eindeutigen Identifikator,
Produktionsdatum, Garantieablauf und aktuellen Betriebszustand trägt." Die Soft-Delete-Strategie
und Aufbewahrungsuntergrenze aus ADR-0012 ermöglichen die jahrzehntlange Traceability.

**ADR-0014 (Montage/Kitting und geteilter Bestand):** `AGGREGATION_EVENT`-Zeilen, die bei
Fertigungsabschluss emittiert werden (ADR-0014), sind die Datengrundlage für das As-Built-BOM
und die Komponentengraph-Traversierung. ADR-0014 definiert: „Bei Fertigungsabschluss MUSS die
Applikationsschicht ein oder mehrere `AGGREGATION_EVENT`-Einträge emittieren, die das
As-Built-BOM erfassen."

## Amendments

### Amendment 2026-05-04 — OQ-0014: Halter-Zeitleiste als vierter Lebenszyklus-Abfragepfad

Der vierte definierte Abfragepfad „Halter-Zeitleiste" (`who_held_it_when`) ist als Projektion
über `StockMovement`-Zeilen mit `disposition ∈ {in_possession, returned}` (ADR-0011,
Amendment OQ-0010) definiert. Die vollständige Spezifikation dieses Abfragepfads befindet sich
im Abschnitt „Lebenszyklus-Abfragepfade" dieses ADR. Die Abfrage ist idempotent und
unveränderlich gegenüber dem Log; sie schreibt keine Daten. Der API-Endpunkt
`GET /api/serial-units/{id}/holder-timeline/` ist der einzige autorisierte Auslieferungspunkt
für diese Projektion; das Frontend (ADR-0002) zeigt die Halterhistorie im
Verfügbarkeitskalender für die Rückgabehistorie an. Die Ergebnisliste ist aufsteigend nach
`from` sortiert und schließt alle Halter seit dem ersten `StockMovement` mit
`disposition = in_possession` ein, auch wenn der erste Halter der Mandant selbst ist
(Testphase, Eigennutzung vor Erstverleih).

### Amendment 2026-05-04 — UC-0008/0009/0010: Standort-Zeitleiste als fünfter Lebenszyklus-Abfragepfad

UC-0008 (Lagerbestand per Barcode lokalisieren), UC-0009 (Komponentenentnahme mit
Bestandsbestätigung) und UC-0010 (Wareneingang mit Lieferschein und Lagerplatzvorschlag)
erfordern, nachvollziehen zu können, an welchem Standort eine `SerialUnit` zu welchem Zeitpunkt
war. Dieser Informationsbedarf entspricht strukturell der `who_held_it_when`-Projektion, richtet
sich jedoch nach `destination_location` statt nach `owner_party`.

Der fünfte definierte Abfragepfad „Standort-Zeitleiste" (`where_was_it_when`) ist als Projektion
über alle `StockMovement`-Zeilen für eine `SerialUnit`, segmentiert nach Wechseln des Feldes
`destination_location`, definiert. Die vollständige Spezifikation befindet sich im Abschnitt
„Lebenszyklus-Abfragepfade" dieses ADR. Der API-Endpunkt
`GET /api/serial-units/{id}/location-timeline/` ist der einzige autorisierte Auslieferungspunkt
für diese Projektion. Kein neues Datenbankschema ist erforderlich; der bestehende
`StockMovement`-Log (ADR-0011) ist die einzige Datenquelle.

Lizenzbeschränkung: Keine neuen Closed-Source-Bausteine.

## Changelog
- 2026-05-03: Erstentscheidung.
- 2026-05-04: OQ-0014 geschlossen: vierter Lebenszyklus-Abfragepfad „Halter-Zeitleiste" (`who_held_it_when(serial_unit) -> list[(holder_party, from, to)]`) als Projektion über `StockMovement.disposition ∈ {in_possession, returned}` definiert; dedizierter API-Endpunkt `GET /api/serial-units/{id}/holder-timeline/` festgelegt. Siehe Amendment 2026-05-04.
- 2026-05-04: UC-0008/0009/0010: fünfter Lebenszyklus-Abfragepfad „Standort-Zeitleiste" (`where_was_it_when(serial_unit) -> list[(location, from, to)]`) als Projektion über `StockMovement.destination_location`-Wechsel definiert; dedizierter API-Endpunkt `GET /api/serial-units/{id}/location-timeline/` festgelegt. Siehe Amendment 2026-05-04 (UC-0008/0009/0010).
