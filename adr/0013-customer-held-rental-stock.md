# ADR-0013: Miet- und Kundengeführter Bestand

## Status
Accepted

## Context

KoalixCRM unterstützt Mandanten, die Maschinen, Fahrzeugteile oder andere Anlagegüter an Kunden
vermieten. Diese Einheiten sind physisch beim Kunden, bleiben aber Eigentum des Mandanten und
unterliegen einer Rückgabeverpflichtung. Ergänzend unterstützt KoalixCRM Werkstatt- und
Servicebetriebe, bei denen Kunden Einheiten zur Reparatur einliefern: die Einheit verbleibt
physisch im Betrieb des Mandanten, gehört aber dem Kunden (Eigentümer und Halter sind
vertauscht, jedoch umgekehrt zur Mietsituation). Das bestehende Modell kennt diese
Eigentumstrennungen nicht: Eine versendete oder eingegangene Einheit gilt schlicht als
„ausgeliefert/verkauft" oder „auf Lager". Für Miet- und Servicebetrieb müssen folgende
Sachverhalte im Bestandsmodell unterscheidbar sein: (1) Eigentümer und physischer Halter
können verschiedene Parteien sein; (2) der physische Standort ist eine Kunden- oder
Unternehmensadresse; (3) die Einheit hat einen Betriebszustand (neu / gebraucht / beschädigt
/ in Reparatur); (4) ein Vertragsbeleg (Miet- oder Servicevertrag) bindet die Einheit;
(5) die Rückgabeverpflichtung und ihr Fälligkeitsdatum sind abrufbar. Diese Anforderungen
betreffen ausschließlich das Lagerdatenmodell; das Vertragsmodell selbst ist ein
Commercial-Document-Domänenproblem und wird in diesem ADR nicht definiert.

## Decision

Miet- und kundengeführter Bestand wird nicht als separates Modell eingeführt, sondern durch
Kombination bestehender Lagerentitäten abgebildet: `OnHandRecord.owner_type` kennzeichnet die
Eigentümerschaft, `OnHandRecord.owner_party` verweist auf die relevante Gegenpartei (FK auf
`Party` aus ADR-0001), `Location.location_type = CUSTOMER_SITE` kennzeichnet Kundenstandorte,
`SerialUnit.condition_state` (ADR-0012) trägt den Betriebszustand, und eine dedizierte
`RentalAssignment`-Entität bindet den `OnHandRecord` an einen Vertragsbeleg (generische
Dokumentreferenz) und trägt Rückgabefälligkeitsdatum und Rückgabeverpflichtung.

`OnHandRecord.owner_type` kennt die folgenden Werte:

- `OWN` — Unternehmen ist Eigentümer; Einheit befindet sich am Unternehmensstandort.
- `RENTAL` — Unternehmen ist Eigentümer; Einheit befindet sich physisch beim Kunden (Miet-Fall:
  wir besitzen, Kunde hält).
- `CUSTOMER_CONSIGNMENT` — Kunde ist Eigentümer; Einheit befindet sich im Unternehmensbestand
  (Kommissions-Fall: Kunde besitzt, wir halten ohne Rückgabepflicht).
- `CUSTOMER_OWNED` — Kunde ist Eigentümer; Einheit befindet sich zur Bearbeitung physisch beim
  Unternehmen (Werkstatt-/Servicefall: Kunde besitzt, wir halten mit Rückgabepflicht nach
  Serviceabschluss).

Sowohl `RENTAL` als auch `CUSTOMER_OWNED` stellen Eigentümer-Halter-Trennungen dar; die
Richtung der Trennung ist umgekehrt. Beide gehen nicht in ATP ein.

## Why

