# Offene Fragen (Open Questions)

Dieses Dokument sammelt blockierende oder klärungsbedürftige Fragen, die in publizierten
Artefakten nicht als offene Punkte eingebettet sein dürfen. Jeder Eintrag verweist auf das
betreffende Artefakt und den zuständigen Agenten oder Entscheider.

---

## OQ-0001 — eCl@ss-Lizenz: Inhalt nicht im Open-Source-Paket

**Referenz:** ADR-0004 (Klassifizierung und erweiterbare Attribute), Abschnitt „Lizenzbeschränkung"

**Frage:** eCl@ss-Code-Listen unterliegen einer kommerziellen Mitgliedslizenz (eCl@ss e.V.).
Dürfen eCl@ss-Klassifizierungsknoten und -Attributdefinitionen für KoalixCRM-Installationen, die
nicht von Quantalq betrieben werden, überhaupt importiert werden? Und wenn ja: Welche
Vertriebsform ist lizenzrechtlich zulässig (separate Download-Anleitung, Operator-Import-Skript,
kein Mitliefern im PyPI-Paket)?

**Auswirkung:** Die technische Unterstützung von eCl@ss im Datenmodell ist unabhängig von dieser
Frage korrekt. Die Frage betrifft ausschließlich, welche Onboarding-Dokumentation und welche
Import-Artefakte für eCl@ss-Inhalte bereitgestellt werden dürfen.

**Zuständig:** `kxcrm-requirements-engineer` klärt lizenzrechtliche Anforderung;
@scaphilo entscheidet über Vertriebsstrategie.

**Status:** Geschlossen (2026-05-05) — Gelöst durch ADR-0018 und ADR-0004 Amendment
2026-05-05. Geschäftslogik hängt am KoalixCRM-eigenen kanonischen Attribut-Vokabular, nicht an
einem Klassifizierungsstandard. Klassifizierungsstandards (eCl@ss, ETIM, GPC, UNSPSC) sind
Adapter, die Attributdefinitionen und Mappings in das kanonische Vokabular einspeisen.
Das Open-Source-Backend bündelt keinen lizenzpflichtigen Fremdinhalt. Betreiber, die eCl@ss
nutzen möchten, importieren den Inhalt unter ihrer eigenen Mitgliedschaft / Concordance- /
Webservice-Lizenz über betreiberseitige Werkzeuge — das ist eine Dokumentations- und
Onboarding-Angelegenheit, keine Architekturentscheidung. Die ursprüngliche Frage nach der
Vertriebsform für lizenzpflichtige Inhalte ist gegenstandslos, da eine Bündelung solcher
Inhalte von Anfang an nicht vorgesehen war.

---

## OQ-0002 — Migrationsstrategie `ProductType` → `Product`

**Referenz:** ADR-0003 (Produkt-Katalog-Backbone), Abschnitt „Negative Consequences"

**Frage:** Die Umbenennung `ProductType` → `Product` bricht bestehende FK-Referenzen aus
`contracts` und `accounting` (`crm_producttype`-Tabelle). Welche Strategie gilt: ein
phasenweiser Ansatz analog zur Party-Migration (ADR-0001 §6) mit Shadow-FKs und Datenmigration,
oder ein Big-Bang-Schnitt im Rahmen des v2.0.0-Majorrelease?

**Auswirkung:** Betrifft den Umfang von Datenmigrations-PRs und die Rollback-Strategie.

**Zuständig:** `kxcrm-requirements-engineer` formuliert Migrationsanforderung;
@scaphilo / @Hacont entscheiden Vorgehen.

**Status:** Geschlossen (2026-06-27) — OQ-0002 framte die Umbenennung als FK-Bruch; tatsächlich
handelt es sich um eine RENAME + APP-RELOCATION ohne per-Zeile-FK-Rewrite. Die gewählte Strategie
ist ein v2.0.0-Schnitt via Django `SeparateDatabaseAndState`: eine einzige Migration benennt
`crm.ProductType` → `products.Product` um; die DB-Operation ist
`ALTER TABLE crm_producttype RENAME TO products_product`; FK-Constraints aus `contracts` und
`accounting` folgen automatisch; das leere `Product`-Hüllenmodell entfällt im selben Schritt;
die Migration ist reversibel; ein Dry-Run-Management-Command prüft Zeilenzahlen und FK-Integrität
vorab. Ein langlebiger phasenweiser Dual-Stack entfällt. Die Änderung wird als BREAKING
v2.0.0-Schnitt ausgeliefert. Siehe ADR-0003 Amendment 2026-06-27.

---

## OQ-0003 — Index-Strategie für getypte EAV-Wertetabellen

**Referenz:** ADR-0004 (Klassifizierung und erweiterbare Attribute), Abschnitt „Getypte Wertetabellen"

**Frage:** ADR-0004 schreibt vor, dass jede Wertetabelle (`ProductAttributeDecimal` usw.) einen
zusammengesetzten Index auf `(product_id, attribute_definition_id)` benötigt. Werden darüber
hinaus partielle Indizes (z. B. nur nicht-null-Werte) oder GIN-Indizes auf dem JSONB-Spiegel
benötigt, um die anvisierten Listenseiten-Ladezeiten bei 10 000+ SKUs sicherzustellen?

**Auswirkung:** Betrifft Performance-Anforderungen, die als Quality Attribute Scenarios
(owned by `kxcrm-requirements-engineer`) formuliert werden sollten.

