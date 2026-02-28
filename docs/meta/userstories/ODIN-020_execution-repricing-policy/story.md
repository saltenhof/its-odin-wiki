# ODIN-020: Repricing Policy fuer Limit-Orders

**Modul:** odin-execution
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine

---

## Kontext

Das Konzept definiert eine Repricing-Policy fuer nicht-gefuellte Limit-Orders: Alle 5 Sekunden wird der Limit-Preis an den aktuellen Markt angepasst, maximal 3 Repricing-Zyklen. Bei Ueberschreitung degradiert der Entry zu OBSERVING. Die aktuelle OMS-Implementierung hat keine Repricing-Logik.

## Scope

**In Scope:**
- Neue Klasse `RepricingManager` in `de.its.odin.execution.oms`
- Timer-basierte Repricing-Logik (5s Intervall, konfigurierbar)
- Max 3 Repricing-Zyklen pro Entry-Versuch
- Cancel + Replace Mechanismus (bestehende Order stornieren, neue mit angepasstem Preis)
- Degradation bei Ueberschreitung: Entry abbrechen, Pipeline zurueck zu OBSERVING
- Monitoring: Cancel/Replace-Rate Alert bei > 5:1 (30 Min)

**Out of Scope:**
- Aenderungen am IB TWS Adapter (Cancel/Replace nutzt bestehende BrokerGateway-Methoden)
- Stop-Order-Repricing (nur Limit-Entry-Orders)
- Modify-Order (IB TWS API unterstuetzt kein Modify -- immer Cancel + New)

## Akzeptanzkriterien

- [ ] `RepricingManager` wird nach Entry-Submission gestartet
- [ ] Nach 5s ohne Fill: Cancel bestehende Order + Submit neue Order mit aktuellem Preis
- [ ] Maximale Repricing-Zyklen: 3 (konfigurierbar)
- [ ] Nach 3 Zyklen ohne Fill: Entry aufgeben, Pipeline -> OBSERVING
- [ ] Stop-Level wird bei Repricing NICHT angepasst (Entry-ATR bleibt frozen)
- [ ] Neuer Preis basiert auf aktuellem MarketSnapshot.currentPrice (+ konfigurierbarer Offset)
- [ ] EventLog: REPRICING_ATTEMPT mit altem und neuem Preis
- [ ] Alert: Cancel/Replace-Rate > 5:1 in 30 Minuten

## Technische Details

**Dateien:**
- `odin-execution/src/main/java/de/its/odin/execution/oms/RepricingManager.java` (neue Klasse)
- `odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java` (Integration: RepricingManager starten/stoppen)
- `odin-execution/src/main/java/de/its/odin/execution/config/ExecutionProperties.java` (Schwellen: Intervall, Max-Zyklen, Offset, Alert-Rate)

**Kritisch:** `MarketClock` verwenden fuer Timer-Logik (Sim-Kompatibilitaet). Kein `Instant.now()` im Trading-Codepfad.

## Konzept-Referenzen

- `docs/concept/07-oms-execution.md` -- Abschnitt 7 "Repricing Policy"
- `docs/concept/07-oms-execution.md` -- Abschnitt 7, Tabelle "Repricing-Parameter"
- `docs/concept/06-risk-management.md` -- Abschnitt 8 "Anti-Overtrade: Max Cancel/Replace-Rate"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- "MarketClock verwenden: kein Instant.now() im Trading-Codepfad"

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (DEFAULT_REPRICING_INTERVAL_SECONDS = 5, DEFAULT_MAX_REPRICING_CYCLES = 3, ALERT_RATE_THRESHOLD = 5)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.execution.oms.repricing.*`
- [ ] Port-Abstraktion: `MarketClock` aus `de.its.odin.api.port` verwenden (KEIN `Instant.now()`)
- [ ] `BrokerGateway`-Port fuer Cancel/Replace verwenden (nicht direkte IB-API)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: 3 Repricing-Zyklen + Degradation (nach 3 erfolglosen Zyklen: Entry aufgegeben, OBSERVING)
- [ ] Unit-Test: Timer-Logik (5s Intervall via MockMarketClock -- kein Instant.now()!)
- [ ] Unit-Test: Cancel + New wird bei jedem Zyklus korrekt ausgefuehrt (nicht Modify)
- [ ] Unit-Test: Stop-Level bleibt unveraendert nach Repricing
- [ ] Unit-Test: Cancel/Replace-Rate-Monitoring (> 5:1 in 30 Min -> Alert ausgeloest)
- [ ] Unit-Test: EventLog-Eintrag REPRICING_ATTEMPT mit altem und neuem Preis
- [ ] Unit-Test: Sofortiger Fill bei Zyklus 1 -> kein weiterer Repricing-Versuch
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mock-BrokerGateway der NICHT sofort fuellt (Sim fuellt sofort, braucht Mock fuer Tests)
- [ ] MockMarketClock fuer zeitbasierte Tests (keine echten Waits)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `RepricingManager` zusammen mit `OrderManagementService` (reale Klassen)
- [ ] Integrationstest: Vollstaendiger Entry-Pfad -- Limit-Order Submission -> kein Fill -> Repricing -> Degradation nach 3 Zyklen
- [ ] Integrationstest: RepricingManager korrekt gestoppt wenn Fill empfangen (kein "dangling" Timer)
- [ ] Mindestens 1 Integrationstest der bestaetigt, dass Stop-Level nach Repricing unveraendert ist
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- RepricingManager hat keinen direkten Datenbankzugriff. EventLog-Eintraege ueber Port-Interface.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: RepricingManager-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn Cancel fehlschlaegt (Order bereits gefuellt oder expired)? Was wenn zwischen Cancel und New-Submission ein Fill kommt? Was bei gleichzeitigem Kill-Switch und laufendem Repricing?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions zwischen Cancel und Fill, Timer-Leak wenn RepricingManager nicht gestoppt wird, Null-Safety bei BrokerGateway-Responses, moegliche Instant.now()-Violations"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/07-oms-execution.md` Abschnitt 7 und `docs/concept/06-risk-management.md` Abschnitt 8 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Repricing-Parameter, Degradations-Logik, Cancel/Replace-Rate-Monitoring, Stop-Level-Einfrierung"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Was passiert bei Market-Orders nach dem 3. fehlgeschlagenen Repricing -- soll auf Market umgestellt werden oder strikt aufgegeben? Was wenn der Markt sich schnell gegen uns bewegt waehrend Repricing laeuft?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Timer-Implementierung mit MarketClock, Cancel-Fehlerbehandlung, Alert-Rate-Berechnung)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- MUESSEN MarketClock verwenden fuer Timer-Logik (Sim-Kompatibilitaet!) -- kein `Instant.now()` oder `Thread.sleep()` im Trading-Codepfad
- Im Sim-Modus: BrokerSimulator fuellt sofort -> Repricing wird nie getriggert. Repricing-Tests brauchen einen Mock-BrokerGateway der nicht sofort fuellt
- Cancel + Replace ist KEIN Modify -- IB TWS API unterstuetzt kein Modify, nur Cancel + New Order
- Race Condition beachten: Zwischen Cancel und New-Submission kann ein Fill eingehen -- der Fill-Handler muss idempotent sein
- Alert-Rate-Berechnung: Rollierendes 30-Minuten-Fenster (nicht kumulativ seit Start)