Die Wiederverwendung von `OnHandRecord.owner_type`, `Location.location_type = CUSTOMER_SITE`
und `SerialUnit.condition_state` — statt separater Miet- oder Servicebestandsmodelle — hält den
Bestand in einer einzigen Abfrageebene für alle Eigentumsverhältnisse; ATP-Abfragen (ADR-0010)
filtern Fremdbestand durch `owner_type`-Constraint aus, ohne eine zweite Datenhaltungsebene zu
traversieren; der Ereignis-Log (ADR-0011) protokolliert alle Eigentumsübergänge als
`StockMovement`-Events ohne gesonderte Event-Tabelle. Ein vierter `owner_type`-Wert
(`CUSTOMER_OWNED`) für den Werkstatt-/Servicefall — statt eines separaten Modells — bewahrt
das Prinzip der einzigen Abfrageebene und vermeidet eine Modellverdoppelung für einen
Anwendungsfall, der strukturell identisch mit `RENTAL` ist (Eigentümer-Halter-Trennung),
aber mit vertauschter Richtung.

## Alternatives Considered

- **Mietbestand als „versandt/verkauft" verbuchen (keine Rückkehr erwartet)** — abgelehnt:
  vermiete Einheiten verschwinden aus dem Bestand des Mandanten; Rückgabeverpflichtung und
  Standort der Einheit sind nicht abrufbar; Inventurbewertung ist falsch (Anlage wird
  abgeschrieben statt als umlaufendes Anlagegut geführt).
- **Separates `RentalInventory`-Modell parallel zu `OnHandRecord`** — abgelehnt: dupliziert
  Mengen- und Standortinformationen; ATP-Abfragen müssen zwei Tabellen zusammenführen;
  `StockMovement`-Log (ADR-0011) müsste zwei Zieltabellen bedienen, was die
  Audit-Trail-Integrität gefährdet.
- **Mietbestand ausschließlich über den Mietvertrag (Commercial Document) verwalten, ohne
  Lagerbuchung** — abgelehnt: der physische Standort und der Betriebszustand der Einheit
  sind Lagerfragen, keine Vertragsfragen; Inventurerfassung und Standortabfragen sind ohne
  Lagereintrag nicht möglich.

## Consequences

### Positive
- Mietbestand erscheint in denselben Bestandsabfragen wie eigener Bestand; ein einziger
  API-Endpunkt liefert den vollständigen Überblick, gefiltert nach `owner_type`.
- `RentalAssignment.return_due_date` ermöglicht Fälligkeitsberichte ohne Beziehungstraversierung
  über das Mietvertragsmodell.
- `SerialUnit.condition_state` (ADR-0012) ist die einzige Quelle des Betriebszustands; keine
  Verdoppelung zwischen Lager- und Vertragsmodul.
- Mietausgabe und Mietrückgabe sind als `StockMovement`-Events mit `business_step = 'rental_out'`
  und `'rental_return'` vollständig auditierbar (ADR-0011).

### Negative
- `RentalAssignment` trägt eine generische Dokumentreferenz auf den Mietvertrag; die
  Applikationsschicht muss die Konsistenz zwischen `StockMovement`-Status und
  `RentalAssignment.status` sicherstellen, da kein Datenbankconstraint diese Beziehung
  erzwingt.
- `Location.location_type = CUSTOMER_SITE`-Einträge wachsen mit jedem neuen Kunden;
  eine Standorthygiene (Deaktivierung nach Mietvertragsende) ist als betrieblicher Prozess
  notwendig.
- Der Übergang `RENTAL → OWN` (Rückkauf durch Kunde oder Übernahme in Eigenbestand nach
  Rückgabe) erfordert einen expliziten `StockMovement`-Event mit Eigentümerwechsel; die
  Applikationsschicht muss diesen Übergang aktiv anstoßen.

---

## Entitäten

**`Location`** (ADR-0009, Erweiterung) — `location_type = CUSTOMER_SITE` kennzeichnet
physische Kundenstandorte. `Location.owner_party` (nullable FK auf `Party`, ADR-0001) verweist
auf den Kunden als Betreiber des Standorts.