**Zuständig:** `kxcrm-requirements-engineer` formuliert messbare Performance-Szenarien.

**Status:** Offen (2026-05-03)

---

## OQ-0004 — Partitionierungsstrategie für `StockMovement`-Log

**Referenz:** ADR-0011 (Lagerbewegungen und Ereignis-Log), Abschnitt „Negative Consequences"

**Frage:** Der `StockMovement`-Log wächst monoton. Bei 10 Millionen+ Lagereinheiten und mehreren
Bewegungen pro Einheit und Monat entstehen innerhalb eines Jahres hunderte Millionen Zeilen.
Welche Partitionierungsstrategie gilt: monatliche Range-Partitionierung nach `occurred_at`
(PostgreSQL declarative partitioning), Archivierung älterer Partitionen in Cold Storage (S3 +
Parquet), oder beides? Ab welchem Datenbankvolumen greift die Partitionierung?

**Auswirkung:** Betrifft Datenbankbetrieb, Backup-Strategie und die Archivierungsanforderungen
des KoalixCRM-Betriebs. Kann als Quality Attribute Scenario (owned by
`kxcrm-requirements-engineer`) formuliert werden.

**Zuständig:** `kxcrm-requirements-engineer` formuliert messbare Kapazitäts-Szenarien;
@scaphilo entscheidet Betriebsstrategie.

**Status:** Geschlossen (2026-05-03) — PostgreSQL declarative Range-Partitionierung nach
`occurred_at`, monatlich, vor erstem Datenlauf eingerichtet. Cold-Storage-Archivierung ist
eine separate operative Policy; Auslöser ≈ 36 aktive Partitionen oder anhaltender
Datenbankgrößendruck. Siehe ADR-0011.

---

## OQ-0005 — Synchrone vs. asynchrone `StockBalance`-Aktualisierung

**Referenz:** ADR-0011 (Lagerbewegungen und Ereignis-Log), Abschnitt „Negative Consequences"

**Frage:** Soll `StockBalance` (ADR-0010) synchron im selben Datenbank-Transaktion wie
`StockMovement` aktualisiert werden (höherer Schreib-Overhead, aber sofortige Konsistenz),
oder asynchron via Celery-Task (geringerer Transaktions-Overhead, aber temporäre
Inkonsistenzfenster)? Wie lang ist ein akzeptables Inkonsistenzfenster für
ATP-Abfragen im produktiven Betrieb?

**Auswirkung:** Betrifft Performance-Anforderungen und Konsistenzgarantien; muss als Quality
Attribute Scenario formuliert werden.

**Zuständig:** `kxcrm-requirements-engineer` formuliert messbare Konsistenz- und
Performance-Szenarien; @scaphilo / @Hacont entscheiden Architekturvariante.

**Status:** Geschlossen (2026-05-03) — Synchrone `StockBalance`-Aktualisierung im selben
Datenbank-Transaktion wie `StockMovement` festgelegt; asynchrone Variante via Celery
abgelehnt, da ATP-Korrektheit finanziell tragend ist und Überverkäufe nicht tolerierbar sind.
Siehe ADR-0011.

---

## OQ-0006 — Archivierung dekommissionierter `SerialUnit`-Einträge

**Referenz:** ADR-0012 (Lebenszeit, Charge, Los und Seriennummernverfolgung), Abschnitt
„Negative Consequences"

**Frage:** Dekommissionierte `SerialUnit`-Einheiten (`decommissioned_at` gesetzt) wachsen
mit der Zeit auf Zigtausende Zeilen. Sollen diese Einheiten in eine Archivtabelle verschoben
werden (mit verlangsamter Traceability-Abfrage über zwei Tabellen), soft-deleted bleiben
(volle Traceability, wachsende Tabelle), oder gibt es eine Aufbewahrungsfrist nach der die
Einträge gelöscht werden dürfen (regulatorische Anforderung)?

**Auswirkung:** Betrifft Datenbankgröße, Traceability-Anforderungen und ggf. regulatorische
Aufbewahrungspflichten (Pharma, Lebensmittel).

**Zuständig:** `kxcrm-requirements-engineer` klärt regulatorische Aufbewahrungspflichten;
@scaphilo entscheidet Archivierungsstrategie.

**Status:** Geschlossen (2026-05-03) — Dekommissionierte `SerialUnit`-Zeilen werden dauerhaft
per Soft-Delete gehalten; keine Archivtabelle; partieller Index auf `decommissioned_at IS NULL`
für Hot-Path-Performance; Traceability über Jahrzehnte in einer Tabelle. Workspace-weite
Aufbewahrungsuntergrenze (`retention_floor`) als additive Schutzsperre konfigurierbar. Siehe
ADR-0012.

---

## OQ-0007 — Mehrstufige BOM-Explosion bei `EXPLODE_ON_PICK`-Kits

**Referenz:** ADR-0014 (Montage/Kitting und geteilter Bestand), Abschnitt „Negative Consequences"

**Frage:** Bei mehrstufigen Stücklisten (Baugruppe enthält Unterbaugruppen, die ihrerseits
Komponenten enthalten) muss die `EXPLODE_ON_PICK`-Logik die gesamte Baumstruktur rekursiv
auflösen. Wie viele BOM-Ebenen sind in der Praxis für KoalixCRM-Mandanten typisch? Gibt es
eine Tiefenbeschränkung, ab der eine manuelle Vorkommissionierung (`PREASSEMBLE`) erzwungen
werden soll? Ist eine rechenintensive Explosion zum Kommissionierzeitpunkt (synchron) oder
eine vorab berechnete Explosionsliste (Celery-Task bei BOM-Änderung) die richtige Strategie?

