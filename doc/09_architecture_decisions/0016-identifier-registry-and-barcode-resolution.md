# ADR-0016: Identifier-Registry und Barcode-Auflösung

## Status
Accepted

## Context

UC-0008 (Lagerbestand per Barcode lokalisieren), UC-0009 (Komponentenentnahme mit
Bestandsbestätigung) und UC-0010 (Wareneingang mit Lieferschein und Lagerplatzvorschlag)
erfordern, dass ein gescannter Barcode-Wert eindeutig gegen mehrere Identifier-Typen aufgelöst
wird: `Product.gtin` (GS1 GTIN), `Location.external_ref` (ADR-0009), `SerialUnit.global_uid`
(ADR-0012) sowie `HandlingUnit.sscc` (ADR-0009). Ein Barcode-Wert kann gleichzeitig als GS1
Application Identifier (AI)-kodierter Compound String oder als freier Text vorliegen. Kein
bestehendes ADR (ADR-0009 bis ADR-0015) definiert eine kanonische Auflösungsreihenfolge oder
eine zentrale Scan-Endpunktschicht. Ohne eine festgelegte Auflösungsregel können zwei
Feature-Teams denselben Wert unterschiedlich interpretieren; Kollisionen (ein Freitext-Code,
der zufällig einer GTIN entspricht) wären unspezifiziert.

## Decision

Der Backend-Dienst stellt einen einzigen kanonischen Endpunkt
`POST /api/v1/scan/resolve` bereit, der einen gescannten Code auflöst und das Zielobjekt
mit Typ und ID zurückgibt. Die Auflösungslogik folgt einer festen, zweistufigen Reihenfolge:
zuerst GS1 AI-Parsing, dann freier Textabgleich.

**Stufe 1 — GS1 Application Identifier-Parsing:**

Die Auflösung prüft, ob der eingehende Code einen erkennbaren GS1 AI-Präfix trägt, und löst
nach folgendem Schema auf:

| GS1 AI       | Identifier-Typ | Zielobjekt                              |
|--------------|----------------|-----------------------------------------|
| `(01)`       | GTIN           | `Product` (über `Product.gtin`)         |
| `(00)`       | SSCC           | `HandlingUnit` (über `HandlingUnit.sscc`) |
| `(01)` + `(21)` | SGTIN       | `SerialUnit` (über `SerialUnit.global_uid`) |
| `(8003)`     | GIAI           | `SerialUnit` (über `SerialUnit.global_uid`) |
| `(414)`      | GLN            | `Location` (über `Location.external_ref`) |

Wird ein GS1 AI-Präfix erkannt, läuft ausschließlich die GS1-Auflösung. Der freie
Textabgleich (Stufe 2) wird nicht gestartet.

**Stufe 2 — Freier Textabgleich (nur wenn kein GS1 AI erkannt):**

Exact-Match-Suche in dieser Reihenfolge:
1. `Location.external_ref`
2. `SerialUnit.global_uid`
3. `Product.sku`

Der erste Treffer gewinnt. Wird derselbe Code in mehr als einer Entität gefunden (Multi-Hit),
antwortet der Endpunkt mit HTTP 409 und liefert die Liste der Kandidaten.

**Antwortformat:**

```
{ "kind": "product" | "handling_unit" | "serial_unit" | "location",
  "id": "<uuid>",
  "matched_field": "<Feldname>" }
```

Kein Treffer ergibt HTTP 404.

**Workspace-Scoping:**

Die Auflösung ist auf den Workspace des aufrufenden Benutzers beschränkt. Ausnahme:
`Product.gtin` ist katalogweit gültig (nicht workspace-gebunden, ADR-0003); ein GTIN-Treffer
liefert eine `Product`-Referenz, impliziert aber keinen `OnHandRecord`-Kontext. Alle anderen
Identifier-Typen sind workspace-scoped (ADR-0001).

## Why

