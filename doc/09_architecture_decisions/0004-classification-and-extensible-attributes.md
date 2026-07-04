# ADR-0004: Klassifizierung und erweiterbare Attribute (getypte EAV-Tabellen)

## Status
Accepted

## Context

KoalixCRM bedient Mandanten aus unterschiedlichen Branchen im selben System. Ein
Lebensmittelhersteller benötigt andere Pflichtfelder (Nährwerte, Allergene) als ein Lackhersteller
(RAL-Farbe, VOC-Gehalt). Neue Attribute für eine Branche dürfen keine Schemaänderung und keinen
Code-Deploy erfordern. Gleichzeitig müssen Attributwerte bei 10 000+ SKUs über B-Tree-Indizes
abfragbar sein; Volltextsuche und Attributfilter auf Listenseiten müssen performant bleiben.
Industrie-Klassifizierungsschemata (UNSPSC, eCl@ss, intern) sollen koexistieren; ein Produkt kann
unter mehreren Taxonomien gleichzeitig eingehängt sein. Der Katalog-Backbone ist in ADR-0003
definiert; das vorliegende ADR legt fest, wie Klassifizierung und Attributsystem darauf aufsetzen.

## Decision

Klassifizierungsschemata, Attributdefinitionen und Attributwerte werden in einem dreistufigen
Modell organisiert: (1) `Classification`/`ClassificationNode` als globale Taxonomiebäume,
(2) `AttributeDefinition` mit `scope`-Flag, `AttributeGroup` und `AttributeSet` als
konfigurierbare Metadaten-Schicht, (3) typisierte Wertetabellen — je eine Tabelle pro
Datentyp (`ProductAttributeDecimal`, `ProductAttributeString`, `ProductAttributeEnum`,
`ProductAttributeBool`, `ProductAttributeReference`, `ProductAttributeInt`) — statt einer
einzigen polymorphen EAV-Tabelle. Ein optionaler denormalisierter JSONB-Spiegel pro Produkt
unterstützt leseintensive Such- und Listenpfade.

## Why

Ausschließlich typisierte Wertetabellen mit je einem zusammengesetzten Index auf
`(product_id, attribute_definition_id)` ermöglichen B-Tree-Abfragen bei 10 000+ SKUs ohne
Cast-Ausdrücke zur Laufzeit; gleichzeitig entstehen neue Branchenattributgruppen per Admin-Aktion
ohne Schema-Migration (siehe OQ-0003 für weitergehende Indexfragen).

## Alternatives Considered

- **JSON-Blob pro Produkt** — abgelehnt: keine Typisierung, keine Einheitenbeziehung, keine
  automatisch generierbare UI-Metadaten, langsame Volltextsuche ab 10 000 SKUs, kein strukturierter
  Export nach GDSN oder DPP-Schema.
- **Polymorphe EAV-Wertetabelle (einzelne `value`-Spalte, Casts zur Laufzeit)** — abgelehnt:
  Abfrageperformance bricht bei 10 000+ SKUs zusammen, da Cast-Ausdrücke keine B-Tree-Indizes
  nutzen. Erfahrungen aus Akeneo- und Magento-Deployments belegen diesen Effekt.

## Consequences

### Positive
- Neue Branchen-Attributgruppen entstehen per Admin-Aktion ohne Code-Änderung oder
  Datenbank-Migration.
- Typisierte Wertetabellen halten B-Tree-Indizes klein und abfragbar; Attributfilter auf
  Listenseiten bleiben bei 10 000+ SKUs performant.
- Klassifizierungsschemata (UNSPSC, eCl@ss, intern) koexistieren; ein Produkt ist unter mehreren
  Taxonomien gleichzeitig eingehängt.
- `AttributeSet`-Metadaten liefern dem Frontend über den DRF-Endpunkt die Feldliste, Reihenfolge,
  Pflichtfeld-Flags und Validierungsregeln für das Produktbearbeitungsformular (ADR-0002).

