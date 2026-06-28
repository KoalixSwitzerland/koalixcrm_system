# ADR-0021: Produkt-Variantengranularität — Topologie, Schlüsselung und Attribut-Vererbungskaskade

## Status
Proposed

## Context

ADR-0003 führt `ProductFamily`, `Product` und `ProductVariant` als Backbone-Entitäten ein, ohne die
Beziehungsrichtung zwischen diesen drei Ebenen vollständig zu fixieren. Die ursprüngliche
Backbone-Beschreibung (ADR-0003, Zeilen 70–73) weist `ProductFamily` die Rolle zu, Varianten zu
gruppieren, und setzt den `ProductVariant`-FK auf `ProductFamily`. Diese Modellierung widerspricht
der gelebten Domänensemantik: Die verkaufbare Einheit leitet ihre Bedeutung aus dem abstrakten
Katalogobjekt (`Product`) ab, nicht aus der Produktlinie. Ohne eine kanonische Festlegung der
Vererbungsrichtung entstehen zudem konkurrierende Implementierungen der Frage, ob ein
Varianten-Attributwert auf Produkt- oder Familienebene gespeichert oder geerbt wird. ADR-0020
setzt die Kaskade bereits als Bindungsebene für Validierungsregeln voraus. ADR-0009 trägt einen
nullable FK auf `ProductVariant` neben einem FK auf `Product` in `OnHandRecord` — ein Smell, der
dasselbe Granularitätsproblem spiegelt.

## Decision

`ProductFamily` (optional) gruppiert `Product`-Objekte; jedes `Product` trägt ≥1
`ProductVariant`; `ProductVariant` trägt einen FK auf `Product` (nicht auf `ProductFamily`).
Attributwerte lösen sich zur Lesezeit nach der Kaskade **Varianten-Override → Produkt-Wert →
Familie-/AttributeSet-Standard** auf; der Effektivwert ist nicht materialisiert gespeichert.
Varianten-Override-Werte werden als nullable `variant_id` auf den getypten EAV-Wertetabellen
(ADR-0004) gespeichert (Option A). `ProductFamily` ist eine zusätzliche Bindungsachse für
`AttributeSet` neben `ClassificationNode` und `kind`.

## Why:

Die Kaskade Variante → Produkt → Familie entspricht dem Domänenmuster, das PIM-Systeme bei
10 000+ SKUs als wartbar belegen: Varianten-eigene Werte sind Ausnahmen, nicht die Regel; der
gemeinsame Stammsatz lebt auf `Product`. Option A erweitert die bestehenden EAV-Wertetabellen
ohne Schema-Proliferation und erhält die B-Tree-Indizierungsstrategie aus ADR-0004 vollständig.

## Alternatives Considered

- **`ProductVariant` FK → `ProductFamily` (ursprüngliche ADR-0003-Formulierung)** — abgelehnt:
  `ProductVariant` wäre nicht direkt mit dem beschreibenden `Product`-Objekt verbunden;
  Klassifizierung, `ServiceProfile`, `BillOfMaterials` und `ProductPassport` würden über einen
  zusätzlichen Join laufen; `ProductFamily` trüge zwei Verantwortlichkeiten (Gruppierung und
  Ankerpunkt).
- **Separate Variantenattribut-Tabellen (eigener EAV-Tabellensatz für Varianten-Overrides)**
  — abgelehnt: Schema-Proliferation, erschwerte B-Tree-Indexstrategie und komplexere
  Kaskadenabfragen gegenüber einem einzigen nullable `variant_id`-Join.
- **Materialisierter Effektivwert (Override-Werte als vollständige Zeile mit Inherited-Flag)**
  — abgelehnt: erzwingt Propagierung beim Schreiben (ein Produktwert-Update muss alle Varianten
  aktualisieren), verteilt die Konsistenzverantwortung auf mehrere Schreibpfade.
