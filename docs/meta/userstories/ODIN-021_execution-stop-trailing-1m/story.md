# ODIN-021: Stop-Trailing auf 1-Minute-Bar-Close

**Modul:** odin-execution
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-014 (R-Based Profit Protection)

---

## Context

Das Konzept spezifiziert, dass der Trailing Stop auf 1-Minute-Bar-Close nachgezogen wird (nicht auf Decision-Bar-Close). Im Drei-Layer-Timeframe-Modell (Stakeholder-Entscheidung 2026-02-26) ist der Decision-Layer 3m oder 5m (parametrisierbar). Die aktuelle Implementierung aktualisiert den Trail nur bei jeder Decision Loop Iteration (alle 3 oder 5 Minuten). Fuer engere Profit-Sicherung muss der Trail auf 1m reagieren.

## Scope

**In Scope:**
- Neuer Listener/Hook in OMS fuer 1m-Bar-Closes (zwischen Decision-Bars)
- Trail-Update-Logik: Berechne neuen Trail auf jedem 1m-Bar-Close
- Nur POSITIONED Pipeline: Trail wird nur aktualisiert wenn Position offen
- Highwater-Mark-Prinzip bleibt: Trail kann nur steigen
- R-Floor-Integration: max(trail, rFloor) auf jedem 1m-Bar-Close

**Out of Scope:**
- Tick-genaues Trailing (nur auf Bar-Close, nicht sub-minute)
- Aenderungen an der Decision-Loop-Logik (3m/5m Decision-Bar)
- Aenderungen am Entry-ATR-Freeze-Mechanismus (existiert bereits)

## Acceptance Criteria

- [ ] OMS erhaelt 1m-Bar-Close Events (neuer Callback oder Listener)
- [ ] Trail wird auf jedem 1m-Bar-Close neu berechnet
- [ ] Formel: candidateTrail = intradayHigh - trailFactor * entryATR
- [ ] effectiveTrail = max(highwaterMark, candidateTrail)
- [ ] R-Floor wird ebenfalls auf 1m geprueft: effectiveStop = max(effectiveTrail, rFloor)
- [ ] Wenn effectiveStop > aktueller Stop -> Stop-Order wird beim Broker angepasst
- [ ] Stop-Order-Anpassung: Cancel + New (nicht Modify)
- [ ] Maximale Stop-Anpassungs-Frequenz: 1x pro Minute (nicht bei jedem Tick)

## Technical Details

**Dateien:**
- `odin-execution/src/main/java/de/its/odin/execution/oms/TrailingStopManager.java` (neue Klasse)
- `odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java` (Integration)
- `odin-core/src/main/java/de/its/odin/core/pipeline/TradingPipeline.java` (1m-Callback-Routing)

**Patterns:**
- `TrailingStopManager` als POJO (kein Spring-Bean, Pro-Pipeline-Instanz)
- Highwater-Mark als `private` Zustandsvariable in `TrailingStopManager`
- Stop-Anpassung immer ueber `BrokerGateway`-Port-Interface (niemals direkt IB-API)

## Concept References

- `docs/concept/07-oms-execution.md` -- Abschnitt 9 "Stop-Trailing auf 1m-Bar-Close"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 1 "ATR-Freeze bei Entry"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 5 "Runner-Trailing-Regeln"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- "Trailing Stop = Fallnetz, darf nur steigen, nie fallen"

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.execution.trailing.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen `BrokerGateway`-Interface aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `TrailingStopManager`: Trail-Update auf 1m (vs. 3m)
- [ ] Unit-Tests: Highwater-Mark-Prinzip (Trail kann nur steigen)
- [ ] Unit-Tests: `max(trail, rFloor)` Logik
- [ ] Unit-Tests: Stop-Order-Anpassung (Cancel + New)
- [ ] Unit-Tests: Maximale 1x/Minute Frequenz (kein Doppel-Update)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `BrokerGateway` und `EventLog` Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `TrailingStopManager` + echtem `OrderManagementService` zusammengeschaltet
- [ ] Integrationstest: 1m-Bar-Sequence mit steigendem und fallendem Preis (Trail bleibt am Highwater)
- [ ] Integrationstest: R-Floor wirkt als Mindest-Stop wenn Trail darunter liegt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — diese Story hat keinen Datenbankzugriff. `TrailingStopManager` ist ein zustandsbehaftetes POJO ohne Persistence.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `TrailingStopManager`-Klasse, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei identischen 1m-Bars? Bei Luecken (Gap-Down)? Bei IB Pacing Limit Verletzung?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert (was vorgeschlagen, was umgesetzt/verworfen)

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions, Null-Safety, Ressource-Leaks — insbesondere bei Cancel+New-Order-Sequenz"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/07-oms-execution.md` (Abschnitt 9) + `docs/concept/05-stops-profit-protection.md` (Abschnitt 5) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Highwater-Mark, ATR-Freeze, R-Floor-Integration und Frequenz-Limit dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche praktischen Szenarien bei IB Stop-Order-Management koennten auftreten, die im Konzept nicht behandelt sind? (z.B. partielle Fills, Order-Rejection bei Cancel)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Frequenz-Limiting-Strategie, Cancel+New vs. Modify)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- Im Sim-Modus: BrokerSimulator.advanceBar() muss auch 1m-Bars verarbeiten
- Achtung: Zu haeufige Stop-Anpassungen koennen IB Pacing Limits verletzen -> maximal 1x/Minute
- Entry ATR ist bereits in TradingPipeline.positionEntryAtr eingefroren
- Highwater-Mark-Prinzip ist absolute Kernregel: Trail darf NIEMALS fallen (CLAUDE.md)