### Negative
- Fehlende Indizes auf den getypten Wertetabellen machen den Performancegewinn zunichte; der
  implementierende Engineer legt je Wertetabelle einen zusammengesetzten Index auf
  `(product_id, attribute_definition_id)` an (siehe OQ-0003).
- Das Modell ist umfangreicher als ein einfaches JSON-Feld; Onboarding neuer Entwickler
  erfordert Kenntnis der dreistufigen Metadaten-Schicht.
- eCl@ss-Inhalte (Code-Listen) unterliegen einer kommerziellen Mitgliedslizenz und dürfen
  nicht als Open-Source-Fixtures im PyPI-Paket enthalten sein (siehe OQ-0001).

---

## Klassifizierungsschemata

**`Classification`** (global) beschreibt ein Klassifizierungsschema: UNSPSC, eCl@ss, intern oder
ein benutzerdefiniertes Schema. Mehrere Schemata koexistieren.

**`ClassificationNode`** (global) repräsentiert einen Knoten im hierarchischen Baum eines Schemas
(z. B. UNSPSC-Segment → Familie → Klasse → Waren-/Dienstleistungsgruppe).

**`ProductClassification`** (workspace-scoped) ist die M:N-Verknüpfung zwischen `Product`
(ADR-0003) und `ClassificationNode`; ein Produkt ist unter mehreren Taxonomien gleichzeitig
eingehängt.

---

## Attributdefinitionen

**`AttributeDefinition`** (global für GLOBAL-Scope, workspace-scoped für WORKSPACE-Scope) trägt:
Datentyp (`string | int | decimal | bool | enum | measure | reference | json`), optionalen FK auf
`core.Unit`, Validierungsregeln (min, max, regex, enum-Werte), Flags `is_localized`,
`is_required`, `is_multivalued` sowie einen **`scope`-Flag**:

- `GLOBAL` — plattformweit gültig, von GS1 GPC / eCl@ss Advanced oder UNSPSC abgeleitet,
  als Fixture ausgeliefert.
- `WORKSPACE` — mandantendefiniert, vom Tenant-Admin angelegt.
- `INHERITED` — von einem `ClassificationNode` übernommen.

**`AttributeGroup`** bündelt thematisch zusammengehörige `AttributeDefinition`-Einträge (z. B.
„Nährwerte je 100 g" oder „Beschichtungsparameter").

**`AttributeSet`** (workspace-scoped) ordnet einer Kombination aus `ClassificationNode` und/oder
`kind`-Enum (ADR-0003) eine Menge von `AttributeGroup`-Einträgen zu. Jedes unter diesem Knoten
klassifizierte Produkt erbt die im Set definierten Felder inklusive Pflichtfeldflag und
Validierungsregeln — ohne Code-Änderung.

---

## Getypte Wertetabellen

Attributwerte werden in je einer Tabelle pro Datentyp gespeichert:

| Tabelle                      | Datentyp               |
|------------------------------|------------------------|
| `ProductAttributeDecimal`    | Dezimalzahl + Einheit  |
| `ProductAttributeString`     | Zeichenkette           |
| `ProductAttributeEnum`       | Enum-Schlüssel         |
| `ProductAttributeBool`       | Boolean                |
| `ProductAttributeReference`  | FK auf Lookup-Entität  |
| `ProductAttributeInt`        | Ganzzahl               |

Jede Wertetabelle trägt einen zusammengesetzten Index auf `(product_id, attribute_definition_id)`.
Ein optionaler denormalisierter JSONB-Spiegel pro Produkt unterstützt leseintensive Such- und
Listenpfade und wird von einem Signal oder einer Celery-Task aktuell gehalten.

---

## Konkrete Branchenbeispiele