- **Override-zu-null als gültiger Zustand** — abgelehnt: ein Null-Override unterscheidet nicht
  zwischen „Wert nicht gesetzt" (INHERIT) und „Wert ist absichtlich leer"; diese Mehrdeutigkeit
  verhindert performante Filterabfragen und erzwingt anwendungsseitige Sonderbehandlung an jeder
  Lesestelle.

## Consequences

### Positive
- `ProductVariant` ist direkt über FK mit dem beschreibenden `Product` verbunden; alle
  Stammdaten-Anker (Klassifizierung ADR-0004, `ServiceProfile` ADR-0007, `BillOfMaterials`
  ADR-0006, `ProductPassport` ADR-0008) sind ohne zusätzlichen Join erreichbar.
- Die Kaskadenauflösung zur Lesezeit im Serializer ist eine deterministische, reine Funktion;
  keine Schreibpropagierung erforderlich.
- Option A (nullable `variant_id`) erweitert die bestehenden EAV-Wertetabellen ohne neue
  Tabellen; der zusammengesetzte Index wird von `(product_id, attribute_definition_id)` zu
  `(product_id, variant_id, attribute_definition_id)` erweitert.
- `ProductFamily` bleibt eine leichte Gruppierungsebene; alle Produkte einer Family tragen
  denselben `kind` (durchgesetzt per ADR-0019 `ProductKindPolicy`).
- `OnHandRecord` (ADR-0009) verwendet `ProductVariant` als autoritativen Lager-Schlüssel; der
  bisherige doppelte FK-Smell (nullable Variant neben Product) ist aufgelöst.
- Validierungsregeln (ADR-0020) binden entlang derselben Kaskade; kein zweiter
  Bindungsmechanismus erforderlich.

### Negative
- Alle Lese- und Schreibpfade, die `ProductPrice`, `OnHandRecord` und EAV-Wertetabellen an
  `Product` statt an `ProductVariant` binden, erfordern Datenmigration.
- Die Kaskadenauflösung muss in zwei Laufzeiten korrekt implementiert sein (Python-Serializer
  im Backend, TypeScript-Evaluator im Frontend gemäß ADR-0020); Abweichungen sind über die
  ADR-0020-Konformitätstestsuite zu erkennen.
- REQ-0001 AC-4 und REQ-0002 AC-2/AC-3 sind mit den hier fixierten Entscheidungen nicht
  konsistent; sie erfordern Korrektur durch den Requirements-Engineer (siehe
  §Folge-Aufgaben).

---

## 3-Ebenen-Topologie

`ProductFamily` (optional, workspace-scoped) fasst thematisch verwandte `Product`-Objekte zu
einer Linie zusammen. Ein `Product` gehört zu höchstens einer `ProductFamily`; ein `Product`
ohne `ProductFamily` ist ein gültiger Zustand (selten, aber erlaubt). Alle `Product`-Objekte
einer `ProductFamily` tragen denselben `kind`-Wert; diese Invariante wird in der
Applikationsschicht durch `ProductKindPolicy` (ADR-0019) durchgesetzt, nicht durch einen
Datenbank-Constraint.

`Product` (workspace-scoped) ist das abstrakte Katalogobjekt. Es trägt `kind`, `brand`, die
`ClassificationNode`-Verknüpfung, Attributdefinitionen und -templates sowie die
produktweiten Stammdaten-Anker (`ServiceProfile`, `BillOfMaterials`, `ProductPassport`). Jedes
`Product` besitzt ≥1 `ProductVariant`.

`ProductVariant` (workspace-scoped) ist die verkaufbare SKU. Sie trägt einen FK auf `Product`
(nicht auf `ProductFamily`). Alle handels- und logistikspezifischen Identifikatoren und
Eigenschaften leben auf `ProductVariant` (siehe Schlüsselungstabelle unten).

**Beispiel (Bäckerei):** `ProductFamily` „Brot" → `Product` „Weissbrot", „Vollkornbrot" →
`ProductVariant` „Weissbrot 250 g", „Weissbrot 500 g", „Weissbrot 1 kg".

