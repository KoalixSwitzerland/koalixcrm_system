# ADR-0020: Gemeinsame deklarative Attributvalidierung zwischen Backend und Frontend

## Status
Proposed

## Context

ADR-0004 definiert das getypte EAV-Attributsystem mit `AttributeDefinition`-Validierungsregeln
(min, max, regex, enum, required) für einzelne Attribute und `AttributeSet`-Metadaten, die dem
Frontend über den DRF-Endpunkt Feldliste, Reihenfolge, Pflichtfeld-Flags und Validierungsregeln
liefern. Dieses Einzelattribut-Validierungsmodell reicht für feldübergreifende, konditionale
Regeln nicht aus: Eine Abhängigkeit wie „Feld B ist Pflicht, wenn Feld A den Wert X hat" oder
eine Einschränkung wie „Feld C und Feld D schließen sich gegenseitig aus" lässt sich nicht als
Eigenschaft einer einzelnen `AttributeDefinition` ausdrücken.

Das Frontend (Next.js) übernimmt das live-Formular-Verhalten — Felder einblenden/ausblenden,
aktivieren/deaktivieren, Optionslisten einschränken — auf Basis der Attributmetadaten. Ohne
eine gemeinsame Validierungslogik entstehen zwangsläufig zwei voneinander abweichende
Implementierungen: eine im Python-Backend (autoritativer Schutz beim Schreiben) und eine in
TypeScript (UX-Sofortfeedback). Divergenz zwischen diesen beiden Seiten erzeugt inkonsistentes
Verhalten — das Frontend lässt Eingaben zu, die das Backend ablehnt, oder umgekehrt.

Ein kommendes ADR (voraussichtlich ADR-0021) legt die Produktgranularitätstopologie
`ProductFamily (optional) → Product → ProductVariant (≥1)` und einen Attributkaskaden-Mechanismus
fest, bei dem der effektive Attributwert zur Lesezeit als Varianten-Override → Produkt →
Familie-/AttributeSet-Standard aufgelöst wird. Validierungsregeln binden entlang dieser
Kaskade.

## Decision

Bedingte und feldübergreifende Attributvalidierungsregeln werden als serialisierbares,
sprachneutrales Regelformat — gespeichert im Backend als Daten — über den DRF-Endpunkt
zusammen mit den `AttributeSet`-Metadaten ausgeliefert. Das Backend (Python) ist die autoritative
Validierungsinstanz und prüft bei jedem Schreibvorgang erneut; das Frontend (TypeScript) wertet
dieselben Regeldaten für das live-Formular-UX aus. Beide Evaluatoren müssen gegen eine
gemeinsame Konformitätstestsuite mit identischen Regel-Daten-Fixtures bestehen.

## Why

Eine gemeinsame Konformitätstestsuite — eine einzige Sammlung von Fixtures, gegen die sowohl der
Python- als auch der TypeScript-Evaluator laufen — ist der einzige Mechanismus, der Divergenz
zwischen den beiden Stacks strukturell verhindert, ohne Sprachgrenzen aufzulösen oder ein
gemeinsames Laufzeitsystem einzuführen. Regeln als Daten (nicht als Code) erhalten die
ADR-0004-These, dass neue Attributbeschränkungen ohne Code-Deploy und ohne Schema-Migration
konfigurierbar sind.

## Alternatives Considered

- **Validierungslogik dupliziert in beiden Stacks (Python und TypeScript getrennt)** —
  abgelehnt: garantierte Divergenz bei jeder Regeländerung; zweifacher Wartungsaufwand ohne
  strukturelle Sicherheit, dass die beiden Seiten übereinstimmen.
- **Nur Backend-Validierung (Frontend zeigt zurückgegebene Fehlermeldungen an)** — abgelehnt:
  das Produkt benötigt ein reaktives Formular-UX mit sofortiger konditionaler Feld-Sichtbarkeit
  und Aktivierungssteuerung; rein servergesteuertes Feedback genügt dieser UX-Anforderung nicht.
  Das Backend bleibt in jedem Fall die autoritative Seite; diese Alternative schränkt lediglich
  die UX-Qualität ein, ohne die Architektur zu vereinfachen.
- **Code-basierte Pro-Produkt-Python-Validatoren** — abgelehnt: verletzt die ADR-0004-These
  (neue Validierungsregeln dürfen keinen Code-Deploy erfordern) und ist nicht an den
  TypeScript-Evaluator weitergebar; die Frontend-Auswertung würde einen separaten Pfad
  benötigen.
- **Ein einzelnes kompiliertes oder gemeinsames Regel-Laufzeitsystem (z. B. WebAssembly-Modul,
  das sowohl in Python als auch in TypeScript läuft)** — offen gelassen: dieses ADR legt den
  Grundsatz (eine deklarative Spezifikation, duale Auswertung, konformitätsgetestet) fest, nicht
  die Technik. Ein gemeinsames Laufzeitsystem ist eine mögliche Umsetzungstechnik; diese
  Entscheidung ist einem Folge-ADR vorbehalten.

## Consequences

### Positive
- Das Backend bleibt die einzige Definitionsinstanz für Validierungsregeln; die Frontend-Seite
  bezieht alle Regeln über die REST-API (ADR-0002) und trägt keine eigenständige Regelbasis.