Die GS1 AI-Struktur ist durch ihren Präfix zuverlässig maschinenlesbar erkennbar und trennt
codierte Identifier von freiem Text, ohne eine externe Registry oder ein Vertragsmodell zwischen
Mandanten zu erfordern. Den GS1-Pfad vor dem Freitext-Pfad auszuführen — statt beide parallel
zu prüfen — verhindert, dass ein freier `Location.external_ref`-Wert, der zufällig dieselbe
Zeichenfolge wie eine GTIN enthält, den GS1-Treffer verdeckt. Ein einziger kanonischer Endpunkt
statt verteilter Suchanfragen pro Feature-Team sichert die Konsistenz der Auflösungsreihenfolge
plattformweit.

## Alternatives Considered

- **Eigenständige `IdentifierRegistry`-Entität als zentrale Lookup-Tabelle** — abgelehnt: eine
  zusätzliche Tabelle erfordert Synchronisierung bei jeder Anlage und Änderung von `Product`,
  `SerialUnit`, `Location` und `HandlingUnit`; konsistenter Betrieb ist schwerer sicherzustellen
  als direkte Feldabfragen; kein struktureller Gewinn gegenüber direkten Index-Lookups.
- **Paralleler Abgleich aller Identifier-Typen ohne Reihenfolge** — abgelehnt: ohne feste
  Priorität ist das Verhalten bei Kollisionen unspezifiziert; HTTP 409 bei jedem
  Mehrdeutigkeitsfall, der durch Reihenfolge vermieden werden kann, verschlechtert die
  Benutzererfahrung.
- **Vollständige GS1-EPCIS-Repository-Implementierung mit ALE/APDL-Endpunkten** — abgelehnt:
  unverhältnismäßiger Aufwand für das aktuelle Nutzerprofil; das Backend muss als PyPI-Wheel
  installierbar bleiben (ADR-0011); ein spezialisiertes EPCIS-Repository kann als separates
  Modul nachgerüstet werden.
- **Keine zentrale Auflösungsschicht; jeder Feature-Endpunkt löst selbst auf** — abgelehnt:
  dupliziert die Auflösungslogik, erzeugt abweichendes Verhalten bei Kollisionen pro Feature,
  und erschwert zukünftige Erweiterungen (neue Identifier-Typen) auf mehrere Endpunkte
  verteilt.

## Consequences

### Positive
- UC-0008, UC-0009 und UC-0010 nutzen denselben Endpunkt und dieselbe Auflösungsreihenfolge;
  kein Auseinanderdriften der Scan-Semantik zwischen Features.
- Die GS1 AI-Priorisierung verhindert Kollisionen zwischen strukturierten GS1-Codes und freien
  Texten ohne manuellen Konfigurationsaufwand.
- Der Endpunkt ist workspace-scoped; kein mandantenübergreifender Datenleck.
- Die GS1 AI-Parsing-Funktion ist als kleine In-Repo-Funktion oder mit einer MIT-lizenzierten
  Bibliothek ohne kommerzielle Abhängigkeit implementierbar; das PyPI-Wheel bleibt
  uneingeschränkt verteilbar (ADR-0002).

### Negative
- Ein `Location.external_ref`-Wert, der strukturell einem GS1 AI-String ähnelt, wird durch
  die GS1-Stufe aufgegriffen; Mandanten müssen darauf hingewiesen werden, freie
  `external_ref`-Werte nicht im AI-Format zu vergeben.
- Der Endpunkt liefert nur die Primäridentifikation; nachgelagerte Kontextabfragen (z. B.
  `OnHandRecord`-Details nach einem `SerialUnit`-Treffer) erfordern einen zweiten API-Aufruf.
- HTTP 409-Verhalten bei Freitext-Multi-Hits erfordert, dass der aufrufende Client die
  Kandidatenliste verarbeiten kann.

---

## Endpunkt-Spezifikation

**`POST /api/v1/scan/resolve`**

Request-Body:
```json
{ "code": "<gescannter Barcode-Wert>" }
```

Erfolgsantwort (HTTP 200):
```json
{
  "kind": "product" | "handling_unit" | "serial_unit" | "location",
  "id": "<uuid>",
  "matched_field": "<Feldname>"
}
```

