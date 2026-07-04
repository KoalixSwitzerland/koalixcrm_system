# ADR-0019: Produkt-`kind`-Invarianten und Gating abhängiger Objekte

## Status
Proposed

## Context

ADR-0003 definiert das `kind`-Enum (`SERVICE`, `TRADING_GOOD`, `MANUFACTURED_GOOD`, `KIT`,
`RAW_MATERIAL`) als maschinenlesbaren Ankerpunkt für kind-spezifische Erweiterungen. Die
nachfolgenden ADRs (ADR-0006, ADR-0007, ADR-0009, ADR-0014) bauen je einzeln auf diesem Enum auf
und formulieren kind-gekoppelte Einschränkungen — jedoch jedes ADR isoliert und in unterschiedlicher
Detailtiefe. ADR-0006 beschränkt `BillOfMaterials` auf `MANUFACTURED_GOOD`; ADR-0007 beschränkt
`ServiceProfile` auf `SERVICE`; ADR-0009 führt `tracking_mode` ein, ohne den Geltungsbereich für
`SERVICE` explizit zu schließen; ADR-0014 erweitert die BOM-Nutzung implizit auf `KIT`, indem
`ProductionOrder` einen FK auf `BillOfMaterials` trägt und für `MANUFACTURED_GOOD` und `KIT`
gilt. Die wiederkehrende Formulierung „die Applikationsschicht muss diese Invariante durchsetzen"
in ADR-0006, ADR-0007 und weiteren ADRs zeigt, dass eine autoritative, vollständige
Regelsammlung fehlt. OQ-0009 benennt die ungelöste Frage, welche Zustände einen kind-Wechsel
sperren und ob `ProductVariant` dazu gehört.

## Decision

Die Applikationsschicht setzt alle `kind`-gekoppelten Gating-Regeln für abhängige Objekte und
die Unveränderlichkeit des `kind`-Felds über eine einzige autoritative Policy-Komponente durch:
`ProductKindPolicy`. Diese Entscheidung konsolidiert die verstreuten „Applikationsschicht
muss…"-Vorbehalte aus ADR-0006, ADR-0007 und ADR-0009 in einem einzigen Regelwerk.

## Why

Eine einzelne `ProductKindPolicy`-Komponente — anstatt pro-Modell verstreuter Ad-hoc-Checks in
Serializern und Services — stellt sicher, dass dieselbe Gating-Matrix an jedem Einstiegspunkt
(DRF-Serializer, Service-Layer, Admin-Aktion) konsistent angewendet wird. Der Widerspruch
zwischen ADR-0006 (BOM nur für `MANUFACTURED_GOOD`) und ADR-0014 (ProductionOrder und
BOM-Explosion für `MANUFACTURED_GOOD` und `KIT`) wird durch dieses ADR kanonisch aufgelöst,
bevor Implementierer auf inkonsistente Quellen treffen.

## Alternatives Considered

- **Datenbankebene: CHECK-Constraints oder Trigger für kind-Gating** — abgelehnt: die
  Gating-Regeln betreffen Beziehungen zwischen mehreren Tabellen und berücksichtigen zeitliche
  Zustände (z. B. ob bereits eine `StockMovement`-Zeile existiert); ein einzelner
  CHECK-Constraint auf einer Tabellenzeile kann diese tabellenübergreifenden und
  zustandsabhängigen Prüfungen strukturell nicht ausdrücken. ADR-0006 hat diesen Weg bereits
  explizit abgelehnt.
- **Pro-Modell verteilte Validatoren (Status quo)** — abgelehnt: diese Struktur ist die Ursache
  des ADR-0006/ADR-0014-Widerspruchs und der offenen OQ-0009-Frage; einzelne Modell-Validatoren
  driften, sobald ein nachgelagertes ADR die kind-Bedingung eines anderen Modells ergänzt.
- **Product-Unterklassen pro kind** — abgelehnt: bereits in ADR-0003 abgelehnt, da
  branchenspezifische Unterklassen Schema-Änderungen und Code-Deploys je Mandant erfordern und
  das Backend nicht als einheitliches PyPI-Wheel auslieferbar halten.

## Consequences

### Positive
- Eine einzige `ProductKindPolicy`-Komponente ist die kanonische Referenz für alle Gating-Fragen;
  Inkonsistenzen zwischen ADRs werden zentral statt pro-Modell aufgelöst.