- Die Konformitätstestsuite macht Divergenz zwischen Python- und TypeScript-Evaluator sichtbar
  und blockiert Integration bei Abweichung; strukturelles Driften ist ausgeschlossen.
- Neue konditionale Validierungsregeln entstehen als Datenkonfiguration ohne Code-Deploy und
  ohne Schema-Migration, konform mit der ADR-0004-These.
- Regeln binden entlang der Attributkaskade (Variante → Produkt → Familie/AttributeSet), die
  das kommende ADR-0021 festlegt; dieses ADR muss bei dessen Ratifizierung nicht geändert
  werden.

### Negative
- Zwei voneinander unabhängige Evaluatoren (Python und TypeScript) müssen dieselbe Regelsprache
  korrekt implementieren; der Wartungsaufwand für die Konformitätstestsuite ist dauerhaft.
- Die Ausdrucksmächtigkeit der Regelsprache ist durch das gewählte serialisierbare Format
  begrenzt; Validierungsanforderungen, die über das Format hinausgehen, erfordern ein Folge-ADR
  zur Erweiterung des Regelformats.
- Das konkrete Regelformat und die Implementierungstechnik für die duale Auswertung sind noch
  offen; nachgelagerte Implementierungsarbeit blockiert auf dem Folge-ADR, das das Format
  festlegt.

---

## Ausdrücklich nicht im Geltungsbereich dieses ADR

Die folgenden Punkte sind explizit aus diesem ADR ausgeschlossen:

- **Das konkrete Regelausdrucksformat, die Grammatik und der Operatorensatz** — ein Folge-ADR
  definiert das serialisierbare Format, sobald die Anforderungen an den Ausdrucksraum
  ausreichend geklärt sind.
- **Die konkrete Validierungslogik für einzelne Attribute oder Produktkategorien** — diese
  gehört in Anforderungen und Konfiguration, nicht in ein ADR.
- **Jede Implementierung** — dieses ADR beschreibt den Grundsatz; kein Django-Modell, kein
  Serializer, keine Frontend-Komponente und kein Datenbankschema sind hier festgelegt.
- **Interaktive Produktkonfiguration zur Bestellzeit (CPQ / Guided Selling)** — das geführte
  Konfigurieren eines kundenspezifischen Produkts, das eine abgeleitete oder benutzerdefinierte
  SKU erzeugt, ist ein eigenständiges, zukünftiges Thema. Es teilt mit diesem ADR den Begriff
  „konditionale Attributregel", unterscheidet sich jedoch grundlegend im Zeitpunkt (Bestellzeit
  vs. Katalogeditionszeit), im Zustand (flüchtige Konfigurationssitzung vs. persistierter
  Attributwert) und im Ergebnis (abgeleitete SKU vs. validierter Attributwert). Dieses Thema
  ist auf ein späteres ADR zurückgestellt und bleibt bis dahin unberührt.

---

## Lizenzbeschränkung

Die Regelspeicherung und die autoritative Auswertung leben vollständig im Open-Source-Backend
(`/app/koalixcrm`), das als PyPI-Wheel und Docker-Image ausgeliefert wird. Das Regelformat
wird über die öffentliche REST-API (ADR-0002) ausgeliefert; kein Domänen-Code und kein
Regelauswertungs-Modul wird direkt in den geschlossenen Next.js-Frontend-Build importiert.
Das Frontend evaluiert die Regeln eigenständig in TypeScript auf Basis der über die API
empfangenen Daten; diese TypeScript-Implementierung lebt ausschließlich im geschlossenen
Docker-Target (`/app/koalixcrm-frontend`) und ist nicht Teil des PyPI-Wheels.

Die Konformitätstestsuite ist sprachneutral und darf in beiden Repositories referenziert
werden; sie enthält ausschließlich Fixtures (Regeldaten und Testfälle), keinen
applikationsspezifischen Code.

---

## Abhängigkeiten zu bestehenden ADRs

**ADR-0002 (Admin-UI-Framework / REST-API-Brücke):** Die REST-API ist die einzige
Kommunikationsbrücke zwischen Open-Source-Backend und geschlossenem Frontend. Validierungsregeln
werden als Teil der `AttributeSet`-Metadaten über diesen Kanal ausgeliefert; kein Regelcode
überquert die Grenze direkt.

**ADR-0004 (Klassifizierung und erweiterbare Attribute):** Dieses ADR erweitert das
EAV-Attributsystem um feldübergreifende, konditionale Validierungsregeln. Die ADR-0004-These
— neue Attribute und Regeln entstehen ohne Code-Deploy — gilt für das vorliegende ADR
unverändert. `AttributeSet` ist der natürliche Träger der Regelmetadaten.

**ADR-0021 (geplant — Produktgranularität und Attributkaskade):** Dieses ADR referenziert die
Variante → Produkt → Familie/AttributeSet-Kaskade als Bindungsebene für Validierungsregeln.
Sobald ADR-0021 ratifiziert ist, gilt seine Kaskadenregel unmittelbar für die hier beschlossene
Validierungsarchitektur; dieses ADR bedarf keiner Änderung.

## Changelog
- 2026-06-28: Erstentwurf (Status: Proposed). Fixiert den Grundsatz der gemeinsamen
  deklarativen Attributvalidierung (Regeln als Daten, Backend-Autorität, duale Auswertung,
  Konformitätstestsuite). Regelformat, Operatorensatz und Implementierung sind explizit
  zurückgestellt.