Fehlerantworten:
- HTTP 404: kein Treffer in keiner Stufe.
- HTTP 409: Freitext-Stufe liefert mehr als einen Treffer; Antwort enthält Array `"candidates"`.

---

## Workspace-Scoping-Matrix

| Auflösungspfad              | Scoping          | Begründung                                                                             |
|-----------------------------|------------------|----------------------------------------------------------------------------------------|
| `Product.gtin` (GTIN/SGTIN) | global (Katalog) | GTINs sind katalogweit; kein Mandantenbezug im Produkt selbst (ADR-0003)               |
| `SerialUnit.global_uid`     | workspace        | Einheiten sind mandantenspezifische Anlagendaten (ADR-0012)                            |
| `HandlingUnit.sscc`         | workspace        | SSCC-Vergabe ist mandantenspezifisch (ADR-0009)                                        |
| `Location.external_ref`     | workspace        | Lagerstruktur ist mandantenspezifisch (ADR-0009)                                       |
| `Product.sku` (Freitext)    | workspace        | SKUs sind im `Product`-Workspace-Scope geführt (ADR-0003)                              |

---

## Lizenzbeschränkung

Dieses ADR führt keine neue Closed-Source-Abhängigkeit ein. Die GS1 AI-Parsing-Logik lebt
vollständig im Open-Source-Backend (`/app/koalixcrm`). GS1 Application Identifiers sind
ein offener Standard; eine optionale MIT-lizenzierte Hilfsbibliothek oder eine handgeschriebene
Parser-Funktion sind beide zulässig. Das Backend bleibt als PyPI-Wheel für Drittinstallationen
verteilbar. Das REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige
Kommunikationsbrücke zum Frontend.

---

## Standards-Verankerung

| Standard         | Verwendung in diesem ADR                                                                  |
|------------------|-------------------------------------------------------------------------------------------|
| GS1 Application Identifiers | Primärauflösungsschema: AI (01) GTIN, AI (00) SSCC, AI (21) SGTIN, AI (8003) GIAI, AI (414) GLN |
| GS1 SSCC         | `HandlingUnit.sscc` (ADR-0009); AI (00)-Auflösung                                        |
| GS1 GTIN         | `Product.gtin` (ADR-0003); AI (01)-Auflösung                                             |
| ISO/IEC 15459    | `SerialUnit.global_uid` (ADR-0012); AI (21) SGTIN und AI (8003) GIAI-Auflösung           |
| GS1 GLN          | `Location.external_ref` (ADR-0009); AI (414)-Auflösung                                   |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0003 (Produkt-Katalog-Backbone):** `Product.gtin` und `Product.sku` sind die Zielfelder
für GTIN- und SKU-basierte Auflösung.

**ADR-0009 (Lager-Domänen-Backbone):** `Location.external_ref` und `HandlingUnit.sscc` sind
Zielfelder für GLN/SSCC-Auflösung. Der `LAYER`-Enum-Wert (ADR-0009, Amendment 2026-05-04)
ist über GLN-Auflösung erreichbar.

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummernverfolgung):** `SerialUnit.global_uid`
ist das Zielfeld für SGTIN- und GIAI-Auflösung.

**ADR-0013 (Miet- und Kundengeführter Bestand):** Workspace-scoped Auflösung ist konsistent
mit dem mandantenspezifischen Scoping aus ADR-0001/0013.

**ADR-0021 (Produkt-Variantengranularität):** AI `(01)` GTIN und Freitext-Stufe-2-Regel 3
lösen gegen `ProductVariant.gtin`/`ProductVariant.sku` auf (Nachtrag 2026-07-04); vollzieht die
ADR-0021-Ripple-Liste für die Identifier-Registry nach.

## Changelog
- 2026-05-04: Erstentscheidung. OQ-0016 geschlossen. UC-0008, UC-0009, UC-0010 als auslösende Use Cases.
- 2026-07-04: Nachtrag — OQ-0022 geschlossen: AI `(01)` GTIN löst gegen `ProductVariant.gtin`
  auf (statt `Product.gtin`); Freitext-Stufe-2-Regel 3 löst gegen `ProductVariant.sku` auf
  (statt `Product.sku`); Antwort-`kind` liefert `product_variant` statt `product`. Siehe
  Nachtrag 2026-07-04.

