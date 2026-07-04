# ADR-0018: Kanonisches Produktattribut-Vokabular

## Status
Accepted

## Context

ADR-0004 definiert ein dreistufiges EAV-System für Produktattribute: globale Taxonomiebäume,
konfigurierbare Attribut-Metadaten und typisierte Wertetabellen. Geschäftslogik-Regeln — FEFO-Rotation,
Lagerzonenvalidierung, Gefahrgut-Mitlagerungsregeln, Versandkosten, Zoll- und Compliance-Routing,
Seriennummern- und Chargenzwang (ADR-0011, ADR-0012) — müssen Produktattribute mit bekannten Schlüsseln
lesen. Ohne eine kanonische Vokabularschicht sind Geschäftsregeln an den jeweiligen Klassifizierungsstandard
gebunden: Eine Regel, die eCl@ss-Attribut `0173-1#02-AAO663#003` liest, bricht, wenn ein Betreiber
stattdessen ETIM oder GPC nutzt — oder wenn gar keine externe Klassifizierung konfiguriert ist.
Gleichzeitig unterliegen mehrere Klassifizierungsstandards kommerziellen Lizenzen: eCl@ss erfordert
eine Mitgliedschaft bei eCl@ss e.V. (Single- oder Concordance-Lizenz, optionaler Webservice mit
IRDI-basierter Abrechnung); ETIM ist ebenfalls lizenzpflichtig; UNSPSC und GPC sind frei nutzbar.
Das Open-Source-Backend muss als PyPI-Wheel auslieferbar bleiben, ohne lizenzpflichtigen Fremdinhalt
zu bündeln. Geschäftslogik, die direkt auf Klassifizierungsstandard-Attribute zugreift, würde dieses
Auslieferungsmodell verletzen oder N×M-Kopplungen zwischen Regeln und Standards erzeugen.

## Decision

KoalixCRM führt ein **kanonisches Produktattribut-Vokabular** ein. Alle Geschäftslogik liest
ausschließlich kanonische Schlüssel mit dem Namensraum `koalix.*`; kein Produktionscode
referenziert Klassifizierungsstandard-Attribut-Identifier direkt.

Das kanonische Vokabular ist:
- **klein** — Zielgröße 20–40 Schlüssel; ein Schlüssel graduiert ins Vokabular nur dann, wenn
  aktive Geschäftslogik ihn lesen muss,
- **stabil** — ausschließlich additive Erweiterungen; bestehende Schlüssel werden nicht umbenannt
  oder entfernt,
- **KoalixCRM-eigentum** — kein externer Standard besitzt oder kontrolliert das Vokabular,
  alle Schlüsseldefinitionen sind lizenzfrei,
- **nicht für freie Operator-Erweiterung vorgesehen** — Betreiber definieren eigene Attribute
  als Layer-2-EAV-Attribute (ADR-0004); der Aufstieg in das kanonische Vokabular erfolgt
  ausschließlich durch ADR-Amendment.

Klassifizierungsstandards (GPC, UNSPSC, eCl@ss, ETIM) sind **Adapter**: Sie liefern
Attributdefinitionen und Mappings in das EAV-System (ADR-0004), aber Geschäftslogik
referenziert ausschließlich kanonische Schlüssel.

## Why

Ein KoalixCRM-eigenes, stabiles Vokabular entkoppelt Geschäftsregeln vollständig von
Klassifizierungsstandards: Regeln werden einmalig geschrieben, Standards werden austauschbar.
Gleichzeitig bleibt das PyPI-Paket lizenzfrei, weil kein Produktionscode lizenzpflichtigen
Fremdinhalt importiert.

## Kanonische Schlüssel (Saatmenge)

Die folgende Tabelle definiert die initialen kanonischen Schlüssel. Erweiterungen erfordern
ein ADR-0018-Amendment.

