# 0025 — Autoritätssignale bei föderierter Identität: Namensraum per Konstruktion, Grant statt Gruppenname

- **Status:** Accepted
- **Date:** 2026-08-22

## Context

KoalixCRM föderiert Kundenverzeichnisse. `koalixcrm/auth/oidc_backend.py` übernahm bis zu dieser
Entscheidung jeden Claim-Wert unverändert als Django-Gruppe:

```python
for group_name in groups_to_add:
    group, _ = Group.objects.get_or_create(name=group_name)
    user.groups.add(group)
```

Unabhängig davon leitete `koalixcrm/core/access.py` den *unrestricted actor* — die Akteursklasse,
die nicht auf ihre Rollen-Workspaces beschränkt ist — aus der Mitgliedschaft in einer über
`settings.M2M_MICROSERVICE_GROUP_NAME` benannten Gruppe ab.

Zusammengesetzt ergab das eine Rechteausweitung: **ein Identity Provider, der einen Claim mit
genau diesem Wert ausstellt, verschafft dem Träger workspace-übergreifende Reichweite** — ohne
dass ein KoalixCRM-Codepfad oder ein Administrator die Gruppe angelegt oder die Zuweisung geprüft
hätte. `get_or_create` legt die Gruppe beim ersten Auftreten selbst an.

Der Docstring der alten Implementierung verteidigte den Ansatz damit, die Gruppe werde
*"administered by hand in each environment, never created by code"*. Genau diese Prämisse hält in
KoalixCRM nicht — die Gruppensynchronisation ist additiv und codegetrieben.

Die Ausnutzbarkeit war bedingt, nicht universell: `base_settings.py` setzt die Einstellung
standardmässig auf `''`, und die Prüfung liefert bei leerem Wert `False`. Der Defekt war also in
jeder Umgebung *ohne* konfiguriertes M2M-Dienstkonto wirkungslos und in jeder *mit* — also
überall dort, wo das System produktiv arbeitet.

Dieselbe Defektklasse wurde in der Workflow Support Webapp unter QUAQ2-403/QUAQ2-398 behoben und
in deren ADR-43 und ADR-44 festgehalten. KoalixCRM trug das Muster unmitigiert.

## Decision

Zwei getrennte Entscheidungen, die zusammen die Komposition auflösen. Sie sind bewusst getrennt,
weil jede für sich eine andere Eigenschaft herstellt.

### 1. Namensraum per Konstruktion, nicht per Prüfung (koalixcrm#430)

Jeder claim-abgeleitete Gruppenname wird bedingungslos als
`oidc:<tenantAlias>:<claimValue>` gebildet — ohne Sonderfall für irgendeinen Claim-Wert.

Der entscheidende Punkt: eine Allow-List und eine Deny-List *beantworten* die Kollisionsfrage
beide, ein Name, den ein Claim syntaktisch nicht erzeugen kann, *beseitigt* sie. Es ist deshalb
keine Sanitisierung nötig und keine Pflege einer Liste, die mit jedem neuen lokal bedeutsamen
Gruppennamen veralten würde.

`<tenantAlias>` stammt ausschliesslich aus `core.OidcTenant.alias`, aufgelöst über den
**validierten** Issuer des Tokens — nie aus Claim-Inhalt. Ein IdP kann formen, was ein Claim
sagt; er kann keine Zeile in dieser Tabelle anlegen.

**Fail closed:** Existiert für den validierten Issuer keine `OidcTenant`-Zeile, entfällt die
Synchronisation für diesen Request vollständig — keine Gruppe, keine Mitgliedschaftsänderung,
keine Exception; die Authentifizierung läuft auf den bereits vorhandenen Gruppen weiter. Das
Anlegen der Tenant-Zeile *ist* das Opt-in pro Issuer.

