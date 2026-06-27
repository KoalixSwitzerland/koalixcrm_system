# QAS-0001: EAV-Attributfilter auf getypten Wertetabellen bei 10 000 SKUs (B-Tree-Pfad)

**Typ:** Performance  
**Lizenzseite:** Open-Source-Backend (`/app/koalixcrm`)  
**Bezug:** REQ-0010 (getypte Wertetabellen), OQ-0003 (Index-Strategie), ADR-0004

---

## Stimulus
Ein API-Consumer sendet eine GET-Anfrage an den Produkt-Listenendpunkt mit einem Attributfilter
auf einer `ProductAttributeDecimal`-Tabelle (z. B. alle Produkte mit `voc_content > 50 g/l`)
in einem Workspace mit 10 000 `Product`-Einträgen, von denen jedes durchschnittlich 50
Dezimal-Attributwerte in `ProductAttributeDecimal` trägt (gesamt: 500 000 Zeilen).

## Umgebung
- PostgreSQL-Datenbank auf dediziertem DB-Server (≥ 4 vCPU, ≥ 8 GB RAM, SSD-Storage)
- Zusammengesetzter B-Tree-Index auf `(product_id, attribute_definition_id)` ist vorhanden
- Kein zusätzlicher partieller Index
- Keine Abfrage-Caching-Schicht (kein Redis-Query-Cache)

## Antwort
Das System führt die Datenbankabfrage über den B-Tree-Index durch und liefert eine paginierte
Ergebnisliste (50 Einträge) als HTTP 200.

## Messbares Ergebnis
- p95-Antwortzeit der Datenbankabfrage (exkl. Netzwerk-Overhead): **< 300 ms**
- p99-Antwortzeit der Datenbankabfrage: **< 600 ms**
- Der PostgreSQL Query-Plan zeigt `Index Scan` oder `Bitmap Index Scan` auf dem
  `(product_id, attribute_definition_id)`-Index; kein `Seq Scan` auf der vollen Tabelle.

## Offene Frage
OQ-0003 fragt, ob darüber hinaus **partielle Indizes** (z. B. `WHERE value IS NOT NULL`)
die Indexgröße reduzieren und damit das p95-Ziel unter Last unterschreiten. Dieses QAS
beschreibt den Baseline-Fall ohne partiellen Index. Ein zweites Szenario (QAS mit partiellen
Indizes) ist Gegenstand der Architekten-Review nach Beantwortung von OQ-0003.

## Annahmen
- Filter-Abfragen verwenden parametrisierte SQL-Abfragen; keine dynamisch konkatetierten
  Strings.
- Datenbankstatistiken sind aktuell (ANALYZE wurde nach dem Laden der 500 000 Zeilen ausgeführt).
