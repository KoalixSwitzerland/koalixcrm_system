# QAS-0002: JSONB-Spiegel-Listenabfrage bei 10 000 SKUs und 200 Attributen pro Produkt (GIN-Pfad)

**Typ:** Performance  
**Lizenzseite:** Open-Source-Backend (`/app/koalixcrm`)  
**Bezug:** REQ-0010 (JSONB-Spiegel), OQ-0003 (Index-Strategie), ADR-0004

---

## Stimulus
Ein API-Consumer sendet eine GET-Anfrage an den Produkt-Listenendpunkt mit einem JSONB-Pfad-Filter
auf dem `attribute_mirror`-Feld (z. B. `attribute_mirror->>'ral_color' = '3020'`) in einem
Workspace mit 10 000 `Product`-Einträgen, von denen jedes durchschnittlich 200 Attributwerte
im `attribute_mirror`-JSONB-Block trägt.

## Umgebung
- PostgreSQL-Datenbank auf dediziertem DB-Server (≥ 4 vCPU, ≥ 8 GB RAM, SSD-Storage)
- GIN-Index auf dem `attribute_mirror`-JSONB-Feld ist vorhanden
- Der JSONB-Spiegel ist vollständig und aktuell (kein Lag durch asynchrone Aktualisierung)
- Keine Abfrage-Caching-Schicht

## Antwort
Das System führt die Datenbankabfrage über den GIN-Index auf `attribute_mirror` durch und liefert
eine paginierte Ergebnisliste (50 Einträge) als HTTP 200.

## Messbares Ergebnis
- p95-Antwortzeit der Datenbankabfrage (exkl. Netzwerk-Overhead): **< 500 ms**
- p99-Antwortzeit der Datenbankabfrage: **< 900 ms**
- Der PostgreSQL Query-Plan zeigt `Bitmap Index Scan` auf dem GIN-Index; kein `Seq Scan` auf
  der `Product`-Tabelle.

## Offene Frage
OQ-0003 fragt, ob der GIN-Index auf dem gesamten JSONB-Block oder selektiv auf einzelnen
JSONB-Pfaden (`jsonb_path_ops` vs. `jsonb_ops`) angelegt werden soll. Dieses QAS beschreibt
den `jsonb_ops`-Baseline-Fall. Die optimale GIN-Variante ist Gegenstand der Architekten-Review
nach Beantwortung von OQ-0003.

## Annahmen
- `attribute_mirror` enthält flache Schlüssel-Wert-Paare (keine verschachtelte JSON-Tiefe > 1).
- Die GIN-Indizierung umfasst alle Schlüssel im JSONB-Block.
- Datenbankstatistiken sind aktuell.