---

## Nachtrag (2026-07-04): GTIN- und SKU-Auflösung → `ProductVariant` (ADR-0021, OQ-0022)

ADR-0021 platziert `gtin` und `sku` auf `ProductVariant`-Ebene ("GTIN ist die handelsseitige
Einheiten-ID; verschiedene Verpackungsgrößen tragen je eigene GTINs"; „Lagerhaltungsnummer ist
variantenspezifisch"). Die ursprüngliche Fassung dieses ADR löst AI `(01)` GTIN gegen
`Product.gtin` und die Freitext-Stufe-2-Regel 3 gegen `Product.sku` auf; keine der beiden
ADR-0021-Ripple-Listen nannte ADR-0016. UC-0009 (Änderungsprotokoll 2026-07-04, BAC-1) setzt
in seinem Auflösungsschritt bereits `ProductVariant.gtin` voraus und widerspricht damit dem
bisherigen ADR-0016-Wortlaut.

**Änderung — Stufe 1 (GS1 AI-Parsing):** AI `(01)` GTIN löst gegen `ProductVariant.gtin`
statt gegen `Product.gtin` auf. Die Zeile `(01) + (21)` SGTIN → `SerialUnit` (über
`SerialUnit.global_uid`) bleibt unverändert; `SerialUnit` ist bereits über ADR-0012 (Amendment
2026-07-04) auf `ProductVariant` geschlüsselt.

**Änderung — Stufe 2 (Freitext), Regel 3:** Der dritte Freitext-Abgleichsschritt löst gegen
`ProductVariant.sku` statt gegen `Product.sku` auf. Die Regeln 1 (`Location.external_ref`) und
2 (`SerialUnit.global_uid`) bleiben unverändert.

**Sweep der übrigen Auflösungsregeln gegen die ADR-0021-Schlüsselungstabelle:** Die verbleibenden
Zeilen von Stufe 1 — AI `(00)` SSCC → `HandlingUnit.sscc` (ADR-0009) und AI `(414)` GLN →
`Location.external_ref` (ADR-0009) — referenzieren keine `Product`- oder
`ProductVariant`-Felder und bleiben unverändert. AI `(8003)` GIAI → `SerialUnit.global_uid`
bleibt aus demselben Grund wie SGTIN unverändert (transitiv bereits variantengekeyt über
ADR-0012). Kein weiteres in ADR-0016 referenziertes Feld verweist auf `Product`-Ebene; die
Schlüsselungstabelle in ADR-0021 platziert nur `gtin`, `sku` und `mpn` auf `ProductVariant` —
`mpn` wird von ADR-0016 nicht referenziert und erfordert keine Änderung.

**Antwortformat:** Der Erfolgsantwort-`kind`-Wert für einen GTIN- oder SKU-Treffer wechselt
von `"product"` auf `"product_variant"`; `id` referenziert die getroffene `ProductVariant`,
nicht mehr das übergeordnete `Product`. Die übrigen `kind`-Werte (`handling_unit`,
`serial_unit`, `location`) bleiben unverändert.

**Workspace-Scoping:** Die Ausnahme „`Product.gtin` ist katalogweit gültig (nicht
workspace-gebunden)" überträgt sich unverändert auf `ProductVariant.gtin`: `ProductVariant` ist
zwar workspace-scoped (ADR-0021), die GTIN-Auflösung bleibt jedoch katalogweit — ein
GTIN-Treffer liefert eine `ProductVariant`-Referenz unabhängig vom aufrufenden Workspace,
impliziert aber weiterhin keinen `OnHandRecord`-Kontext (dieser bleibt workspace-scoped). Die
Freitext-SKU-Auflösung (Regel 3) bleibt workspace-scoped, da sie bereits vor diesem Nachtrag
workspace-scoped war (`Product.sku` im `Product`-Workspace-Scope geführt) und `ProductVariant`
ebenfalls workspace-scoped ist.

Damit ist OQ-0022 geschlossen; siehe ADR-0021 §Ripple-Liste.