**Auswirkung:** Betrifft Performance-Anforderungen für die Auftragserfüllung und die
Komplexität der Fertigungssteuerungslogik.

**Zuständig:** `kxcrm-requirements-engineer` formuliert messbare Performance-Szenarien
für Kit-Explosion; @scaphilo entscheidet Betriebsstrategie.

**Status:** Geschlossen (2026-05-03) — Celery-Task berechnet abgeflachte BOM-Explosion in
`BillOfMaterialsExplosion`-Snapshot vor; Pick-Zeitpfad liest Snapshot; synchrone Neuberechnung
nur bei BOM-Versionsabweichung (selten). Weiche Tiefengrenze: 10 Ebenen (Warnung +
`PREASSEMBLE`-Empfehlung); harte Tiefengrenze: 20 Ebenen (Ablehnung, erzwingt `PREASSEMBLE`).
Siehe ADR-0014.

---

## OQ-0008 — Standard-kind-Wert für migrierte ProductType-Einträge

- **Raised:** 2026-05-03
- **Context:** REQ-0007 (Migration ProductType → Product), ADR-0003 (Backbone-Entscheidung)
- **Frage:** ADR-0003 definiert fünf `kind`-Werte (`SERVICE`, `TRADING_GOOD`, `MANUFACTURED_GOOD`,
  `KIT`, `RAW_MATERIAL`). Bestehende `ProductType`-Einträge tragen keinen `kind`-Wert. Welcher
  `kind`-Standardwert wird bei der Datenmigration für alle bisherigen `ProductType`-Zeilen
  verwendet, für die kein Branchenkontext bekannt ist? Soll der Wert null bleiben (optionales
  `kind`-Feld bis zur Nachpflege) oder muss ein konservativer Standardwert (`TRADING_GOOD`)
  gesetzt werden?
- **Blocks:** REQ-0007 AC-4 (kind-Standardwert in der Migration)

**Status:** Geschlossen (2026-06-27) — `TRADING_GOOD` ist der Standardwert für `kind` bei allen
migrierten `ProductType`-Zeilen. Das Feld ist Pflichtfeld (ADR-0019 / REQ-0001 AC-1); `null` ist
keine Option. `TRADING_GOOD` ist der am wenigsten einschränkende Wert: er erfordert kein
`ServiceProfile`, keine `BillOfMaterials`-Stückliste und impliziert keinen BOM; Lagerbestand ist
erlaubt. Das ADR-0019-Sperr-Set für frisch migrierte Zeilen ist leer; `kind` bleibt nach der
Migration frei korrigierbar — Betreiber reklassifizieren Einträge ohne Sperrwirkung. Der Wert wird
in der Migrations-Dokumentation festgehalten (REQ-0007 AC-4). Siehe ADR-0003 Amendment 2026-06-27.

---

## OQ-0009 — Unveränderlichkeit des kind-Felds nach Anlage abhängiger Objekte

- **Raised:** 2026-05-03
- **Context:** REQ-0001 (Product-Backbone), REQ-0015 (BillOfMaterials), REQ-0016 (ServiceProfile)
- **Frage:** REQ-0001 beschreibt, dass der `kind`-Wert unveränderlich ist, sobald abhängige
  Objekte (z. B. `BillOfMaterials` für `MANUFACTURED_GOOD`, `ServiceProfile` für `SERVICE`)
  existieren. Gilt diese Invariante auch für den Fall, dass `ProductVariant`-Einträge existieren?
  Und gilt sie bereits ab der Anlage eines einzigen Attributwerts (da `AttributeSet` kind-abhängig
  ist)? Die vollständige Liste der Zustände, die einen `kind`-Wechsel sperren, ist in der
  Applikationsschicht zu definieren.
- **Blocks:** REQ-0001 AC-4 (Unveränderlichkeit), REQ-0015 AC-1 (kind-Invariante BOM), REQ-0016 AC-2 (kind-Invariante ServiceProfile)

**Status:** Geschlossen (2026-06-27) — Das vollständige Sperr-Set ist in ADR-0019
(Produkt-`kind`-Invarianten und Gating abhängiger Objekte) definiert. `kind` ist frei änderbar,
solange keines der folgenden Elemente existiert: `ServiceProfile`, `BillOfMaterials`,
`ProductionOrder`, eine Lagertatsache (`StockMovement`, `SerialUnit`, `Batch`, `OnHandRecord`),
ein gesetzter `kit_mode`-Wert oder ein Attributwert in einem kind-gebundenen `AttributeSet`.
Das bloße Vorhandensein von `ProductVariant`-Einträgen sperrt `kind` nicht — `ProductVariant`
ist kind-agnostisch. Ein Attributwert in einem kind-gebundenen `AttributeSet` (ADR-0004, `kind`
als Achse) gehört zum Sperr-Set. Durchsetzung erfolgt durch die `ProductKindPolicy`-Komponente
in der Applikationsschicht; kein Datenbankconstraint. REQ-0015 AC-1 erfordert eine
Folgekorrektur durch den `kxcrm-requirements-engineer` (BOM-Gating auf
`kind ∈ {MANUFACTURED_GOOD, KIT}` ausweiten). Siehe ADR-0019.