**`RentalAssignment`** (workspace-scoped) — Bindet eine `SerialUnit` oder eine
`OnHandRecord`-Zeile an einen aktiven Mietvertrag.
Felder: FK auf `SerialUnit` (ADR-0012, nullable), FK auf `OnHandRecord` (ADR-0009, nullable),
FK auf `Party` (ADR-0001, Mieter), `document_type` (Django ContentType, Mietvertragsbeleg),
`document_id` (PositiveIntegerField), `rental_start` (Datetime), `return_due_date`
(Date, nullable), `returned_at` (Datetime, nullable), `status` (Enum: `ACTIVE`, `RETURNED`,
`OVERDUE`, `WRITTEN_OFF`), `condition_at_return` (Enum: `NEW`, `USED`, `DAMAGED`, `IN_REPAIR`,
nullable).

---

## Eigentumssemantik

| `owner_type`            | Eigentümer     | Physischer Standort       | Rückgabepflicht | In ATP (ADR-0010)       |
|-------------------------|----------------|---------------------------|-----------------|--------------------------|
| `OWN`                   | Unternehmen    | Unternehmensstandort      | Nein            | Ja                       |
| `RENTAL`                | Unternehmen    | Kundenstandort            | Ja              | Nein (ausgeschlossen)    |
| `CUSTOMER_CONSIGNMENT`  | Kunde          | Unternehmensstandort      | Nein            | Nein (Fremdbestand)      |
| `CUSTOMER_OWNED`        | Kunde          | Unternehmensstandort      | Ja (nach Service) | Nein (Fremdbestand)   |

Mietbestand (`RENTAL`) geht nicht in ATP ein, da er physisch nicht verfügbar ist.
Fremdbestand (`CUSTOMER_CONSIGNMENT` und `CUSTOMER_OWNED`) geht nicht in ATP ein, da er nicht
im Eigentum des Mandanten steht. Der Unterschied zwischen `CUSTOMER_CONSIGNMENT` und
`CUSTOMER_OWNED`: bei `CUSTOMER_CONSIGNMENT` hält das Unternehmen den Bestand ohne
Rückgabeverpflichtung (Kommission); bei `CUSTOMER_OWNED` hält das Unternehmen die Einheit
zur Bearbeitung und muss sie nach Serviceabschluss zurückgeben (Werkstattfall).

---

## Workspace-Scoping-Matrix

