# Architektur-Übersicht KoalixCRM

KoalixCRM ist eine mandantenfähige, webbasierte CRM- und ERP-Plattform. Das Open-Source-Backend
(`/app/koalixcrm`) ist als Django-Applikation implementiert, über PyPI und Docker vertrieben und
stellt ausschließlich eine JSON-REST-API bereit. Das geschlossene Frontend
(`/app/koalixcrm-frontend`) ist eine Next.js-Applikation (Refine.dev), die ausschließlich als
Docker-Image für den internen Einsatz bei Quantalq ausgeliefert wird. Die einzige
Kommunikationsbrücke zwischen den beiden Seiten ist die öffentliche REST-API des Backends.

Weiterführende Diagramme:
- Kontextdiagramm: `context_diagram.md`
- Systemdiagramm: `system_diagram.md`
- Produktrisiken: `product_risks.md`

---

## ADR-Verzeichnis

| ID       | Titel                                                                         | Status   | Zusammenfassung                                                                                                                                                                              |
|----------|-------------------------------------------------------------------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ADR-0001 | Kontakt- und Partei-Datenmodell                                               | Accepted | Ersetzt das bisherige `Contact`/`Customer`/`Supplier`-MTI-Modell durch das UBL-2.3-ausgerichtete Party-Muster mit losen Rollenzuweisungen und entkoppelten Kontaktmechanismen.               |
| ADR-0002 | Ersatz des Django-Admins durch ein Next.js-basiertes Admin-UI                | Accepted | Refine.dev auf Next.js App Router; Django schrumpft auf reines DRF-JSON-Backend; shadcn/ui als UI-Kit; django-filebrowser wird durch django-storages + S3-Presigned-URLs ersetzt.            |
| ADR-0003 | Produkt-Katalog-Backbone                                                      | Accepted | Benennt `ProductType` in `Product` um, entfernt das leere `Product`-Hüllenmodell, führt `kind`-Enum, `ProductFamily`, `ProductVariant`, `ProductTranslation` und `ProductMedia` ein.        |
| ADR-0004 | Klassifizierung und erweiterbare Attribute (getypte EAV-Tabellen)            | Accepted | Dreistufiges Modell aus globalen Taxonomiebäumen, konfigurierbaren Attribut-Metadaten und je einer typisierten Wertetabelle pro Datentyp; kein polymorphes EAV; JSONB-Spiegel für Lesepfade. |
| ADR-0005 | Preisgestaltung und Maßeinheiten                                              | Accepted | Bestehende `ProductPrice`/`customer_group_transform`-Logik bleibt erhalten; ergänzt durch `UnitOfMeasureConversion` (produktspezifisch) und `PriceList` (Kanal-/Kundensegment-Gruppierung). |
| ADR-0006 | Beschaffung und Stücklisten                                                   | Accepted | `ProductSupply` verknüpft `Product` mit Lieferanten-`Party` (ADR-0001); `BillOfMaterials`/`BomItem` nach ISA-95 Part 2 für `kind = MANUFACTURED_GOOD`; kein separates Lieferantenmodell.   |
| ADR-0007 | Dienstleistungsprofil                                                         | Accepted | `ServiceProfile` 1:1 mit `Product` (`kind = SERVICE`); trägt Abrechnungsmodell, Standarddauer, Leistungsbeschreibung und SLA-Referenz; keine separate Dienstleistungsmodellhierarchie.      |
| ADR-0008 | Digital Product Passport — JSONB-Vorhalter                                   | Accepted | `ProductPassport` JSONB-Block pro `Product` als Vorhalter für EU-DPP-Anforderungen; Struktur wird durch Folge-ADR festgelegt, wenn delegierte EU-Rechtsakte je Produktkategorie abgeschlossen sind. |
| ADR-0009 | Lager-Domänen-Backbone                                                        | Accepted | `Location` (n-stufige Hierarchie, `location_type`-Enum jetzt mit `LAYER` zwischen `SHELF` und `BIN` für 4-stufige Regal-Fach-Ebene-Position-Hierarchie), `OnHandRecord`, `HandlingUnit` (GS1-SSCC), `Product.tracking_mode`-Enum; alle workspace-scoped. |
| ADR-0010 | Lagerbestandszustände und Reservierungen                                      | Accepted | Sechs virtuelle Mengensegmente auf `StockBalance`; `StockReservation` mit `kind`-Feld (`SALE`/`RENTAL`/`PROJECT_HOLD`), `reservation_status` (`PROVISIONAL`/`CONFIRMED`), Zeitfensterfeldern und `serial_unit`-FK; zeitfensterbasierte ATP-Funktionen `is_free()` und `free_windows()`; Angebots-Lebenszyklus-Kopplung mit first-SENT-wins-Konkurrenzregel. |
| ADR-0011 | Lager- und Lebenszyklus-Ereignis-Log                                          | Accepted | Unveränderlicher `StockMovement`-Log, EPCIS-2.0-orientiert; Lager-, Lifecycle- und Inventur-Businesssteps (inkl. `inventorying` für Ad-hoc-Zykluszählungen ohne Saldo-Mutation; Diskrepanz erzeugt separaten `adjustment`-Event); nullable `qty`; `disposition`-Feld (EPCIS CBV); synchrone `StockBalance`-Aktualisierung; konfigurierbare Aufbewahrungsuntergrenze. |
| ADR-0012 | Lebenszeit, Charge, Los und Seriennummernverfolgung                           | Accepted | `Batch` (Ablaufdatum, MHD, Produktionsdatum, Lieferantencharge) und `SerialUnit` (ISO/IEC-15459-UID, Betriebszustand); FEFO-Pickreihenfolge; Soft-Delete-Forever; konfigurierbare Aufbewahrungsuntergrenze. |
| ADR-0013 | Miet- und Kundengeführter Bestand                                             | Accepted | `OnHandRecord.owner_type` mit Werten `OWN`, `RENTAL`, `CUSTOMER_CONSIGNMENT`, `CUSTOMER_OWNED`; `RentalAssignment` als Spezialisierung von `StockReservation` (physische Übergabephase); `StockReservation` ist alleinige Belegungsquelle für den Verfügbarkeitskalender; `RentalAssignment` trägt Pflicht-FK auf erfüllte `StockReservation`. |
| ADR-0014 | Montage/Kitting und geteilter Bestand                                         | Accepted | `ProductionOrder` + `ProductionOrderComponent`; `BillOfMaterialsExplosion`-Snapshot (Celery, Tiefengrenzen 10/20); Fertigstellung als `TRANSFORMATION_EVENT` + `AGGREGATION_EVENT` für As-Built-BOM. |
| ADR-0015 | Geräte-Lebenszyklus-Historie                                                  | Accepted | Per-`SerialUnit`-Zeitleiste im `StockMovement`-Log als System of Record; As-Built-BOM via `AGGREGATION_EVENT`; Komponentengraph-Traversierung; vierter Abfragepfad `who_held_it_when` (Halter-Zeitleiste über `disposition`); fünfter Abfragepfad `where_was_it_when` (Standort-Zeitleiste über `destination_location`-Wechsel, UC-0008/0009/0010); AAS-Readiness als Gestaltungslinse. |
| ADR-0016 | Identifier-Registry und Barcode-Auflösung                                     | Accepted | Kanonischer Endpunkt `POST /api/v1/scan/resolve`; zweistufige Auflösung: GS1 AI-Parsing zuerst (GTIN→Product, SSCC→HandlingUnit, SGTIN/GIAI→SerialUnit, GLN→Location), danach Freitext-Abgleich; GS1 schlägt Freitext; Freitext-Multi-Hit → HTTP 409; workspace-scoped außer `Product.gtin` (katalogweit). |
| ADR-0017 | GoodsReceipt als Prozess-Aggregat                                             | Accepted | `GoodsReceipt`-Aggregat mit `status ∈ {DRAFT, IN_PROGRESS, COMPLETED, CANCELLED}` und `GoodsReceiptLine` (`line_status ∈ {PENDING, CONFIRMED, MISMATCHED}`); COMPLETED erzeugt synchrone `StockMovement`-Buchungen je Position (`business_step=receiving`); Ingestion via strukturiertem JSON-Payload; OCR/EDI als externe Adapter; Put-Away-Strategie offen (OQ-0015). |
| ADR-0018 | Kanonisches Produktattribut-Vokabular                                         | Accepted | Geschäftslogik liest ausschließlich KoalixCRM-eigene kanonische Schlüssel (`koalix.*`); Klassifizierungsstandards (GPC, UNSPSC, eCl@ss, ETIM) sind Adapter, die Mappings in das EAV-System (ADR-0004) einspeisen; lizenzpflichtiger Inhalt wird nie gebündelt. |

---

## Anforderungen und Use Cases

- Anforderungen: `requirements/REQ-*.md` (owned by `kxcrm-requirements-engineer`)
- Use Cases: `use_cases/use_case_*.md` (owned by `kxcrm-requirements-engineer`)
- Qualitätsattribut-Szenarien: `quality_attribute_scenario_*.md` (owned by
  `kxcrm-requirements-engineer`)
- Glossar: `glossar.md` (owned by `kxcrm-requirements-engineer`)