**Lebensmittelhersteller — Nährwerte und Vitamine:** Der Admin legt `AttributeGroup
"Nährwerte je 100 g"` an (Energie_kJ, Fett_gesamt, Fett_gesättigt, Kohlenhydrate, Zucker,
Protein, Salz — alle `decimal`, Einheit je Attribut) und `AttributeGroup "Vitamine"` (Vitamin_A,
Vitamin_B1, … `decimal`, Einheit mg/µg). Beide Gruppen werden an `AttributeSet "Lebensmittel"`
gebunden, das am `ClassificationNode "Lebensmittel"` hängt. Jedes unter diesem Knoten
klassifizierte Produkt erbt die Felder mit Pflichtfeldflag und Einheitenvalidierung — keine
Code-Änderung, keine Schema-Migration.

**Lackhersteller — RAL-Farbe und Beschichtungsattribute:** Der Admin definiert `ral_color`
(`reference` auf Lookup-Entität `RalColor(code, name, hex)` oder `enum`), `gloss_level` (enum:
matt / satin / gloss), `coverage_m2_per_l` (`decimal`, Einheit m²/l), `voc_content` (`decimal`,
Einheit g/l), `base` (enum: water / solvent). Die Attributgruppe wird an `AttributeSet
"Beschichtungen"` am `ClassificationNode "Coatings"` gebunden. Dieselbe Maschinerie, völlig
andere Branche.

---

## Workspace-Scoping-Matrix

| Entität                                      | Scoping   |
|----------------------------------------------|-----------|
| `Classification`                             | global    |
| `ClassificationNode`                         | global    |
| `AttributeDefinition` (GLOBAL-Scope)         | global    |
| `AttributeGroup` (GLOBAL-Scope)              | global    |
| GS1-GTIN/UNSPSC-Codelisten (Fixtures)        | global    |
| `ProductClassification`                      | workspace |
| `AttributeDefinition` (WORKSPACE-Scope)      | workspace |
| `AttributeGroup` (WORKSPACE-Scope)           | workspace |
| `AttributeSet`                               | workspace |
| `ProductAttributeDecimal/String/Enum/…`      | workspace |

Workspace-scoped Entitäten erben den `WorkspaceScopedModel`-Mechanismus aus ADR-0001. Globale
Stamm- und Nachschlagedaten werden als Fixtures oder Datenmigration ausgeliefert.

---

## Lizenzbeschränkung

Dieses Modell lebt vollständig im Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und
Docker-Image ausgeliefert wird.

Klassifizierungsinhalte (Code-Listen) dürfen nur dann als Open-Source-Fixtures mitgeliefert
werden, wenn ihr Lizenzstatus dies erlaubt:

- **UNSPSC** — frei nutzbar und weiterverteilbar. Kann als Fixture ausgeliefert werden.
- **eCl@ss** — unterliegt einer kommerziellen Mitgliedslizenz. eCl@ss-Code-Listen dürfen **nicht**
  als Open-Source-Fixtures im PyPI-Paket enthalten sein. Das Modell unterstützt eCl@ss technisch
  über `Classification` + `ClassificationNode`; der Inhalt muss vom Betreiber separat importiert
  werden (siehe OQ-0001).

Das REST-API-Integrationsprotokoll zwischen dem Open-Source-Backend und dem geschlossenen
Next.js-Frontend (ADR-0002) bleibt die einzige Kommunikationsbrücke. Kein Domänen-Code darf
direkt in den Frontend-Build importiert werden.

---

## Standards-Verankerung

| Standard                          | Verwendung im Modell                                              |
|-----------------------------------|-------------------------------------------------------------------|
| UNSPSC 8-stellig (4 Ebenen)       | `Classification` + `ClassificationNode`; als Fixture lieferbar   |
| eCl@ss 10-stellig (ISO/IEC)       | `Classification` + `ClassificationNode`; Inhalt separat lizenziert (OQ-0001) |
| GS1 GPC Bricks / eCl@ss Advanced | Quelle für GLOBAL-scope `AttributeDefinition`-Einträge           |
| PIM Family/Variant + EAV mit getypten Tabellen | Architekturmuster (Akeneo / Pimcore / Inriver)    |

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0001 (Kontakt- und Partei-Datenmodell):** Workspace-scoped Entitäten erben den
`WorkspaceScopedModel`-Mechanismus.