| Schlüssel                    | Typ                              | Erlaubte Werte / Bereich                         | Geschäftslogik                                              | Standardverhalten bei fehlendem Wert                          |
|------------------------------|----------------------------------|--------------------------------------------------|-------------------------------------------------------------|---------------------------------------------------------------|
| `koalix.shelf_life_days`     | int (≥ 0)                        | 0–36 500 (100 Jahre)                             | FEFO-Rotation (ADR-0012); Ablaufwarnungen                   | Keine Ablaufwarnung; keine FEFO-Bevorzugung nach MHD-Abstand  |
| `koalix.storage_temp_min_c`  | decimal (°C)                     | −60,00–+50,00                                    | Lagerzonenvalidierung: Einheit darf nur in passende Kühlzone eingelagert werden | Keine Zoneneinschränkung                          |
| `koalix.storage_temp_max_c`  | decimal (°C)                     | −60,00–+50,00; muss ≥ `storage_temp_min_c` sein | Lagerzonenvalidierung                                       | Keine Zoneneinschränkung                                      |
| `koalix.hazard_class`        | enum (string)                    | UN/ADR-Klassen: `1`, `1.1`–`1.6`, `2.1`, `2.2`, `2.3`, `3`, `4.1`, `4.2`, `4.3`, `5.1`, `5.2`, `6.1`, `6.2`, `7`, `8`, `9` | Mitlagerungsregeln (ADR-0011); Versanddokumente | Keine Gefahrgutbehandlung; unbeschränkte Mitlagerung          |
| `koalix.weight_kg`           | decimal (kg, ≥ 0)                | 0,000–999 999,999                                | Versandkostenberechnung; Fahrzeugbeladungsgrenze            | Versandkostenkalkulation ohne Gewichtskomponente              |
| `koalix.country_of_origin`   | string (ISO 3166-1 Alpha-2)      | Zweistelliger ISO-Ländercode                     | Zollabwicklung; Tarifklassifizierung                        | Kein Ursprungsland in Versand- und Zolldokumenten             |
| `koalix.is_serialized`       | bool                             | `true` / `false`                                 | Erzwingt `SerialUnit`-Anlage bei Wareneingang (ADR-0012)    | Keine Pflicht zur Seriennummernanlage                         |
| `koalix.is_lot_tracked`      | bool                             | `true` / `false`                                 | Erzwingt `Batch`-Anlage bei Wareneingang (ADR-0012)         | Keine Pflicht zur Chargenanlage                               |
| `koalix.regulated_substance` | bool                             | `true` / `false`                                 | Compliance-Routing; löst regulatorische Prüfschritte im Wareneingang aus | Kein Compliance-Routing                               |

Hinweis: `koalix.hazard_class` folgt der UN/ADR-Klassennomenklatur (Übereinkommen über die internationale
Beförderung gefährlicher Güter auf der Straße). Für Seefracht (IMDG) und Luftfracht (IATA) gelten
dieselben Gefahrklassen; das Vokabular trägt nur den UN/ADR-Klassenwert, da die Übergangsregeln
zwischen den Trägerarten außerhalb des KoalixCRM-Geschäftslogikbereichs liegen.

## Mapping-Mechanismus

Ein `ProductAttributeMapping`-Datensatz bildet einen Layer-3-Quell-Identifier auf einen kanonischen
Schlüssel ab. Das Mapping ist **datengetrieben** — keine Codierung pro Standard erforderlich.

Jeder Mapping-Eintrag trägt:
- `source_standard` — Bezeichner des Klassifizierungsstandards (z. B. `eclass`, `gpc`, `etim`, `unspsc`)
- `source_attribute_id` — nativer Attribut-Identifier des Standards (z. B. eine IRDI für eCl@ss)
- `canonical_key` — Ziel-Schlüssel aus dem kanonischen Vokabular
- optional `transform` — Einheiten-Konversion oder Wert-Mapping als datengetriebene Formel
  (z. B. Konversion von Kelvin nach Celsius für `storage_temp_min_c`)

Geschäftslogik fragt ausschließlich den kanonischen Schlüssel ab; der Mapping-Mechanismus
schreibt den aufgelösten Wert in die kanonische Wertetabelle. Kein Produktionscode liest
`source_attribute_id`.

## Quellpriorität bei mehreren konfigurierten Layer-3-Adaptern

Wenn für denselben kanonischen Schlüssel Werte aus mehreren Quellen vorhanden sind, gilt
folgende Prioritätsreihenfolge (höchste Priorität zuerst):

1. **Operator-gesetzter expliziter Wert** — ein Betreiber hat den kanonischen Schlüssel direkt
   auf dem Produkt gesetzt (über das EAV-System, Layer 2 nach ADR-0004).
2. **Zuletzt importierter Quellen-Mapping-Wert** — bei mehreren Layer-3-Quellen gewinnt der
   Wert aus dem Import mit dem jüngsten `imported_at`-Zeitstempel.