Zusätzlich wird der `iss`-Claim des Tokens gegen den Issuer geprüft, gegen den der aufrufende
Pfad tatsächlich validiert hat. Das macht die Eigenschaft "der Alias stammt aus dem validierten
Issuer" *für diesen Request* wahr statt nur konstruktionsbedingt wahr. Eine Abweichung degradiert
wie ein unregistrierter Tenant: überspringen, protokollieren, nie die Anmeldung blockieren.

### 2. Ein eigenes Grant-Modell statt eines Gruppennamens (koalixcrm#432)

`_is_m2m_microservice_account()` liest `core.ServiceAccountGrant` — eine reine
FK-Existenzprüfung, unabhängig von Gruppenmitgliedschaft und Gruppennamen.

Entscheidung 1 allein würde die konkrete Lücke schliessen. Sie lässt aber die schwächere
Eigenschaft bestehen, dass das Signal für die weitreichendste Autorität des Systems ein
*Stringvergleich gegen einen Gruppennamen* ist — in einem System, dessen Gruppennamen teilweise
extern gespeist werden. Das bleibt ein Refactoring davon entfernt, wieder falsch zu sein.

`M2M_MICROSERVICE_GROUP_NAME` darf weiterhin Django-Modellrechte tragen; die Frage "ist das das
Dienstkonto" beantwortet die Gruppe nicht mehr.

Grant-Zeilen werden **ausschliesslich administrativ** geschrieben: über das nur für Superuser
sichtbare Admin oder das Kommando `grant_service_account`. Kein Authentifizierungs- oder
Synchronisationspfad legt eine an — damit ist der Besitz eines Tokens nie hinreichend, um zum
Dienstkonto zu werden. Aus demselben Grund ist das Kommando bewusst *nicht* in den
Container-Entrypoint verdrahtet: eine automatische Vergabe der weitesten Autorität bei jedem
Start wäre genau die Eigenschaft, die dieses Modell beseitigen soll.

## Consequences

- Bestehende, aus Claims entstandene Gruppen behalten ihre alten unnamensräumigen Namen. Sie
  werden von der Synchronisation nicht mehr getroffen (sie entsteht künftig unter `oidc:`) und
  sind manuell zu prüfen — insbesondere ist `auth_group` auf eine Zeile zu prüfen, die
  `M2M_MICROSERVICE_GROUP_NAME` entspricht.
- Rollenbindungen an claim-abgeleitete Gruppen (`RoleInWorkspace`) müssen auf die neuen Namen
  gesetzt werden. Ein Mandanten-Containment-Invariant auf `RoleInWorkspace` — die Prüfung, dass
  eine `oidc:<alias>:…`-Gruppe nur in einem Workspace desselben Tenants gebunden werden darf
  (WFS ADR-44 §4) — ist hier **noch nicht** umgesetzt und bleibt offen.
- Jede Umgebung braucht einmalig eine `OidcTenant`-Zeile pro Issuer und einen
  `ServiceAccountGrant` für den M2M-Benutzer, sonst verliert das Dienstkonto seine
  workspace-übergreifende Reichweite. Das ist beabsichtigt (fail closed), aber ein
  Betriebsschritt, der in der Inbetriebnahme stehen muss.
- Der Identitätsschlüssel bleibt die E-Mail-Adresse; das ist koalixcrm#431 und hier nicht
  entschieden.
- Die Rollenpolitik-Projektion aus Org-ADR-0013 (`Group.permissions` als Ausgabe statt Eingabe)
  ist weiterhin nicht adoptiert; das ist koalixcrm#433.

## Related

- koalixcrm#430, koalixcrm#432 (umgesetzt), koalixcrm#431, koalixcrm#433 (offen)
- [ADR-0022: Backend Architecture — Org-Wide ADR Binding](0022-backend-architecture-org-binding.md)
- WFS ADR-43 "Externe IdP-Gruppenprovisionierung", WFS ADR-44 "Gruppennamensraum-Schema &
  Mandanten-Containment"