**ADR-0002 (Admin-UI-Framework):** Das `AttributeSet`-gesteuerte dynamische Formular-Rendering
ist der zentrale Anwendungsfall für Refine-Ressourcen im Next.js-Admin. Jedes `AttributeSet`
liefert dem Frontend über den DRF-Endpunkt die Feldliste, Reihenfolge, Pflichtfeld-Flags und
Validierungsregeln. Die Konfiguration dieser Formular-Metadaten liegt ausschließlich im
Python-Backend als `AttributeSet`-Objekte in der Datenbank, nicht in TSX-Ressourcendeklarationen.

**ADR-0003 (Produkt-Katalog-Backbone):** `ProductClassification` verknüpft `Product` mit
`ClassificationNode`. Das `kind`-Enum aus ADR-0003 dient als optionale Achse für `AttributeSet`.
Die Entscheidung, keine Produktunterklassen pro Branche einzuführen (ADR-0003), ist die direkte
Voraussetzung für das vorliegende Attributsystem.

## Changelog
- 2026-05-03: Erstentscheidung. Herausgelöst aus dem vormaligen omnibus ADR-0003
  (Produkt-Katalog-Domänenmodell).
- 2026-05-05: Amendment — Schichtenmodell, Adapter-Rolle von Klassifizierungsstandards und
  Lizenzbeschränkung präzisiert (siehe unten).
- 2026-06-28: Amendment — Variantenebene in EAV-Wertetabellen (Option A) und
  `ProductFamily` als AttributeSet-Bindungsachse per ADR-0021 eingeführt (siehe unten).

---

## Amendment 2026-06-28 — Variantenebene in EAV-Wertetabellen und Family als AttributeSet-Achse (ADR-0021)

ADR-0021 legt die Attribut-Vererbungskaskade und die Variantengranularität fest und erweitert
das vorliegende EAV-Modell in zwei Punkten. ADR-0021 ist die autoritative Quelle; das
vorliegende Amendment dokumentiert die Auswirkungen auf ADR-0004.

### Option A — nullable `variant_id` auf getypten Wertetabellen

Jede der sechs getypten EAV-Wertetabellen (`ProductAttributeDecimal`, `ProductAttributeString`,
`ProductAttributeEnum`, `ProductAttributeBool`, `ProductAttributeReference`,
`ProductAttributeInt`) erhält einen nullable FK `variant_id → ProductVariant`. Der
zusammengesetzte Index auf jeder Wertetabelle wird von `(product_id, attribute_definition_id)`
zu `(product_id, variant_id, attribute_definition_id)` erweitert.

Eine Zeile mit `variant_id IS NULL` ist der Produkt-Wert. Eine Zeile mit `variant_id = <v>`
ist der Varianten-Override für Variante `v`. Kein separater EAV-Tabellensatz für Varianten
entsteht; die B-Tree-Indizierungsstrategie aus dem ursprünglichen ADR bleibt vollständig
erhalten.

### `ProductFamily` als zusätzliche AttributeSet-Bindungsachse

`AttributeSet` (bisher an `ClassificationNode` und/oder `kind` gebunden) akzeptiert
`ProductFamily` als dritte Bindungsachse. Ein `AttributeSet` kann an eine `ProductFamily`,
einen `ClassificationNode`, einen `kind`-Wert oder eine Kombination gebunden sein.
`ProductFamily`-gebundene Sets liefern Standard-Attributwerte auf Kaskadenstufe 3 der
Vererbungskaskade (ADR-0021: Varianten-Override → Produkt-Wert → Familie-/AttributeSet-Standard).

