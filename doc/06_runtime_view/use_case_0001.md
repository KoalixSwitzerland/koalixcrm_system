# UC-0001: Produkt anlegen und klassifizieren

**ID:** UC-0001  
**Bezug:** [ADR-0003](../09_architecture_decisions/0003-product-catalog-backbone.md), [ADR-0004](../09_architecture_decisions/0004-classification-and-extensible-attributes.md), [REQ-0001](../01_introduction_and_goals/requirements/REQ-0001.md), [REQ-0005](../01_introduction_and_goals/requirements/REQ-0005.md), [REQ-0008](../01_introduction_and_goals/requirements/REQ-0008.md)  
**Lizenzseite:** Open-Source-Backend (Datenmodell und API); Closed-Source-Frontend (UI-Interaktion)

---

## Akteure
- **Primär:** Produktmanager (eingeloggter Benutzer mit Schreibrecht auf den aktiven Workspace)
- **System:** KoalixCRM-Backend (DRF), KoalixCRM-Frontend (Next.js/Refine)

## Vorbedingungen
- Der Produktmanager ist authentifiziert und hat einen aktiven Workspace.
- Mindestens eine `Classification` (z. B. UNSPSC) und der zugehörige `ClassificationNode`-Baum sind im System geladen.
- Die Basismaßeinheit (`base_uom`) und die Steuerklasse (`tax_class`) für den Workspace sind konfiguriert.

## Auslöser
Der Produktmanager navigiert zur Produktlistenseite und klickt auf „Neues Produkt anlegen".

## Hauptablauf

### Hauptablauf (Übersicht)
Der folgende Ablauf fasst den Standardfall aus Sicht des Produktmanagers zusammen#59; Details zu Schnittstellen zeigt das Sequenzdiagramm darunter.

```mermaid
flowchart TD
    A["Neues Produkt anlegen"] --> B["Produktformular ausfüllen<br/>(SKU, kind, lifecycle_status,<br/>base_uom, tax_class, brand)"]
    B --> C["ClassificationNode auswählen"]
    C --> D["Produkt speichern"]
    D --> E{"SKU bereits vorhanden#63;"}
    E -- "Nein" --> F["Produktdetailseite anzeigen"]
    E -- "Ja" --> B
```

```mermaid
sequenceDiagram
    actor PM as "Produktmanager"
    participant FE as "Frontend<br/>(Next.js)"
    participant BE as "Backend<br/>(DRF)"
    participant DB as "Datenbank"

    PM->>FE: Klick auf „Neues Produkt anlegen"
    FE->>BE: GET /api/classifications/ (verfügbare Schemata)
    BE->>DB: SELECT Classification, ClassificationNode<br/>(workspace-gefiltert + global)
    DB-->>BE: Klassifizierungsbaum
    BE-->>FE: 200 OK — Klassifizierungsdaten

    FE->>PM: Produktformular anzeigen
    PM->>FE: SKU, kind, lifecycle_status, base_uom,<br/>tax_class, brand eingeben
    PM->>FE: ClassificationNode auswählen

    FE->>BE: POST /api/products/ {sku, kind, lifecycle_status, …}
    BE->>DB: INSERT Product (workspace-scoped)
    DB-->>BE: Product-ID
    BE->>DB: INSERT ProductClassification<br/>(product_id, classification_node_id)
    DB-->>BE: OK
    BE-->>FE: 201 Created — Product-Objekt

    FE->>PM: Produktdetailseite anzeigen
```

## Alternativablauf A: Doppelte SKU
- Im Schritt „POST /api/products/" erkennt das Backend, dass die Kombination `(workspace, sku)` bereits existiert.
- Das Backend antwortet mit HTTP 400 und einer Fehlermeldung, die die doppelte SKU benennt.
- Das Frontend zeigt die Fehlermeldung im Formular an; der Produktmanager korrigiert die SKU.

## Alternativablauf B: Kein ClassificationNode ausgewählt
- Das Frontend erlaubt das Absenden des Formulars ohne Klassifizierung.
- Das `ProductClassification`-Eintrag entfällt; das `Product`-Objekt wird ohne Klassifizierung angelegt.
- Der Produktmanager kann die Klassifizierung später in der Produktdetailansicht nachtragen.

## Nachbedingungen
- Genau ein `Product`-Objekt mit dem gewählten `kind` und `lifecycle_status = DRAFT` ist im Workspace angelegt.
- Optional: Genau ein `ProductClassification`-Eintrag verknüpft das neue Produkt mit dem gewählten `ClassificationNode`.

## Behavioral Acceptance Criteria

### BAC-1: Formular-Verhalten
- [ ] Das Produktformular zeigt einen Pflichtfeld-Hinweis, wenn `kind`, `sku`, `base_uom` oder `tax_class` leer sind, bevor der Produktmanager absenden kann.
- [ ] Das Produktformular zeigt die `kind`-Enum-Werte als lesbare Auswahlliste (`SERVICE`, `TRADING_GOOD`, `MANUFACTURED_GOOD`, `KIT`, `RAW_MATERIAL`).
- [ ] Nach erfolgreichem Anlegen navigiert das Frontend automatisch zur Produktdetailseite des neuen Produkts.

### BAC-2: Fehlerdarstellung
- [ ] Eine HTTP-400-Antwort wegen doppelter SKU erzeugt eine inline Fehlermeldung am SKU-Feld; kein generisches Toast.
- [ ] Das Formular behält die eingegebenen Werte nach einem Fehler.

### BAC-3: Klassifizierungsbaum
- [ ] Der Klassifizierungsbaum ist durchsuchbar; der Produktmanager gibt einen Suchbegriff ein und sieht passende Knoten.

---

## Referenzen
- [ADR-0003](../09_architecture_decisions/0003-product-catalog-backbone.md) — Produktkatalog-Fundament: `Product`-Basisfelder und `kind`-Enum
- [ADR-0004](../09_architecture_decisions/0004-classification-and-extensible-attributes.md) — `Classification`/`ClassificationNode`-Baum
- [ADR-0019](../09_architecture_decisions/0019-product-kind-invariants.md) — autoritative `kind`-Gating-Tabelle (`ProductKindPolicy`)
- [REQ-0001](../01_introduction_and_goals/requirements/REQ-0001.md), [REQ-0005](../01_introduction_and_goals/requirements/REQ-0005.md), [REQ-0008](../01_introduction_and_goals/requirements/REQ-0008.md) — governing requirements
- [Glossar](../12_glossary/glossar.md) — Begriffsdefinitionen (`kind`, `lifecycle_status`, `Classification`, `ClassificationNode`)
