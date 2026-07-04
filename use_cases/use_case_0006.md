# UC-0006: Dienstleistungsprofil für ein SERVICE-Produkt anlegen und bearbeiten

**ID:** UC-0006  
**Bezug:** ADR-0007, REQ-0016  
**Lizenzseite:** Open-Source-Backend (Datenmodell und API); Closed-Source-Frontend (UI)

---

## Akteure
- **Primär:** Produktmanager (eingeloggter Benutzer mit Schreibrecht auf den aktiven Workspace)
- **System:** KoalixCRM-Backend (DRF), KoalixCRM-Frontend (Next.js/Refine)

## Vorbedingungen
- Das `Product` existiert im aktiven Workspace mit `kind = SERVICE`.
- Der Produktmanager hat Schreibrecht auf `ServiceProfile`.

## Auslöser
Der Produktmanager öffnet die Produktdetailseite eines SERVICE-Produkts. Das Frontend erkennt
`kind = SERVICE` und zeigt den Dienstleistungsprofil-Abschnitt an.

## Hauptablauf

```mermaid
sequenceDiagram
    actor PM as "Produktmanager"
    participant FE as "Frontend<br/>(Next.js)"
    participant BE as "Backend<br/>(DRF)"
    participant DB as "Datenbank"

    PM->>FE: Produktdetailseite eines SERVICE-Produkts öffnen
    FE->>BE: GET /api/products/{id}/service-profile/
    BE->>DB: SELECT ServiceProfile WHERE product_id = {id}
    DB-->>BE: ServiceProfile (oder 404)
    BE-->>FE: 200 OK (oder 404)

    alt ServiceProfile existiert nicht
        FE->>PM: „Dienstleistungsprofil anlegen"-Schaltfläche anzeigen
        PM->>FE: Abrechnungsmodell, Standarddauer,<br/>Leistungsbeschreibung, SLA-Referenz eingeben
        FE->>BE: POST /api/products/{id}/service-profile/<br/>{billing_model, default_duration?,<br/>deliverable, sla_reference?}
        BE->>BE: kind = SERVICE prüfen
        BE->>DB: INSERT ServiceProfile (workspace-scoped)
        DB-->>BE: ServiceProfile-ID
        BE-->>FE: 201 Created — ServiceProfile
    else ServiceProfile existiert
        FE->>PM: Bestehendes Dienstleistungsprofil anzeigen
        PM->>FE: Felder bearbeiten
        FE->>BE: PATCH /api/products/{id}/service-profile/<br/>{geänderte Felder}
        BE->>DB: UPDATE ServiceProfile
        DB-->>BE: OK
        BE-->>FE: 200 OK — aktualisiertes ServiceProfile
    end

    FE->>PM: Aktualisiertes Dienstleistungsprofil anzeigen
```

## Alternativablauf A: Falscher kind-Wert
- Das Backend empfängt eine POST-Anfrage für ein Produkt mit `kind != SERVICE`.
- Das Backend antwortet mit HTTP 400 und einer Fehlermeldung.
- Das Frontend blendet den Dienstleistungsprofil-Abschnitt für Produkte mit anderen kind-Werten aus.

## Alternativablauf B: Doppeltes ServiceProfile
- Das Backend erkennt, dass bereits ein `ServiceProfile` für das Produkt existiert.
- Das Backend antwortet mit HTTP 400 (POST nicht erlaubt; PATCH verwenden).

## Nachbedingungen
- Genau ein `ServiceProfile` ist dem SERVICE-Produkt zugeordnet.
- `billing_model` trägt einen der erlaubten Werte `fixed`, `hourly`, `subscription`, `tiered`.

## Behavioral Acceptance Criteria

### BAC-1: Abschnittsanzeige
- [ ] Der Dienstleistungsprofil-Abschnitt ist auf der Produktdetailseite ausschließlich sichtbar, wenn `kind = SERVICE`.
- [ ] Für Produkte mit anderen kind-Werten ist kein Dienstleistungsprofil-Abschnitt sichtbar oder erreichbar.

### BAC-2: Formular-Verhalten
- [ ] Das Abrechnungsmodell-Feld ist eine Pflichtauswahlliste mit den Werten `fixed`, `hourly`, `subscription`, `tiered`.
- [ ] Das Standarddauer-Feld akzeptiert nur positive Ganzzahlen und zeigt die Einheit (Minuten) als Label.
- [ ] Das Leistungsbeschreibungsfeld (`deliverable`) ist als Pflichtfeld markiert.

### BAC-3: Gleichbehandlung in Produktliste
- [ ] Die Produktlistenseite zeigt SERVICE-Produkte in derselben Liste wie andere Produktarten.
- [ ] Der Filter `kind = SERVICE` in der Produktliste liefert ausschließlich Dienstleistungen.