3. **Systemstandard** — das in der Schlüsseldefinition festgelegte Standardverhalten bei
   fehlendem Wert (Spalte „Standardverhalten" in der Schlüsseltabelle).

Diese Reihenfolge ist eine Systemregeln, keine Konfigurationsoptionen; sie gilt
plattformweit einheitlich.

## Nicht-Ziele

- KoalixCRM ist kein generischer Klassifikations-Übersetzer. Das kanonische Vokabular
  bildet nur ab, was aktive Geschäftslogik liest.
- Das kanonische Vokabular ist nicht für freie Operator-Erweiterung vorgesehen. Betreiber
  erweitern über Layer-2-EAV-Attribute (ADR-0004). Aufstieg in das kanonische Vokabular
  erfolgt ausschließlich durch ADR-0018-Amendment.
- KoalixCRM bündelt keinen lizenzpflichtigen Layer-3-Inhalt (eCl@ss, ETIM). Mappings für
  lizenzfreie Standards (GPC, UNSPSC) dürfen als optionale Bundles ausgeliefert werden.
  Mappings für lizenzpflichtige Standards werden vom Betreiber zusammen mit dem lizenzierten
  Inhalt importiert — unter der eigenen Mitgliedschaft / Concordance- / Webservice-Lizenz
  des Betreibers.

## Alternatives Considered

- **eCl@ss als einzigen Standard etablieren** — abgelehnt: Lizenzkosten treffen jeden
  Betreiber, erzeugt Herstellerabhängigkeit, hilft Betreibern mit ETIM- oder GPC-Präferenz
  nicht.
- **GPC als einzigen Standard etablieren** — abgelehnt: bessere Lizenzbedingungen, aber
  koppelt Geschäftslogik weiterhin an einen einzelnen Standard und unterstellt Betreiber
  der GS1-Governance.
- **Geschäftslogik liest, welches Attribut zufällig gesetzt ist** — abgelehnt:
  N×M-Kopplung zwischen Regeln und Standards; jeder neue Standard bricht jede Regel neu.
- **Keine attributbasierte Geschäftslogik** — abgelehnt: Lagerrotation (FEFO), Gefahrgut-
  Mitlagerung, Lagerzonenvalidierung und Compliance-Routing erfordern typisierte
  Produktsemantik.

## Consequences

### Positive
- Geschäftslogik für FEFO-Rotation, Lagerzonenvalidierung, Gefahrgut-Mitlagerung,
  Versandkostenberechnung und Compliance-Routing wird einmalig gegen kanonische Schlüssel
  geschrieben und funktioniert mit jedem konfigurierten Layer-3-Adapter.
- Klassifizierungsstandards werden zu austauschbaren Adaptern; ein Betreiber wechselt von
  UNSPSC zu GPC, ohne Geschäftslogik zu ändern.
- Die Lizenzfläche des Open-Source-Backends beschränkt sich auf das Datenbankschema und
  die Mapping-Infrastruktur; der lizenzpflichtige Inhalt bleibt beim Betreiber.
- Betreiber ohne jede externe Klassifizierung erhalten volle Geschäftslogik, sobald sie
  kanonische Schlüssel direkt setzen.

### Negative
- KoalixCRM trägt die laufende Verantwortung für das kanonische Vokabular. Jede Addition
  erfordert ein ADR-0018-Amendment; die Governance darf nicht schleifen gelassen werden,
  sonst driftet das Vokabular unkontrolliert.
- Bestehende EAV-Attribute (ADR-0004), auf die aktive Geschäftslogik zugreift, müssen
  entweder in kanonische Schlüssel hochgestuft oder die Geschäftslogik muss auf kanonische
  Schlüssel umgestellt werden. Diese Migrationsarbeit ist Folgearbeit und liegt nicht im
  Umfang dieses ADR.

## Referenzen

- **ADR-0003 (Produkt-Katalog-Backbone):** „`ProductType` wird zu `Product` umbenannt und
  übernimmt die Rolle des kanonischen Katalogobjekts." — `Product` ist der Ankerpunkt, an
  dem kanonische Schlüssel gesetzt werden.
- **ADR-0004 (Klassifizierung und erweiterbare Attribute):** „Klassifizierungsschemata,
  Attributdefinitionen und Attributwerte werden in einem dreistufigen Modell organisiert."
  — Layer-2-EAV ist die Trägerstruktur für kanonische Schlüsselwerte und Layer-3-Quellen.
- **ADR-0011 (Lager- und Lebenszyklus-Ereignis-Log):** „Jede Lagerbewegung und jeder
  Lebenszyklus-Touch einer Einheit wird als unveränderlicher `StockMovement`-Datensatz
  gespeichert." — `koalix.hazard_class` steuert Mitlagerungsregeln, die `StockMovement`-
  Buchungen validieren oder blockieren.
- **ADR-0012 (Lebenszeit, Charge, Los und Seriennummernverfolgung):** „Für
  `Product.tracking_mode = BATCH` wird eine `Batch`-Entität eingeführt … Für
  `Product.tracking_mode = SERIAL` wird eine `SerialUnit`-Entität eingeführt." —
  `koalix.is_serialized` und `koalix.is_lot_tracked` steuern, welches Tracking-Regime beim
  Wareneingang erzwungen wird.

## Changelog
- 2026-05-05: Erstentscheidung.