- Die vollständige Gating-Matrix ist als eine Tabelle lesbar; Implementierer benötigen kein
  Querlesen mehrerer ADRs, um alle kind-Einschränkungen zu kennen.
- Die Unveränderlichkeits-Regel mit explizitem Sperr-Set schließt OQ-0009 und gibt den
  Anforderungen REQ-0001 AC-4, REQ-0015 AC-1 und REQ-0016 AC-2 eine autoritative Grundlage.
- kind-agnostische Objekte (`ProductVariant`, `ProductSupply`, `ProductPassport` usw.) sind
  explizit benannt; Implementierer können sie ohne kind-Prüfung anlegen.

### Negative
- `ProductKindPolicy` ist ein neues, querschneidend genutztes Service-Objekt; Änderungen an der
  Gating-Matrix müssen in diesem einen Objekt eingepflegt werden und erfordern kein
  Streuen von Fixes auf viele Modelle.
- Die `BillOfMaterials`-Ausweitung auf `KIT` (Abschnitt „Konflikt mit ADR-0006 / REQ-0015")
  erfordert eine Korrektur von REQ-0015 AC-1 durch den `kxcrm-requirements-engineer`.

---

## Gating-Matrix

Die folgende Tabelle ist die autoritative Regelquelle für alle kind-gekoppelten abhängigen Objekte.
„Erlaubt" bedeutet, dass die Applikationsschicht das Anlegen dieses Objekts zulässt. „Verboten"
bedeutet, dass `ProductKindPolicy` den Anlege-Versuch mit einem validierten Fehler ablehnt.
„Nicht ausgewertet" bedeutet, dass das Feld technisch existiert, aber von der Applikationsschicht
für diesen kind-Wert ignoriert wird.

| Abhängiges Objekt / Feld                                    | SERVICE   | TRADING_GOOD | MANUFACTURED_GOOD | KIT       | RAW_MATERIAL |
|-------------------------------------------------------------|-----------|--------------|-------------------|-----------|--------------|
| `ServiceProfile` (1:1, ADR-0007)                            | Erlaubt   | Verboten     | Verboten          | Verboten  | Verboten     |
| `BillOfMaterials` (1:1, ADR-0006)                           | Verboten  | Verboten     | Erlaubt           | Erlaubt   | Verboten     |
| `ProductionOrder` (ADR-0014)                                | Verboten  | Verboten     | Erlaubt           | Erlaubt   | Verboten     |
| `kit_mode` (additives Feld auf `Product`, ADR-0014)         | Nicht ausgewertet | Nicht ausgewertet | Nicht ausgewertet | Erlaubt | Nicht ausgewertet |
| `tracking_mode` (additives Feld auf `Product`, ADR-0009)    | Nur `NONE` | Erlaubt     | Erlaubt           | Erlaubt   | Erlaubt      |
| `OnHandRecord` / `StockMovement` / `SerialUnit` / `Batch`   | Verboten  | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| `ProductVariant` (ADR-0003)                                 | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| `ProductFamily` (ADR-0003)                                  | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| `ProductTranslation` (ADR-0003)                             | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| `ProductMedia` (ADR-0003)                                   | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| Klassifizierung / Attribute (ADR-0004)                      | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| Preislogik / `ProductPrice` (ADR-0005)                      | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| `ProductSupply` (ADR-0006)                                  | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |
| `ProductPassport` (ADR-0008)                                | Erlaubt   | Erlaubt      | Erlaubt           | Erlaubt   | Erlaubt      |

**Erläuterungen:**

- `tracking_mode = NONE` ist für `SERVICE` Pflicht. `ProductKindPolicy` weist jeden Versuch ab,
  `tracking_mode` auf `BATCH` oder `SERIAL` zu setzen, wenn `kind = SERVICE`.
- Lagerentitäten (`OnHandRecord`, `StockMovement`, `SerialUnit`, `Batch`, `StockBalance`,
  `StockReservation`) sind für `SERVICE` vollständig verboten, da Dienstleistungen keinen
  physischen Bestandsort haben.
- `kit_mode` ist ein additives Feld auf `Product` (ADR-0014); die Applikationsschicht wertet es
  ausschließlich für `kind = KIT` aus. Bei allen anderen kinds gilt der Feldwert als irrelevant
  und wird nicht zur Steuerung von Kommissionier- oder Fertigungslogik herangezogen.
- `ProductSupply` ist nicht kind-gated: Auch eine Dienstleistung darf einen Lieferanten-
  bzw. Subunternehmer-Link tragen.
- Kind-agnostische Objekte (`ProductVariant`, `ProductFamily`, `ProductTranslation`,
  `ProductMedia`, Klassifizierung, Preislogik, `ProductSupply`, `ProductPassport`) darf die
  Applikationsschicht ohne kind-Prüfung anlegen; `ProductKindPolicy` prüft sie nicht.

---

## kind-Unveränderlichkeit

`kind` ist frei änderbar, solange kein Element des folgenden **Sperr-Sets** existiert. Sobald
eines dieser Elemente angelegt wurde, setzt `ProductKindPolicy` das `kind`-Feld auf unveränderlich:

- ein `ServiceProfile`-Eintrag für dieses Produkt,
- ein `BillOfMaterials`-Eintrag für dieses Produkt,
- ein `ProductionOrder`-Eintrag, der dieses Produkt referenziert,
- mindestens eine Lagertatsache: `StockMovement`-, `SerialUnit`-, `Batch`- oder
  `OnHandRecord`-Zeile mit FK auf dieses Produkt,
- ein gesetzter (nicht-Defaultwert) `kit_mode` auf diesem Produkt,
- ein Attributwert in einem kind-gebundenen `AttributeSet` (ADR-0004) für dieses Produkt
  — d. h. ein `AttributeSet`, das `kind` als Achse verwendet und dessen Attribute ausschließlich
  für den aktuellen kind-Wert gelten.

**Explizit nicht im Sperr-Set:** Das bloße Vorhandensein von `ProductVariant`-Einträgen sperrt
`kind` nicht. `ProductVariant` ist kind-agnostisch (siehe Gating-Matrix); seine Existenz ändert
die Gültigkeitsbedingungen keines kind-gekoppelten Objekts.

Solange `kind` noch änderbar ist, führt eine kind-Änderung zur Neuauflösung kind-gebundener
`AttributeSet`-Zuordnungen (ADR-0004): Pflichtattribute, die für den bisherigen kind-Wert
galten, können entfallen; für den neuen kind-Wert können neue Pflichtattribute hinzukommen.
`ProductKindPolicy` signalisiert diese Änderung der betroffenen Schicht, damit fehlende
Pflichtattribute dem Benutzer angezeigt werden.

---

## Durchsetzungsschicht

`ProductKindPolicy` ist eine einzelne, autoritative Komponente in der Django-Applikationsschicht.
DRF-Serializer und Service-Layer rufen `ProductKindPolicy` gemeinsam auf; kein Modell-, Serializer-
oder View-Code implementiert kind-Gating eigenständig. Datenbankebene (CHECK-Constraints,
Trigger) setzt keine kind-Regeln durch, da diese Regeln tabellenübergreifend und zustandsabhängig
sind (s. „Alternatives Considered").

`ProductKindPolicy` deckt genau drei Aufgaben ab:

1. **Gating beim Anlegen:** Prüft vor dem Anlegen eines abhängigen Objekts, ob der kind-Wert
   des referenzierten Produkts das Objekt erlaubt (s. Gating-Matrix).
2. **Unveränderlichkeit beim kind-Wechsel:** Prüft vor dem Speichern eines geänderten
   `Product.kind`-Werts, ob das Sperr-Set leer ist.
3. **AttributeSet-Neuauflösung:** Benachrichtigt die Attributschicht (ADR-0004) über einen
   kind-Wechsel, damit Pflichtattribute neu berechnet werden.

---

## Konflikt mit ADR-0006 / REQ-0015

ADR-0006 formuliert `BillOfMaterials` ausschließlich für `kind = MANUFACTURED_GOOD`. ADR-0014
(jünger) definiert `ProductionOrder` mit FK auf `BillOfMaterials` für `kind = MANUFACTURED_GOOD`
**und** `kind = KIT` — und setzt damit eine `BillOfMaterials` für KIT-Produkte voraus, da ohne
BOM keine `ProductionOrder` und keine `BillOfMaterialsExplosion` existieren kann.

Dieses ADR behandelt ADR-0014 als authoritative Quelle und weitet den erlaubten Geltungsbereich
von `BillOfMaterials` auf `MANUFACTURED_GOOD` und `KIT` aus. Die ADR-0006-Formulierung „gilt
ausschließlich für `Product` mit `kind = MANUFACTURED_GOOD`" ist durch diese Entscheidung
ergänzt; ADR-0006 wird nicht rückwirkend geändert.

REQ-0015 AC-1 enthält eine analoge kind-Einschränkung auf `MANUFACTURED_GOOD`, die nach dieser
Entscheidung nicht mehr vollständig ist. Der `kxcrm-requirements-engineer` korrigiert REQ-0015
AC-1 so, dass die Bedingung `kind ∈ {MANUFACTURED_GOOD, KIT}` lautet. Dieses ADR ändert
REQ-0015 nicht selbst.

---

## Workspace-Scoping-Matrix

`ProductKindPolicy` ist eine zustandslose Applikationsschicht-Komponente ohne eigenes
Datenbankschema. Die von ihr verwalteten Objekte tragen alle das `WorkspaceScopedModel`-Erbe
aus ADR-0001. Es entsteht kein neues, workspace-gescoptes Datenbankmodell.

---

## Lizenzbeschränkung

`ProductKindPolicy` und die in diesem ADR definierten Gating-Regeln leben vollständig im
Open-Source-Backend (`/app/koalixcrm`), das als PyPI-Wheel und Docker-Image ausgeliefert wird.
Das ADR enthält keinen Quantalq-proprietären Inhalt. Das REST-API-Integrationsprotokoll zwischen
dem Open-Source-Backend und dem geschlossenen Next.js-Frontend (ADR-0002) bleibt die einzige
Kommunikationsbrücke; kein Domänen-Code darf direkt in den Frontend-Build importiert werden.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0003 (Produkt-Katalog-Backbone):** Definiert das `kind`-Enum und die Backbone-Entitäten
(`Product`, `ProductVariant`, `ProductFamily`, `ProductTranslation`, `ProductMedia`). Dieses ADR
regelt, welche kind-Werte welche Erweiterungsobjekte erlauben; ADR-0003 definiert das Enum selbst.

**ADR-0004 (Klassifizierung und erweiterbare Attribute):** `AttributeSet` kann `kind` als Achse
verwenden. Ein kind-gebundener Attributwert ist Teil des Sperr-Sets dieses ADR. Ein kind-Wechsel
löst die AttributeSet-Neuauflösung gemäß ADR-0004 aus.

**ADR-0006 (Beschaffung und Stücklisten):** Definiert `BillOfMaterials` und `ProductSupply`.
Dieses ADR weitet den erlaubten Geltungsbereich von `BillOfMaterials` auf `KIT` aus (Konflikt
mit ADR-0006 dokumentiert im Abschnitt „Konflikt mit ADR-0006 / REQ-0015"). `ProductSupply` ist
kind-agnostisch gemäß diesem ADR.

**ADR-0007 (Dienstleistungsprofil):** Definiert `ServiceProfile` als 1:1-Erweiterung für
`kind = SERVICE`. Dieses ADR übernimmt diese Einschränkung als Zeile der Gating-Matrix und
erhebt sie zur `ProductKindPolicy`-Regel.

**ADR-0009 (Lager-Domänen-Backbone):** Definiert `tracking_mode` als additives Feld auf
`Product`. Dieses ADR legt fest, dass `tracking_mode` für `kind = SERVICE` ausschließlich `NONE`
sein darf und dass Lagerentitäten für `SERVICE` vollständig verboten sind.

**ADR-0012 (Lebenszeit, Charge, Los und Seriennummer):** `SerialUnit`- und `Batch`-Einträge
sind Lagertatsachen im Sperr-Set dieses ADR; ihr Vorhandensein sperrt den kind-Wechsel.

**ADR-0014 (Montage/Kitting und geteilter Bestand):** Definiert `ProductionOrder`,
`ProductionOrderComponent` und `kit_mode`. Dieses ADR übernimmt die kind-Einschränkung
(`MANUFACTURED_GOOD` und `KIT`) als Gating-Matrix-Zeilen und behandelt ADR-0014 als
autoritative Quelle für den erweiterten BOM-Geltungsbereich.

## Changelog
- 2026-06-27: Erstentwurf (Status: Proposed). Konsolidiert die verstreuten „Applikationsschicht muss…"-Vorbehalte
  aus ADR-0006, ADR-0007, ADR-0009 und ADR-0014 in einer kanonischen Gating-Matrix und
  einer `ProductKindPolicy`-Durchsetzungsregel. Schließt OQ-0009.