| Entität            | Scoping   | Begründung                                                             |
|--------------------|-----------|------------------------------------------------------------------------|
| `RentalAssignment` | workspace | Mietverträge und Rückgabeverpflichtungen sind Mandantendaten           |
| `Location` (CUSTOMER_SITE) | workspace | Kundenlagerplätze referenzieren workspace-eigene `Party`-Einträge |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`+`WorkspaceScopedViewSetMixin`-Mechanismus
aus ADR-0001.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird. Es enthält keinen Quantalq-proprietären Inhalt. Die generische
Mietvertragsreferenz (`document_type` + `document_id`) vermeidet eine Importabhängigkeit auf
ein geschlossenes Vertragsmodell. Das REST-API-Integrationsprotokoll (ADR-0002) bleibt die
einzige Kommunikationsbrücke zum Frontend.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** `RentalAssignment.party` referenziert `Party`
(ADR-0001) als Mieter. Die Aussage aus ADR-0001: „`Party.id` ist der lang-lebige Geschäftsschlüssel
für alles Dokumenten-bezogene" gilt für `RentalAssignment.party` direkt.
`Location.owner_party` referenziert `Party` für Kundenstandorte.

**ADR-0002 (Admin-UI-Framework):** `RentalAssignment` ist über DRF-Endpunkte exponiert; keine
direkte Modell-Referenz im Frontend.

**ADR-0009 (Lager-Domänen-Backbone):** `OnHandRecord.owner_type = RENTAL` und
`owner_party` (FK auf `Party`) sind die Datenbankgrundlage. `Location.location_type =
CUSTOMER_SITE` kennzeichnet Kundenstandorte.

**ADR-0010 (Lagerbestandszustände und Reservierungen):** Mietbestand (`owner_type = RENTAL`)
geht nicht in ATP ein.

**ADR-0011 (Lagerbewegungen und Ereignis-Log):** `business_step = 'rental_out'` und
`'rental_return'` sind die Event-Typen für Mietbewegungen.

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `SerialUnit.condition_state` trägt
den Betriebszustand der Mieteinheit oder Fremdeinheit; `RentalAssignment.condition_at_return`
spiegelt diesen Zustand zum Zeitpunkt der Rückgabe.

## Amendments

### Amendment 2026-05-04 — OQ-0013: `RentalAssignment` als Spezialisierung von `StockReservation`

`StockReservation` (ADR-0010, Amendment OQ-0013) ist die alleinige autoritative Belegungsquelle
für den Verfügbarkeitskalender einer `SerialUnit`. `RentalAssignment` bleibt als eigenständige
Entität erhalten, übernimmt jedoch nicht mehr die Rolle der primären Belegungsquelle.

`RentalAssignment` ist die mietvertragsspezifische Spezialisierung: Es entsteht bei physischer
Übergabe der Einheit an den Mieter und trägt die betrieblichen Felder, die nach dem
physischen Mietbeginn relevant sind (`returned_at`, `condition_at_return`, `status ∈ {ACTIVE,
RETURNED, OVERDUE, WRITTEN_OFF}`). Jeder `RentalAssignment`-Datensatz verweist auf genau
eine `StockReservation` mit `kind = RENTAL` und `status = FULFILLED`.

**Überarbeitete `RentalAssignment`-Felder:**

FK auf `SerialUnit` (ADR-0012, nullable), FK auf `OnHandRecord` (ADR-0009, nullable),
FK auf `StockReservation` (ADR-0010) — Pflichtfeld; verweist auf die erfüllte Reservierung,
FK auf `Party` (ADR-0001, Mieter), `document_type` (Django ContentType, Mietvertragsbeleg),
`document_id` (PositiveIntegerField), `rental_start` (Datetime — physisches Übergabedatum),
`return_due_date` (Date, nullable), `returned_at` (Datetime, nullable),
`status` (Enum: `ACTIVE`, `RETURNED`, `OVERDUE`, `WRITTEN_OFF`),
`condition_at_return` (Enum: `NEW`, `USED`, `DAMAGED`, `IN_REPAIR`, nullable).

Das Feld `rental_start` auf `RentalAssignment` bezeichnet den Zeitpunkt der physischen
Übergabe; das gleichnamige Feld auf `StockReservation` bezeichnet den geplanten Mietbeginn.
Beide Felder können voneinander abweichen (Frühübergabe, Spätübergabe).

`RentalAssignment.status = OVERDUE` wird gesetzt, wenn `return_due_date` überschritten ist
und kein `returned_at` vorliegt. Das Backend blockiert neue `StockReservation`-Einträge für
dieselbe `SerialUnit` ab `return_due_date` bis zum Schreiben eines `StockMovement` mit
`business_step = rental_return`, der `returned_at` setzt und `RentalAssignment.status =
RETURNED` überführt.

**Keine Doppelzählung im Verfügbarkeitskalender:**

Der Verfügbarkeitskalender liest ausschließlich `StockReservation.status = ACTIVE`. Sobald
ein `RentalAssignment` angelegt wird, wird die zugehörige `StockReservation` auf
`status = FULFILLED` gesetzt. Ein `RentalAssignment` mit `status ∈ {ACTIVE, OVERDUE}` ohne
zugehörige aktive `StockReservation` blockiert den Kalender als Fallback-Quelle; dies ist
nur im Legacy-Migrationszustand möglich (Amendment OQ-0013, ADR-0010).

## Changelog
- 2026-05-03: Erstentscheidung.
- 2026-05-03: `owner_type = CUSTOMER_OWNED` als vierter Enum-Wert ergänzt (Werkstatt-/
  Servicefall: Kunde besitzt, Unternehmen hält zur Bearbeitung). Titel auf „Miet- und
  Kundengeführter Bestand" aktualisiert. Eigentumssemantik-Tabelle um `CUSTOMER_OWNED`
  erweitert.
- 2026-05-04: OQ-0013 geschlossen: `RentalAssignment` als Spezialisierung von `StockReservation` positioniert; `StockReservation` ist alleinige Belegungsquelle für den Verfügbarkeitskalender; neues Pflichtfeld `FK auf StockReservation` auf `RentalAssignment` eingeführt; Überdue-Blockierungsregel und Doppelzählungsschutz festgelegt. Siehe Amendment 2026-05-04.
- 2026-07-04: Amendment — Klarstellung: `RentalAssignment` trägt keinen direkten FK auf
  `Product` und benötigt daher kein eigenes Feld-Amendment zu ADR-0021; die
  Variantenschlüsselung wirkt transitiv über `SerialUnit` (ADR-0012, Amendment 2026-07-04) und
  `StockReservation` (ADR-0010, Amendment 2026-07-04). Siehe Amendment 2026-07-04.

---

## Amendment 2026-07-04 — Transitive Variantenschlüsselung über `SerialUnit` und `StockReservation` (ADR-0021)

ADR-0021 verschiebt die autoritative Lager-Schlüsselung von `Product` auf `ProductVariant`; die
Amendments 2026-06-28 (ADR-0009, `OnHandRecord`), 2026-07-04 (ADR-0012, `Batch`/`SerialUnit`)
und 2026-07-04 (ADR-0010, `StockBalance`/`StockReservation`) vollziehen diese Umstellung für
alle unmittelbar am Bestand beteiligten Entitäten. `RentalAssignment` (oben, Entitäten-Abschnitt)
trägt keinen eigenen FK auf `Product` oder `ProductVariant`: Es referenziert `SerialUnit`
(nullable), `OnHandRecord` (nullable), die erfüllte `StockReservation` (Pflichtfeld, Amendment
OQ-0013) und `Party`. Alle drei Bestandsreferenzen sind durch die oben genannten Amendments
bereits variantengekeytet.

### Korrekte Aussage

`RentalAssignment` erfordert keine eigene Feldänderung. Die Variantenschlüsselung wirkt
transitiv: `RentalAssignment → SerialUnit → ProductVariant` und
`RentalAssignment → StockReservation → ProductVariant` liefern in jedem Fall dieselbe
`ProductVariant` (die Konsistenz dieser beiden Pfade wird durch die
`FULFILLED`/`kind = RENTAL`-Kopplung in Amendment OQ-0013 sichergestellt: eine
`RentalAssignment` verweist nur auf eine `StockReservation`, deren `serial_unit`-FK dieselbe
Einheit trägt wie der `RentalAssignment.serial_unit`-FK). Die Eigentumssemantik-Tabelle
(`owner_type`, oben) bleibt unverändert, da `owner_type` auf `OnHandRecord` lebt und dort
bereits durch ADR-0009 (Amendment 2026-06-28) auf `ProductVariant`-Ebene geführt wird.

### Migrationsbedeutung

Da `RentalAssignment` keine eigene FK-Änderung erfährt, ist für diese Entität keine gesonderte
Migration erforderlich; sie profitiert unverändert von der Standardvariante, die die
v2.0.0-Migration (REQ-0007) für `SerialUnit`, `OnHandRecord` und `StockReservation` anlegt.

ADR-0021 ist die autoritative Quelle für die Schlüsselung; das vorliegende Amendment
dokumentiert (das Ausbleiben einer) Auswirkung auf ADR-0013.