---

## OQ-0010 — Soft-Reservierungs-Businessstep: Planungs- vs. physisches `rental_out`

- **Raised:** 2026-05-04
- **Context:** UC-0007 (Mietangebot erstellen), ADR-0011 (Lager- und Lebenszyklus-Ereignis-Log)
- **Frage:** ADR-0011 enthält `rental_out` im `business_step`-Wertebereich, definiert aber nicht den Unterschied zwischen einem geplanten Mietbeginn (Angebot gespeichert, Gerät noch im Lager) und einem physisch vollzogenen Mietbeginn (Übergabe an Kunden stattgefunden). GS1 EPCIS 2.0 CBV kennt das `disposition`-Attribut (z. B. `in_progress`) für diesen Zweck. Soll das `StockMovement`-Modell ein `disposition`-Feld erhalten, oder wird ein separater Businessstep-Wert (z. B. `planned_rental_out`) für Soft-Reservierungen eingeführt?
- **Blocks:** UC-0007 BAC-6 (EPCIS-CBV-Konsistenz), UC-0007 BAC-3 (Soft-Reservierung beim Angebotsspeichern)

**Status:** Geschlossen (2026-05-04) — `StockMovement` erhält ein `disposition`-Feld (EPCIS CBV Enum, nullable; Werte: `in_progress`, `reserved`, `in_transit`, `in_possession`, `returned`, `destroyed`); geplante Mietausgabe trägt `disposition = reserved`, physische Übergabe `disposition = in_possession`; kein separater `planned_rental_out`-Businessstep. Siehe ADR-0011, Amendment 2026-05-04.

---

## OQ-0011 — Zeitfensterbasierte Verfügbarkeitsfunktion für Miet-ATP

- **Raised:** 2026-05-04
- **Context:** UC-0007 (Mietangebot erstellen), ADR-0010 (Lagerbestandszustände und Reservierungen)
- **Frage:** Die skalare ATP-Formel aus ADR-0010 (`ATP = qty_on_hand − qty_booked − qty_reserved_for_document + qty_ordered`) ist für Mietanwendungen nicht ausreichend, da Miet-Verfügbarkeit eine Zeitfensterfrage ist: `is_free(serial_unit, [start, end])`. Welcher API-Endpunkt und welche Datenbankabfragestrategie implementieren die zeitfensterbasierte Verfügbarkeitsprüfung pro `SerialUnit`? Wird ein dedizierter Endpunkt (`GET /api/products/{id}/serial-units/availability/`) oder eine Erweiterung von `StockBalance` um zeitgebundene Reservierungsauswertung spezifiziert?
- **Blocks:** UC-0007 BAC-1 (Verfügbarkeitskalender), UC-0007 BAC-2 (Überschneidungskonflikt)

**Status:** Geschlossen (2026-05-04) — `is_free(serial_unit, start, end) -> bool` und `free_windows(product, start, end) -> list[(serial_unit, [(from, to)])]` als berechnete Abfrage über `StockReservation` (ohne materialisierte `UnitAvailabilityWindow`-Tabelle) definiert; dedizierter Endpunkt `GET /api/products/{id}/serial-units/availability/` festgelegt. Siehe ADR-0010, Amendment 2026-05-04.

---

## OQ-0012 — Kopplung Angebots-Lebenszyklus an `StockReservation`-Übergänge

- **Raised:** 2026-05-04
- **Context:** UC-0007 (Mietangebot erstellen), ADR-0010 (Lagerbestandszustände und Reservierungen)
- **Frage:** Kein bestehendes ADR definiert, welche Angebotszustände (DRAFT, SENT, ACCEPTED, REJECTED, EXPIRED) eine `StockReservation` erzeugen, aufrechterhalten oder auf `CANCELLED` setzen. Welche Zustandsübergänge des Angebots lösen welche `StockReservation`-Zustandsübergänge aus? Was passiert, wenn zwei konkurrierende Angebote für dieselbe `SerialUnit` und denselben Zeitraum gleichzeitig im Status SENT sind?
- **Blocks:** UC-0007 BAC-4 (Freigabe der Reservierung), UC-0007 Alternativablauf C

**Status:** Geschlossen (2026-05-04) — Zustandsmaschine Angebot → `StockReservation` mit neuem `reservation_status`-Feld (`PROVISIONAL`/`CONFIRMED`) definiert; DRAFT und SENT erzeugen `PROVISIONAL`-Reservierung, ACCEPTED überführt in `CONFIRMED`; REJECTED/EXPIRED/CANCELLED setzen `status = CANCELLED`; Konkurrenzregel: first-SENT-wins (FIFO nach Zeitstempel), spätere SENT-Versuche auf dieselbe SerialUnit/Zeitfenster-Kombination werden mit HTTP 409 abgewiesen. Siehe ADR-0010, Amendment 2026-05-04.

---

## OQ-0013 — Abgrenzung `StockReservation` vs. `RentalAssignment` als autoritative Belegungsquelle