---

## Schlüsselungstabelle — Was lebt wo

| Feld / Entität                          | Ebene            | Begründung                                                                                                        |
|-----------------------------------------|------------------|-------------------------------------------------------------------------------------------------------------------|
| `sku`                                   | `ProductVariant` | Lagerhaltungsnummer ist variantenspezifisch.                                                                      |
| `gtin`                                  | `ProductVariant` | GTIN ist die handelsseitige Einheiten-ID; verschiedene Verpackungsgrößen tragen je eigene GTINs.                  |
| `mpn`                                   | `ProductVariant` | Hersteller-Artikelnummer unterscheidet sich je Variante.                                                          |
| `weight_kg`, `dimensions_*`             | `ProductVariant` | Physische Maße unterscheiden sich je Verpackungseinheit.                                                          |
| `ProductPrice`                          | `ProductVariant` | Preis ist variantenspezifisch; `ProductPrice` trägt FK → `ProductVariant`.                                        |
| `OnHandRecord` / `tracking_mode`        | `ProductVariant` | Lagermengen und Trackingart sind auf Variantenebene; `OnHandRecord` FK → `ProductVariant` ist autoritativer Schlüssel. |
| `kind`                                  | `Product`        | Das `kind`-Enum (ADR-0003, ADR-0019) gilt produktweit; alle Varianten eines `Product` teilen dasselbe `kind`.    |
| `brand`                                 | `Product`        | Marke ist am Produkt verankert.                                                                                   |
| `ClassificationNode`-Verknüpfung        | `Product`        | `ProductClassification` (ADR-0004) FK → `Product`; Klassifizierung ist produktweit.                              |
| Attributdefinitionen / -template        | `Product`        | `AttributeSet`-Bindung (ADR-0004) gilt produktweit; Varianten überschreiben Werte, nicht Definitionen.           |
| `ServiceProfile`                        | `Product`        | 1:1 mit `Product` (`kind = SERVICE`); ADR-0007.                                                                   |
| `BillOfMaterials`                       | `Product`        | Stückliste am Produkt (`kind = MANUFACTURED_GOOD`); ADR-0006.                                                    |
| `ProductPassport`                       | `Product`        | DPP-Block per `Product`; ADR-0008.                                                                                |
| `ProductTranslation`                    | `Product`        | Mehrsprachige Bezeichnung am `Product`; ADR-0003.                                                                 |
| `ProductMedia`                          | **beide**        | Medien existieren auf `Product`- und auf `ProductVariant`-Ebene. Der effektive Medien-Pool einer Variante ist die Vereinigungsmenge (Union) von Produkt-Medien und Varianten-Medien. |

---

## Attribut-Vererbungskaskade

Die Auflösung des effektiven Attributwerts für eine gegebene Kombination aus Attributdefinition
und `ProductVariant` folgt dieser geordneten Priorität (höchste zuerst):

1. **Varianten-Override** — eine Zeile in der getypten EAV-Wertetabelle mit
   `variant_id = <variante>` und `product_id = <produkt>`.
2. **Produkt-Wert** — eine Zeile in derselben EAV-Wertetabelle mit `variant_id IS NULL` und
   `product_id = <produkt>`.
3. **Familie-/AttributeSet-Standard** — der über `AttributeSet` an `ProductFamily` oder
   `ClassificationNode`/`kind` gebundene Standardwert (ADR-0004).

Die Auflösung erfolgt zur Lesezeit im Serializer (Python-Backend) und im TypeScript-Evaluator
(Frontend). Kein materialisierter Effektivwert wird in der Datenbank vorgehalten.

### Zweiwertige Zustandssemantik

Pro Kombination aus (Entität, Attributdefinition) existieren genau zwei Zustände:

- **INHERIT** — keine Zeile mit `variant_id = <variante>` vorhanden; der nächste
  Kaskadenschritt gilt.
