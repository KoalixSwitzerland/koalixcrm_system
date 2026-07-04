# Glossar — KoalixCRM

Dieses Dokument definiert die im System verwendeten Domänenbegriffe verbindlich.
Nachgelagerte Anforderungen, Use Cases und ADRs verwenden die hier definierten Begriffe.

---

## Produktdomäne

**AttributeDefinition**  
Metadaten-Eintrag, der ein einzelnes Attribut beschreibt: Datentyp, Validierungsregeln, Flags
(`is_required`, `is_localized`, `is_multivalued`) und Scope (`GLOBAL`, `WORKSPACE`, `INHERITED`).
Geregelt durch ADR-0004.

**AttributeGroup**  
Bündel thematisch zusammengehöriger `AttributeDefinition`-Einträge (z. B. „Nährwerte je 100 g").
Geregelt durch ADR-0004.

**AttributeSet**  
Workspace-scoped Verknüpfung zwischen einer Kombination aus `ClassificationNode` und/oder
`kind`-Enum und einer Menge von `AttributeGroup`-Einträgen. Bestimmt, welche Attribute ein
Produkt erbt. Geregelt durch ADR-0004.

**BillOfMaterials (BOM)**  
Stückliste nach ISA-95 Part 2 für ein `Product` mit `kind = MANUFACTURED_GOOD`. Enthält eine
geordnete Liste von `BomItem`-Einträgen. Geregelt durch ADR-0006.

**BomItem**  
Einzelne Zeile einer `BillOfMaterials`: Komponenten-Produkt, Menge, Einheit, Ausschuss-Prozentsatz
und optionale Alternativkomponente. Geregelt durch ADR-0006.

**Classification**  
Globales Klassifizierungsschema (z. B. UNSPSC, eCl@ss, intern). Mehrere Schemata koexistieren.
Geregelt durch ADR-0004.

**ClassificationNode**  
Einzelner Knoten im hierarchischen Baum eines `Classification`-Schemas. Geregelt durch ADR-0004.

**customer_group_transform**  
Bestehendes Modell für Umrechnungsfaktoren nach Parteigruppe, Einheit und Währung. Referenziert
`PartyGroup` (ADR-0001). Geregelt durch ADR-0005.

**kind**  
Enum-Feld auf `Product` mit den Werten `SERVICE`, `TRADING_GOOD`, `MANUFACTURED_GOOD`, `KIT`,
`RAW_MATERIAL`. Bestimmt, welche nachgelagerten Entitäten (`BillOfMaterials`, `ServiceProfile`)
an ein Produkt gebunden werden können. Geregelt durch ADR-0003.

**lifecycle_status**  
Status-Enum auf `Product` mit mindestens den Werten `DRAFT`, `ACTIVE`, `DISCONTINUED`, `ARCHIVED`.
Steuert, ob ein Produkt in neuen kommerziellen Dokumenten verwendet werden darf. Geregelt durch
ADR-0003, REQ-0005.

**Party**  
Gemeinsamer Supertyp für `Organization` und `Contact` gemäß UBL-2.3-Partei-Muster. Geregelt durch
ADR-0001.

**PartyGroup**  
Workspace-scoped Gruppierung von `Party`-Einträgen (ehemals `CustomerGroup`). Geregelt durch
ADR-0001. Wird von `customer_group_transform` (ADR-0005) referenziert.

**PriceList**  
Workspace-scoped Gruppierung von `ProductPrice`-Einträgen nach Kanal oder Kundensegment.
Geregelt durch ADR-0005.

**Product**  
Kanonisches Stammdatenobjekt des Produktkatalogs (ehemals `ProductType`). Trägt `sku`, `gtin`,
`mpn`, `kind`, `lifecycle_status`, `base_uom`, `tax_class`, `brand`. Geregelt durch ADR-0003.

**ProductAttributeDecimal / String / Enum / Bool / Reference / Int**  
Die sechs typisierten Wertetabellen des EAV-Systems. Jede trägt einen zusammengesetzten Index auf
`(product_id, attribute_definition_id)`. Geregelt durch ADR-0004.

**ProductClassification**  
Workspace-scoped M:N-Verknüpfung zwischen `Product` und `ClassificationNode`. Geregelt durch
ADR-0004.

**ProductFamily**  
Workspace-scoped Gruppierung aller Varianten einer Produktlinie. Geregelt durch ADR-0003.

**ProductMedia**  
Workspace-scoped Verweis auf ein Medienobjekt (Bild, Datenblatt, Zertifikat) im S3/MinIO-Speicher.
Liefert Presigned-URLs über die API. Geregelt durch ADR-0003.

**ProductPassport**  
Workspace-scoped JSONB-Vorhalter für EU-Digital-Product-Passport-Daten (1:1 mit `Product`).
Struktur wird durch ein Folge-ADR festgelegt. Geregelt durch ADR-0008.

**ProductPrice**  
Workspace-scoped zeitgebundener Preis mit FK auf `Product`. Bestehendes Modell; unverändert
gemäß ADR-0005.

**ProductSupply**  
Workspace-scoped Verknüpfung eines `Product` mit einem Lieferanten-`Party`
(`PartyRole.role_type = 'supplier'`). Trägt Lieferanten-SKU, Lieferzeit, MOQ, Einkaufspreis.
Geregelt durch ADR-0006.

**ProductTranslation**  
Workspace-scoped mehrsprachige Bezeichnung, Kurzbeschreibung und Langbeschreibung eines `Product`,
je Sprachcode (BCP-47). Geregelt durch ADR-0003.

**ProductVariant**  
Workspace-scoped Einzelzeile pro handelbare SKU mit FK auf `ProductFamily` und Varianten-Achsen-Werten.
Geregelt durch ADR-0003.

**ServiceProfile**  
Workspace-scoped 1:1-Erweiterung von `Product` für `kind = SERVICE`. Trägt `billing_model`
(`fixed | hourly | subscription | tiered`), `default_duration`, `deliverable` und `sla_reference`.
Geregelt durch ADR-0007.

**SKU (Stock Keeping Unit)**  
Eindeutiger Artikelidentifikator eines `Product` oder einer `ProductVariant` innerhalb eines
Workspace.

**UnitOfMeasureConversion**  
Workspace-scoped produktspezifischer Einheitenumrechnungsfaktor (z. B. Stück ↔ Karton à 12).
Geregelt durch ADR-0005.

**WorkspaceScopedModel**  
Django-Basisklasse aus ADR-0001, die sicherstellt, dass tenant-eigene Daten automatisch nach
`request.active_workspace` gefiltert werden. Alle produktdomänen-internen Entitäten erben diesen
Mechanismus.