- **Raised:** 2026-05-04
- **Context:** UC-0007 (Mietangebot erstellen), ADR-0010, ADR-0013
- **Frage:** ADR-0010 definiert `StockReservation` für Angebotsreservierungen; ADR-0013 definiert `RentalAssignment` für aktive Mietverträge. Beide können denselben `SerialUnit`-Zeitraum blockieren. Welche Entität ist die autoritative Quelle für den Verfügbarkeitskalender — und müssen beide Quellen beim Aufbau des Kalenders vereinigt werden? Wie verhindert die Applikationsschicht, dass `StockReservation` und `RentalAssignment` denselben Zeitraum doppelt blockieren?
- **Blocks:** UC-0007 BAC-1 (Verfügbarkeitskalender), UC-0007 Nachbedingungen

**Status:** Geschlossen (2026-05-04) — `StockReservation` (mit neuem `kind`-Feld: `SALE`, `RENTAL`, `PROJECT_HOLD`) ist alleinige Belegungsquelle für den Verfügbarkeitskalender; `RentalAssignment` bleibt als Spezialisierung (physische Übergabephase) erhalten und verweist per Pflicht-FK auf die erfüllte `StockReservation`; Doppelzählung ist strukturell ausgeschlossen, da `StockReservation` beim Anlegen des `RentalAssignment` auf `status = FULFILLED` gesetzt wird; Migrationspfad (retroaktive synthetische `StockReservation` für historische `RentalAssignment`-Einträge) festgelegt. Siehe ADR-0010 und ADR-0013, Amendment 2026-05-04.

---

## OQ-0014 — Halter-Zeitleiste als dedizierter Abfragepfad in ADR-0015

- **Raised:** 2026-05-04
- **Context:** UC-0007 (Mietangebot erstellen), ADR-0015 (Geräte-Lebenszyklus-Historie)
- **Frage:** ADR-0015 definiert den Abfragepfad „Vollständige Einheitenhistorie" (alle `StockMovement`-Zeilen per `serial_unit_id`, sortiert nach `occurred_at`). Eine spezifische Projektion „Halter-Zeitleiste" — wer hat die Einheit in welchem Zeitfenster gehalten, abgeleitet aus `rental_out`/`rental_return`-Paaren und `owner_party`-Feldern — ist kein eigenständiger Abfragepfad in ADR-0015. Soll ADR-0015 diesen Abfragepfad ergänzen? Wird er als dedizierter API-Endpunkt oder als materialisierte Sicht implementiert?
- **Blocks:** UC-0007 BAC-5 (Lebenszyklus-Log — Halter-Zeitlinie), UC-0007 BAC-1

**Status:** Geschlossen (2026-05-04) — Vierter Lebenszyklus-Abfragepfad „Halter-Zeitleiste" (`who_held_it_when(serial_unit) -> list[(holder_party, from, to)]`) als Projektion über `StockMovement.disposition ∈ {in_possession, returned}` (ADR-0011, Amendment OQ-0010) als berechnete Abfrage definiert; dedizierter Endpunkt `GET /api/serial-units/{id}/holder-timeline/` festgelegt; keine materialisierte Sicht. Siehe ADR-0015, Amendment 2026-05-04.

---

## OQ-0015 — Put-Away-Strategie beim Wareneingang: Service-Algorithmus oder konfigurierbares Attribut?

- **Raised:** 2026-05-04
- **Context:** UC-0010 (Wareneingang mit Lieferschein und Lagerplatzvorschlag)
- **Question:** UC-0010 beschreibt die gewünschte Lagerplatzvorschlag-Regel beim Wareneingang (1. bestehender Stellplatz mit gleichem Produkt + Charge bevorzugt; 2. freier Stellplatz nächster passender Größe). Kein bestehendes ADR legt fest, ob diese Regel als eigenständiger Put-Away-Algorithmus im Backend-Service implementiert wird, als konfigurierbare Strategie pro Produkt (Attribut auf `Product` oder `Location`), oder als mandantenweite Konfiguration. Die Entscheidung hat Auswirkungen auf die Schnittstelle des `POST /api/stock/movements/`-Endpunkts und auf die Testbarkeit der Vorschlagslogik.
- **Blocks:** UC-0010 BAC-4 (Lagerplatzvorschlag-Regel)
- **Zuständig:** `kxcrm-architect` entscheidet Implementierungsstrategie; `kxcrm-requirements-engineer` präzisiert Anforderung nach Entscheid.

**Status:** Offen (2026-05-04) — Das `GoodsReceipt`-Aggregat (ADR-0017) ist etabliert; das Aggregat nimmt `destination_location` je Position auf. Die einzige verbleibende Entscheidung ist der Algorithmus, der diesen Vorschlag erzeugt. OQ-0015 bleibt offen bis dieser Algorithmus als separates ADR entschieden wird.

---

## OQ-0016 — Identifier-Registry: eigenständige Entität oder Suchfunktion im Endpunkt?

- **Raised:** 2026-05-04
- **Context:** UC-0009 (Komponentenentnahme), UC-0010 (Wareneingang)
- **Question:** Beide Use Cases benötigen einen Scan-Auflösungsmechanismus, der einen gescannten Barcode-Wert gegen mehrere Identifier-Typen auflöst: `Product.gtin` (GS1 GTIN), `Location.external_ref` (ADR-0009), `SerialUnit.global_uid` (ADR-0012). Kein bestehendes ADR definiert diese Auflösungslogik als eigenständige Identifier-Registry-Entität oder als wiederverwendbare Funktion. Soll eine dedizierte `IdentifierRegistry`-Entität oder ein zentraler `POST /api/stock/scan/`-Endpunkt als kanonische Auflösungsschicht eingeführt werden? Welche Reihenfolge gilt bei Identifier-Kollisionen (z. B. ein Wert, der sowohl einer GTIN als auch einem `external_ref` entspricht)?
- **Blocks:** UC-0009 BAC-1, BAC-2, BAC-3 (Scan-Auflösungsvarianten); UC-0010 BAC-3 (Scan-Auflösung pro Position)
- **Zuständig:** `kxcrm-architect` entscheidet Architektur der Identifier-Registry; `kxcrm-requirements-engineer` präzisiert Anforderung nach Entscheid.