- **OVERRIDE** — eine Zeile mit `variant_id = <variante>` vorhanden; ihr konkreter Wert gilt.

Ein Override-zu-null existiert nicht. Ein Varianten-Override-Wert ist immer ein konkreter,
nicht-null Domänenwert. „Wert nicht gesetzt" ist ausschließlich durch Abwesenheit einer Zeile
(INHERIT) modellierbar.

### Null-Semantik und domänenexplizite None-Werte

Werte, die semantisch „keiner" bedeuten (z. B. ein Produkt ohne Ärmel, eine transparente
Farbe), werden als explizite, filterbare Domänenmitglieder modelliert — etwa ein Enum-Wert
`TRANSPARENT`, `SLEEVELESS` oder eine Dezimalzahl `0`. Diese Werte sind über B-Tree-Abfragen
filterbar und eindeutig von INHERIT zu unterscheiden. „Attribut trifft nicht zu" (Applicability)
ist von dieser Entscheidung nicht abgedeckt und auf ein späteres ADR zurückgestellt.

### Option A — nullable `variant_id` auf getypten EAV-Wertetabellen

Varianten-Override-Werte werden in denselben Wertetabellen gespeichert wie Produkt-Werte
(ADR-0004: `ProductAttributeDecimal`, `ProductAttributeString`, `ProductAttributeEnum`,
`ProductAttributeBool`, `ProductAttributeReference`, `ProductAttributeInt`). Jede dieser
Tabellen erhält einen nullable FK `variant_id → ProductVariant`. Der zusammengesetzte Index
auf jeder Wertetabelle lautet `(product_id, variant_id, attribute_definition_id)`. Eine Zeile
mit `variant_id IS NULL` ist der Produkt-Wert; eine Zeile mit `variant_id = <v>` ist der
Varianten-Override für Variante `v`.

### Familie als zusätzliche AttributeSet-Bindungsachse

`ProductFamily` ist eine zusätzliche Bindungsachse für `AttributeSet` neben `ClassificationNode`
und `kind` (ADR-0004). Ein `AttributeSet` kann an eine `ProductFamily`, einen
`ClassificationNode`, einen `kind`-Wert oder eine Kombination davon gebunden sein.
`ProductFamily`-gebundene Sets liefern die Standard-Attributwerte für alle zugeordneten Produkte
auf Kaskadenstufe 3.

---

## Validierungs-Bindung

Kreuzattribut-Validierungsregeln (ADR-0020) binden entlang derselben Kaskade: Regeln, die an
`AttributeSet`, `ProductFamily`, `kind` oder `ClassificationNode` gebunden sind, gelten auf
Kaskadenstufe 3; Varianten-Overrides auf Stufe 1 unterliegen denselben Regeln, wobei der per
Kaskade aufgelöste Effektivwert als Eingabe für den Validator dient.

---

## Explizit nicht im Geltungsbereich

Die folgenden Punkte sind aus diesem ADR ausgeschlossen und auf spätere Entscheidungen
zurückgestellt:

- **Abgeleitete / konfigurierte SKUs (Configure-to-Order)** — ein Stamm-SKU erzeugt pro Auftrag
  abgeleitete Varianten-SKUs, die typischerweise nicht auf Lager gehalten werden. Diese
  Entscheidung hängt vom CPQ-Ansatz ab.
- **Attribut-Anwendbarkeit (Applicability)** — „Attribut trifft auf diese Variante nicht zu"
  erfordert eine eigene Modellierungsentscheidung und hängt von der Configure-to-Order-Entscheidung
  ab.
- **Bestellzeit-CPQ / Guided Selling** — ADR-0020 §Ausschlüsse; gilt unverändert.
- **Regelformat und -vokabular für Validierungsregeln** — Folge-ADR zu ADR-0020; gilt
  unverändert.

---

## Folge-Aufgaben für andere Agenten

Der `kxcrm-requirements-engineer` korrigiert:

