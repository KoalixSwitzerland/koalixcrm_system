# QAS-0003: Kombinierter Attributfilter (typisierte Tabelle + JSONB-Spiegel) bei 10 000 SKUs unter gleichzeitiger Last

**Typ:** Performance / Skalierbarkeit  
**Lizenzseite:** Open-Source-Backend (`/app/koalixcrm`)  
**Bezug:** REQ-0010 (getypte Wertetabellen + JSONB-Spiegel), OQ-0003 (Index-Strategie), ADR-0004

---

## Stimulus
Zehn gleichzeitige API-Consumer senden je eine GET-Anfrage an den Produkt-Listenendpunkt. Jede
Anfrage kombiniert einen Filter auf `ProductAttributeDecimal` (B-Tree-Pfad) mit einem Filter auf
`attribute_mirror` (GIN-Pfad) in einem Workspace mit 10 000 `Product`-Einträgen und
200 Attributen pro Produkt.

## Umgebung
- PostgreSQL-Datenbank auf dediziertem DB-Server (≥ 4 vCPU, ≥ 8 GB RAM, SSD-Storage)
- Zusammengesetzter B-Tree-Index auf `(product_id, attribute_definition_id)` ist vorhanden
- GIN-Index auf `attribute_mirror` ist vorhanden
- Django-Backend mit 4 Worker-Prozessen (Gunicorn oder Uvicorn)
- Keine Abfrage-Caching-Schicht

## Antwort
Das System führt alle zehn Parallelabfragen über die vorhandenen Indizes durch und liefert für
jede Anfrage eine paginierte Ergebnisliste (50 Einträge) als HTTP 200, ohne Query-Queue-Stau.

## Messbares Ergebnis
- p95-Antwortzeit der Datenbankabfrage unter 10-facher Parallelast: **< 800 ms**
- p99-Antwortzeit unter 10-facher Parallelast: **< 1 500 ms**
- Keine der zehn Anfragen erhält HTTP 5xx.
- Der PostgreSQL Query-Plan für jede Teilabfrage zeigt Index-Nutzung; kein `Seq Scan` auf
  der vollen `ProductAttributeDecimal`- oder `Product`-Tabelle.

## Offene Frage
Dieses QAS definiert die Benchmark-Parameter für die Index-Strategie-Entscheidung in OQ-0003.
Wenn partielle Indizes oder `jsonb_path_ops`-GIN-Indizes die p95-Werte aus QAS-0001 und QAS-0002
unterschreiten, ersetzt ein aktualisiertes QAS dieses Dokument. Die Architekten-Review (OQ-0003)
entscheidet, ob und welche weitergehenden Indizes implementiert werden.

## Annahmen
- Alle zehn API-Consumer greifen auf denselben Workspace zu (worst case für Index-Contention).
- Der JSONB-Spiegel ist vollständig und aktuell.
- Datenbankverbindungspool ist auf ≥ 20 Verbindungen konfiguriert.
- Keine Transaktionskonflikte, da alle Anfragen lesend sind.