### Kaskadenauflösung zur Lesezeit

Der JSONB-Spiegel und der Serializer-Lesepfad lösen den effektiven Attributwert zur Lesezeit
nach der in ADR-0021 definierten dreistufigen Kaskade auf. Kein materialisierter Effektivwert
wird als eigene Zeile gespeichert.

---

## Amendment 2026-05-05 — Schichtenmodell und Adapter-Rolle der Klassifizierungsstandards

Die ursprüngliche Entscheidung (dreistufiges EAV-Modell) bleibt unverändert. Dieses Amendment
präzisiert die Rolle der Klassifizierungsstandards und führt das Schichtenmodell explizit ein.

### Schichtenmodell

Das Produktattribut-Modell ist in drei Schichten gegliedert:

- **Layer 1 — Minimal Core Product**: Pflichtfelder direkt auf dem `Product`-Objekt
  (ADR-0003): `name` / `ProductTranslation`, `description`, `gtin`, `base_uom`,
  `weight_kg` (Kandidat für kanonisches Vokabular), Dimensions-Felder (optional), `tax_class`,
  `manufacturer_party`, `country_of_origin`. Diese Felder sind direkte Spalten auf `Product`,
  nicht EAV. Sie sind immer vorhanden und stets lizenzfrei.
- **Layer 2 — Operator-definierte Attribute**: Der EAV-Mechanismus dieses ADR. Betreiber
  definieren eigene `AttributeDefinition`-Einträge für ihren Katalog, ohne
  Schemaänderung und ohne Code-Deploy. Lizenzfrei.
- **Layer 3 — Klassifizierungsgesteuerte Attributvorlagen**: Opt-in-Plug-in-Slot. GPC,
  UNSPSC, eCl@ss und ETIM liefern als Adapter `AttributeDefinition`-Einträge und
  `ProductAttributeMapping`-Einträge (ADR-0018) in die EAV-Schicht. Kein Kerncode
  von KoalixCRM hängt von einem Layer-3-Adapter ab.

### Adapter-Rolle der Klassifizierungsstandards

Klassifizierungsstandards sind Adapter, die Attributdefinitionen und Mappings in Layer 2
einspeisen. Geschäftslogik liest ausschließlich kanonische Schlüssel aus dem
KoalixCRM-eigenen Vokabular (ADR-0018). Kein Produktionscode referenziert
Standard-spezifische Attribut-Identifier.

### Lizenzbeschränkung (erweitert)

Das Open-Source-Backend (`/app/koalixcrm`) bündelt keinen lizenzpflichtigen Fremdinhalt:

- **UNSPSC** — frei nutzbar und weiterverteilbar. Kann als Fixture und als optionaler
  Layer-3-Adapter-Bundle ausgeliefert werden.
- **GPC (GS1 Global Product Classification)** — frei nutzbar. Kann als Fixture und als
  optionaler Layer-3-Adapter-Bundle ausgeliefert werden.
- **eCl@ss** — unterliegt einer kommerziellen Mitgliedslizenz (eCl@ss e.V., Single-Lizenz
  oder Concordance-Lizenz; optionaler Webservice mit IRDI-basierter Abrechnung). eCl@ss-
  Code-Listen und Attributdefinitionen dürfen **nicht** als Fixtures oder Bundles im
  PyPI-Paket enthalten sein. Das Datenbankschema unterstützt eCl@ss technisch über
  `Classification` + `ClassificationNode` + `ProductAttributeMapping` (ADR-0018); der
  Inhalt wird vom Betreiber unter dessen eigener Mitgliedschaft importiert.
- **ETIM** — ebenfalls lizenzpflichtig. Gleiche Regel wie eCl@ss: Schema-Unterstützung ja,
  Inhalt nicht gebündelt, Betreiber-Import unter eigener Lizenz.

OQ-0001 ist durch ADR-0018 geschlossen (2026-05-05).