**Status:** Geschlossen (2026-05-04) — GS1 AI-gesteuerter kanonischer Endpunkt `POST /api/v1/scan/resolve` eingeführt; zweistufige Auflösungsreihenfolge (GS1 AI-Parsing zuerst, Freitext-Abgleich bei fehlendem AI-Präfix); keine eigenständige Registry-Entität. Kollisionsregel: GS1 schlägt Freitext; Freitext-Multi-Hit → HTTP 409 mit Kandidatenliste. Siehe ADR-0016.

---

## OQ-0017 — business_step „inventorying" fehlt im ADR-0011-Wertebereich

- **Raised:** 2026-05-04
- **Context:** UC-0009 (Komponentenentnahme mit Bestandsbestätigung)
- **Question:** UC-0009 verwendet `business_step = inventorying` für den Korrektur-`StockMovement` bei einer Ad-hoc-Zykluszählung. Der Wertebereich von ADR-0011 enthält `inventorying` nicht. Soll ADR-0011 um `inventorying` als gültigen `business_step`-Wert erweitert werden? Ist `inventorying` ein Standard-GS1-EPCIS-CBV-Businessstep oder ein projektspezifischer Wert?
- **Blocks:** UC-0009 BAC-5 (Buchung des Korrektur-Events)
- **Zuständig:** `kxcrm-architect` entscheidet Amendment zu ADR-0011; `kxcrm-requirements-engineer` aktualisiert UC-0009 BAC-5 nach Entscheid.

**Status:** Geschlossen (2026-05-04) — `inventorying` als projektspezifischer `business_step`-Wert in ADR-0011 ergänzt; Semantik: mengenloser Verifikationsdatensatz (qty = null), kein Saldo-Update; Diskrepanzfall erzeugt separaten `adjustment`-Event mit `reason_code = cycle_count_discrepancy`. Siehe ADR-0011, Amendment 2026-05-04 (OQ-0017).

---

## OQ-0018 — GoodsReceipt-Datenmodell fehlt in bestehenden ADRs

- **Raised:** 2026-05-04
- **Context:** UC-0010 (Wareneingang mit Lieferschein und Lagerplatzvorschlag)
- **Question:** UC-0010 setzt ein `GoodsReceipt`-Aggregat mit Status-Enum (`IN_PROGRESS`, `COMPLETED`) und `GoodsReceiptPosition`-Einträgen voraus. Kein bestehendes ADR (ADR-0009 bis ADR-0015) definiert dieses Aggregat. Soll ein neues ADR-0017 das `GoodsReceipt`-Datenmodell spezifizieren? Welche Felder sind erforderlich (Lieferant, Lieferschein-Nummer, Buchungsdatum, Positions-Statusmaschine)? Wie verhält sich `GoodsReceipt` bei Teillieferungen über mehrere Tage (mehrere Ingestions-Calls auf dasselbe Aggregat)?
- **Blocks:** UC-0010 BAC-2 (GoodsReceipt-Statusübergänge), UC-0010 BAC-7 (Verknüpfung über document_id)
- **Zuständig:** `kxcrm-architect` erstellt ADR-0017; `kxcrm-requirements-engineer` aktualisiert UC-0010 nach Entscheid.

