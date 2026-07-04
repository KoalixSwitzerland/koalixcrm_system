# Use-Case-Verzeichnis

Alle Use Cases sind als Markdown-Dokumente unter `use_cases/use_case_*.md` abgelegt.
Jeder Use Case enthält PlantUML-Aktivitäts- und Ablaufdiagramme in eingebetteten Codeblöcken.

---

## Produktdomäne (ADR-0003 bis ADR-0008)

| ID       | Titel                                                             | ADR-Bezug         | Anforderungsbezug              |
|----------|-------------------------------------------------------------------|-------------------|-------------------------------|
| UC-0001  | Produkt anlegen und klassifizieren                                | ADR-0003, ADR-0004 | REQ-0001, REQ-0005, REQ-0008 |
| UC-0002  | Erweiterbare Attribute für ein Produkt pflegen                    | ADR-0004          | REQ-0009, REQ-0010            |
| UC-0003  | Produktübersetzung verwalten                                      | ADR-0003          | REQ-0003                      |
| UC-0004  | Preisliste und Produktpreise verwalten                            | ADR-0005          | REQ-0011, REQ-0012            |
| UC-0005  | Stückliste für ein Fertigprodukt pflegen                          | ADR-0006          | REQ-0015                      |
| UC-0006  | Dienstleistungsprofil für ein SERVICE-Produkt anlegen und bearbeiten | ADR-0007       | REQ-0016                      |
| UC-0011  | Produktfamilie mit Varianten anlegen und Attribute kaskadieren    | ADR-0021, ADR-0003, ADR-0004, ADR-0005 | REQ-0001, REQ-0002, REQ-0010, REQ-0011 |

## Lagerdomäne (ADR-0009 bis ADR-0015)

| ID       | Titel                                                             | ADR-Bezug                                      | Anforderungsbezug |
|----------|-------------------------------------------------------------------|------------------------------------------------|-------------------|
| UC-0007  | Mietangebot für eine Einzeleinheit erstellen und Verfügbarkeit prüfen | ADR-0009, ADR-0010, ADR-0011, ADR-0012, ADR-0013, ADR-0015 | —        |
| UC-0008  | Komponenten-Bestandssuche mit Stellplatz-Anzeige                      | ADR-0003, ADR-0009, ADR-0010, ADR-0011                      | —        |
| UC-0009  | Komponentenentnahme mit Bestandsbestätigung (Ad-hoc-Zykluszählung)    | ADR-0009, ADR-0010, ADR-0011, ADR-0012                      | —        |
| UC-0010  | Wareneingang mit Lieferschein und Lagerplatzvorschlag                 | ADR-0002, ADR-0003, ADR-0009, ADR-0011, ADR-0012            | —        |

## Produkt-/Lagerdomäne (spannt ADR-0006/ADR-0014 sowie ADR-0009 bis ADR-0011)

| ID       | Titel                                                             | ADR-Bezug                                      | Anforderungsbezug |
|----------|-------------------------------------------------------------------|------------------------------------------------|-------------------|
| UC-0012  | Kit kommissionieren und Fertigprodukt montieren                       | ADR-0014, ADR-0006, ADR-0019, ADR-0021, ADR-0009, ADR-0010, ADR-0011 | REQ-0015 |