- **REQ-0001 AC-4** — listet `ProductFamily`, `ProductVariant`, `ProductTranslation` und
  `ProductMedia` mit FK-Beziehungen auf; abzugleichen mit der hier fixierten Topologie
  (insbesondere: `ProductVariant` FK → `Product`, nicht → `ProductFamily`).
- **REQ-0002 AC-2** — besagt, dass `ProductVariant` FK auf `ProductFamily` trägt; muss zu FK
  auf `Product` korrigiert werden.
- **REQ-0002 AC-3** — Varianten-Achsenwerte sind abzugleichen mit Option A (nullable
  `variant_id` auf EAV-Wertetabellen).

---

## Lizenzbeschränkung

Alle in diesem ADR beschriebenen Entitäten und Mechanismen leben vollständig im Open-Source-Backend
(`/app/koalixcrm`), das als PyPI-Wheel und Docker-Image ausgeliefert wird. Die
Kaskadenauflösung im Frontend (TypeScript) lebt ausschließlich im geschlossenen Docker-Target
(`/app/koalixcrm-frontend`). Das REST-API-Integrationsprotokoll (ADR-0002) bleibt die einzige
Kommunikationsbrücke; kein Domänen-Code überquert die Grenze.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Alle workspace-scoped Entitäten erben
`WorkspaceScopedModel`.

**ADR-0002 (Admin-UI-Framework):** Die Kaskadenauflösung wird über DRF-Endpunkte exponiert;
kein Domänen-Code im Frontend.

**ADR-0003 (Produkt-Katalog-Backbone):** Dieses ADR korrigiert die Backbone-Beschreibung
(Zeilen 70–73 von ADR-0003) via Amendment und ist die autoritative Quelle für die
3-Ebenen-Topologie; ADR-0003 bleibt in allen übrigen Punkten unverändert.

**ADR-0004 (Klassifizierung und erweiterbare Attribute):** Option A (nullable `variant_id`)
erweitert die getypten EAV-Wertetabellen. `ProductFamily` wird als zusätzliche
`AttributeSet`-Bindungsachse eingeführt.

**ADR-0005 (Preisgestaltung und Maßeinheiten):** `ProductPrice` FK → `ProductVariant` (statt
bisher FK → `Product`).

**ADR-0006 (Beschaffung und Stücklisten):** `BillOfMaterials` verbleibt auf `Product`-Ebene;
unverändert.

**ADR-0007 (Dienstleistungsprofil):** `ServiceProfile` verbleibt auf `Product`-Ebene;
unverändert.

**ADR-0008 (Digital Product Passport):** `ProductPassport` verbleibt auf `Product`-Ebene;
unverändert.

**ADR-0009 (Lager-Domänen-Backbone):** `OnHandRecord` FK → `ProductVariant` ist der
autoritative Lager-Schlüssel; der bisherige Zustand (nullable FK auf Variant neben FK auf
Product) ist durch dieses ADR aufgelöst. `tracking_mode` lebt auf `ProductVariant`.

**ADR-0019 (Produkt-`kind`-Invarianten):** Die Invariante, dass alle Produkte einer
`ProductFamily` denselben `kind` tragen, wird in der Applikationsschicht durch
`ProductKindPolicy` (ADR-0019) durchgesetzt.

**ADR-0020 (Gemeinsame deklarative Attributvalidierung):** Validierungsregeln binden entlang
der hier definierten Kaskade (Variante → Produkt → Familie/AttributeSet).

## Changelog
- 2026-06-28: Erstentscheidung (Status: Proposed). Fixiert die 3-Ebenen-Topologie,
  Schlüsselung, Attribut-Vererbungskaskade, zweiwertige Zustandssemantik, Option A und
  Validierungs-Bindung. Korrigiert ADR-0003 (Zeilen 70–73) via Amendment; erweitert ADR-0004
  (Option A, Family-Achse), ADR-0005 (ProductPrice → Variant) und ADR-0009 (OnHandRecord →
  Variant autoritativ, tracking_mode → Variant) via Amendment.