**Status:** Geschlossen (2026-05-04) — `GoodsReceipt`-Aggregat mit Status-Enum (`DRAFT`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`) und `GoodsReceiptLine`-Kindentitäten mit `line_status ∈ {PENDING, CONFIRMED, MISMATCHED}` eingeführt; COMPLETED-Übergang erzeugt synchrone `StockMovement`-Buchungen je Position. Teillieferungen sind über mehrfache Ingestion-Calls auf dasselbe IN_PROGRESS-Aggregat möglich. Siehe ADR-0017.

---

## OQ-0019 — Komponenten-Varianten-Auflösung zwischen BOM (Product-Ebene) und Bestandsbuchung (Variant-Ebene)

- **Raised:** 2026-07-04
- **Context:** UC-0012 (Kit kommissionieren und Fertigprodukt montieren), ADR-0006, ADR-0011, ADR-0014, ADR-0021
- **Frage:** `BomItem` (ADR-0006, REQ-0015 AC-2) trägt einen FK auf das Komponenten-`Product`, nicht auf eine konkrete `ProductVariant`. `ProductionOrderComponent` (ADR-0014) trägt zusätzlich einen FK auf `ProductVariant`, der jedoch nullable ist. `OnHandRecord` ist seit ADR-0021 ausschließlich über `ProductVariant` autoritativ geschlüsselt, und `StockMovement.variant` (ADR-0011) bleibt laut Feldtabelle nullable — die Ripple-Liste der ADR-0021-Amendments nennt `ADR-0012` und `ADR-0010`, aber nicht `ADR-0011`. Trägt das Komponenten-`Product` eines `BomItem` mehr als eine `ProductVariant`, ist nicht definiert, welche konkrete Komponenten-`ProductVariant` beim Kit-Pick oder bei der `ProductionOrderComponent`-Reservierung für die Bestandsbuchung herangezogen wird. Soll `BomItem` einen optionalen oder verpflichtenden FK auf eine konkrete Komponenten-`ProductVariant` erhalten, oder wird `StockMovement.variant` verpflichtend gemacht und die Variantenwahl an anderer Stelle (z. B. Pick-Zeitpfad, Konfigurationsattribut auf `ProductionOrderComponent`) entschieden?
- **Blocks:** UC-0012 Hauptablauf (Teil A und Teil B), UC-0012 BAC-5

**Status:** Geschlossen (2026-07-04) — `BomItem` bleibt Product-gekeyt (unverändert gegenüber
ADR-0006/ADR-0021); ergänzt um ein optionales, nicht-bindendes
`default_component_variant`-Feld (ADR-0006, Nachtrag 2026-07-04). Die verbindliche
Komponenten-Variantenauflösung bindet erst am Buchungspunkt, dreistufig: (1) explizite Angabe im
Request, (2) `BomItem.default_component_variant`, (3) die einzige `ProductVariant` des
Komponenten-`Product`; ohne eindeutige Auflösung weist das Backend die Buchung mit HTTP 422 ab.
`ProductionOrderComponent.variant` (ADR-0014) und `StockMovement.variant` (ADR-0011) werden
dazu beide obligatorisch; damit ist die von ADR-0021 offengelassene Ripple-Lücke (`ADR-0011`
fehlte in der Ripple-Liste) geschlossen. Siehe ADR-0006, ADR-0011 und ADR-0014, je Nachtrag/
Amendment 2026-07-04, sowie ADR-0021 §Ripple-Liste Komponenten-Variantenauflösung und
As-Built-Anker.

---

## OQ-0020 — Eltern-Kind-Multiplizität von `AGGREGATION_EVENT`-Zeilen bei nicht-serialisiertem Fertigprodukt

- **Raised:** 2026-07-04
- **Context:** UC-0012 (Kit kommissionieren und Fertigprodukt montieren), ADR-0011, ADR-0014
- **Frage:** ADR-0014 beschreibt die As-Built-Erfassung als „ein oder mehrere `AGGREGATION_EVENT`-Einträge, die den Eltern-`SerialUnit`-FK (Fertigprodukt) mit den Kind-Komponenten-Lots/Serien verknüpfen". Das `StockMovement`-Schema (ADR-0011) trägt je Zeile genau einen `serial_unit`-FK und genau einen `batch`-FK; kein Feld verknüpft mehrere Kind-Zeilen eindeutig mit derselben Eltern-Einheit desselben Fertigungsabschlusses. Trägt die Fertigprodukt-`ProductVariant` `tracking_mode ∈ {NONE, BATCH}` statt `SERIAL`, existiert kein Eltern-`SerialUnit`, auf den die `AGGREGATION_EVENT`-Zeilen zeigen könnten. Soll `StockMovement` ein Gruppierungsfeld (z. B. `parent_event_id` oder `production_order_id` als gemeinsamer Schlüssel für alle Kind-Zeilen eines Fertigungsabschlusses) erhalten, und wie wird der Eltern-Anker für nicht-serialisierte Fertigprodukte (`tracking_mode ∈ {NONE, BATCH}`) modelliert?
- **Blocks:** UC-0012 Hauptablauf (Teil B, Fertigungsabschluss), UC-0012 BAC-3

**Status:** Geschlossen (2026-07-04) — `StockMovement` erhält ein tracking-mode-unabhängiges
Gruppierungsfeld `aggregation_group` (UUID; obligatorisch bei `event_type = AGGREGATION_EVENT`),
das alle Kind-Zeilen desselben Fertigungsabschlusses verknüpft, unabhängig davon, ob ein
Eltern-`SerialUnit` existiert. Ergänzend tragen die Zeilen `parent_serial_unit` (gesetzt bei
`tracking_mode = SERIAL`) bzw. `parent_batch` (gesetzt bei `tracking_mode = BATCH`) als
diskreten Eltern-Anker; bei `tracking_mode = NONE` ist `aggregation_group` der alleinige Anker.
Keine separate As-Built-Aggregatentität wird eingeführt (ADR-0014 lehnt diese Alternative
bereits ab); die Lösung bleibt vollständig innerhalb der `StockMovement`-Tabelle und ist über
`aggregation_group` als interne URN EPCIS-2.0-`parentID`-exportfähig, auch ohne physischen
Trägeridentifikator. Siehe ADR-0011, Amendment 2026-07-04, und ADR-0014, Nachtrag 2026-07-04.

---

## OQ-0021 — `GoodsReceiptLine.product` fehlt in der ADR-0021-Ripple-Liste (Product statt ProductVariant)

- **Raised:** 2026-07-04
- **Context:** REQ-0027 (Wareneingang als Prozess-Aggregat), ADR-0017 (GoodsReceipt als Prozess-Aggregat), ADR-0021 (Produkt-Variantengranularität), UC-0010
- **Frage:** ADR-0017 definiert `GoodsReceiptLine.product` als FK auf `Product`. Die ADR-0021-Ripple-Liste (Ripple-Liste Lager-/Serien-/Reservierungsdomäne und Ripple-Liste Komponenten-Variantenauflösung und As-Built-Anker) führt `OnHandRecord`, `Batch`, `SerialUnit`, `StockBalance`, `StockReservation`, `StockMovement` und `ProductionOrderComponent` als durchgängig auf `ProductVariant` umgestellt auf, nennt `GoodsReceiptLine`/ADR-0017 jedoch nicht. UC-0010 setzt in seinem Beispiel-Payload (`POST /api/stock/goods-receipts/`) bereits `product_variant_id` je Position voraus und widerspricht damit der aktuellen ADR-0017-Feldliste. Soll `GoodsReceiptLine.product` zu einem obligatorischen FK auf `ProductVariant` geändert werden (analog zu `StockMovement.variant`, ADR-0011 Amendment OQ-0019), und wie wird der Chargenbezug (`GoodsReceiptLine.batch`) mit der Variantenschlüsselung konsistent gehalten?
- **Blocks:** REQ-0027 AC-2 (variantengenaue Fassung von `GoodsReceiptLine`), UC-0010 BAC-1/BAC-3/BAC-6 (Konsistenz des Payload-Schemas mit dem tatsächlichen Datenmodell)
- **Zuständig:** `kxcrm-architect` entscheidet über ein ADR-0017-Amendment; `kxcrm-requirements-engineer` aktualisiert REQ-0027 nach Entscheid.

**Status:** Geschlossen (2026-07-04) — `GoodsReceiptLine.product` (FK → `Product`) wird zu
`GoodsReceiptLine.variant` (FK → `ProductVariant`, obligatorisch); der Bezug zum abstrakten
Katalogobjekt bleibt über den FK-Pfad `GoodsReceiptLine.variant → ProductVariant → Product`
erreichbar. Der Chargenbezug bleibt konsistent: die Applikationsschicht erzwingt, dass eine
gesetzte `GoodsReceiptLine.batch` zur selben `ProductVariant` gehört wie
`GoodsReceiptLine.variant` (kein Datenbank-Constraint). Der beim `COMPLETED`-Übergang erzeugte
`StockMovement`-Datensatz übernimmt `variant` direkt von `GoodsReceiptLine.variant`, ohne
eigene Variantenauflösung am Buchungspunkt. Siehe ADR-0017, Nachtrag 2026-07-04, sowie
ADR-0021 §Ripple-Liste Wareneingang und Identifier-Registry.

---

## OQ-0022 — Identifier-Registry: GTIN-Auflösungsziel `Product` (ADR-0016) vs. `ProductVariant` (ADR-0021)

- **Raised:** 2026-07-04
- **Context:** REQ-0026 (Identifier-Registry und Barcode-Auflösung), ADR-0016, ADR-0021, UC-0009
- **Frage:** ADR-0016 definiert die GS1-AI-Stufe-1-Auflösung für `(01)` GTIN gegen `Product.gtin`. Die Schlüsselungstabelle in ADR-0021 platziert `gtin` jedoch auf `ProductVariant`-Ebene ("GTIN ist die handelsseitige Einheiten-ID; verschiedene Verpackungsgrößen tragen je eigene GTINs"), und UC-0009 (Änderungsprotokoll 2026-07-04, BAC-1) setzt bereits eine Auflösung gegen `ProductVariant.gtin` voraus. ADR-0016 ist von keiner der beiden ADR-0021-Ripple-Listen erfasst. Soll ADR-0016 dahingehend geändert werden, dass AI `(01)` GTIN und die Freitext-Stufe-2-Regel 3 (`Product.sku`) gegen `ProductVariant.gtin`/`ProductVariant.sku` statt gegen `Product` auflösen?
- **Blocks:** REQ-0026 AC-2 (variantengenaue Fassung der GS1-AI-Auflösung), UC-0009 BAC-1 (Konsistenz mit dem tatsächlich zitierten ADR-0016-Wortlaut)
- **Zuständig:** `kxcrm-architect` entscheidet über ein ADR-0016-Amendment; `kxcrm-requirements-engineer` aktualisiert REQ-0026 nach Entscheid.

**Status:** Geschlossen (2026-07-04) — AI `(01)` GTIN löst gegen `ProductVariant.gtin` statt
`Product.gtin` auf; die Freitext-Stufe-2-Regel 3 löst gegen `ProductVariant.sku` statt
`Product.sku` auf; der Erfolgsantwort-`kind`-Wert für einen GTIN-/SKU-Treffer wechselt von
`"product"` auf `"product_variant"` (`id` referenziert die getroffene `ProductVariant`). Die
übrigen Auflösungsregeln (AI `(00)` SSCC → `HandlingUnit.sscc`, AI `(01)`+`(21)` SGTIN und AI
`(8003)` GIAI → `SerialUnit.global_uid`, AI `(414)` GLN → `Location.external_ref`, Freitext-
Regeln 1/2 → `Location.external_ref`/`SerialUnit.global_uid`) referenzieren kein `Product`-Feld
und bleiben unverändert; ein Sweep gegen die ADR-0021-Schlüsselungstabelle bestätigt, dass nur
`gtin` und `sku` betroffen sind (`mpn`, ebenfalls variantengekeyt, wird von ADR-0016 nicht
referenziert). Die katalogweite (nicht workspace-gebundene) Sonderstellung der GTIN-Auflösung
überträgt sich unverändert von `Product.gtin` auf `ProductVariant.gtin`. Siehe ADR-0016,
Nachtrag 2026-07-04, sowie ADR-0021 §Ripple-Liste Wareneingang und Identifier-Registry.
