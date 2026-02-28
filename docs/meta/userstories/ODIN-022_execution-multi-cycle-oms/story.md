# ODIN-022: Multi-Cycle OMS Handling

**Modul:** odin-execution
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-006 (Cycle Tracking Model)

---

## Context

Die aktuelle OMS hat eine `reset()`-Methode fuer neue Trading-Zyklen, aber keine Cycle-bewusste Logik. Das Konzept fordert: Cycle-2+-Sizing mit reduziertem Faktor, Cycle-Budget-Tracking, und korrekte Reset-Logik zwischen Cycles innerhalb eines Tages.

## Scope

**In Scope:**
- OMS-Erweiterung: Cycle-Number als Kontext bei Entry
- Sizing-Faktor fuer Cycle 2+ (Default 0.6-0.8, konfigurierbar)
- Budget-Tracking: verbleibendes Risk-Budget nach realisiertem P&L
- Korrekte Reset-Logik: OMS-State zuruecksetzen ohne Tages-Metriken zu verlieren
- Cycle-Wechsel Event im EventLog

**Out of Scope:**
- Cycle-Orchestrierung und Cooling-Off-Logik (das ist ODIN-025 in odin-core)
- Aenderungen am GlobalRiskManager (Cycle-Counter verwaltet odin-core)
- Neue Entries nach Cycle-Limit-Erschoepfung (Pruefung liegt in odin-core)

## Acceptance Criteria

- [ ] `submitEntry()` akzeptiert `CycleContext` als Parameter
- [ ] Cycle-2+ Sizing: `shares = shares_standard * cycle2SizeFactor` (konfigurierbar)
- [ ] Budget-Check: `remainingBudget = (dailyCapital * 10%) - abs(realizedLosses)`
- [ ] Wenn remainingBudget < minBudgetThreshold -> Entry blockiert
- [ ] `reset()` behalt Tages-Metriken (Realized P&L, Round-Trip-Count), setzt nur Order/Position zurueck
- [ ] EventLog: CYCLE_START mit cycleNumber und remainingBudget
- [ ] EventLog: CYCLE_END mit cycleResult (P&L)

## Technical Details

**Dateien:**
- `odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java` (Erweiterung)
- `odin-execution/src/main/java/de/its/odin/execution/risk/PositionSizer.java` (Cycle-Faktor)
- `odin-execution/src/main/java/de/its/odin/execution/config/ExecutionProperties.java` (Cycle-Sizing-Config)

**Konfiguration:**
- `odin.execution.oms.cycle2-size-factor` (Default: 0.7)
- `odin.execution.oms.min-budget-threshold-pct` (Default: 0.02, d.h. 2% des Tageskapitals)

## Concept References

- `docs/concept/07-oms-execution.md` -- Abschnitt 11 "Multi-Cycle OMS Handling"
- `docs/concept/06-risk-management.md` -- Abschnitt 2.2 "Cycle-2+ Sizing"
- `docs/concept/06-risk-management.md` -- Abschnitt 3 "Dynamisches Tages-Budget"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 7 "Multi-Cycle-Day Guardrails"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln, Konfigurationskonventionen

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (z.B. `CycleContext`)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.execution.oms.*` fuer Konfiguration
- [ ] Port-Abstraktion: `EventLog`-Interface aus `de.its.odin.api.port` fuer Logging

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Cycle-2 Sizing (reduzierter Faktor vs. Cycle-1)
- [ ] Unit-Tests: Budget-Tracking (Gewinne erhoehen Budget NICHT — zentrale Regel)
- [ ] Unit-Tests: Reset-Logik (Tages-Metriken bleiben erhalten, Order-State wird zurueckgesetzt)
- [ ] Unit-Tests: Budget-Gate (Entry blockiert wenn Budget unter Threshold)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `EventLog` Port-Interface
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendiger Cycle-1 -> Reset -> Cycle-2 Ablauf mit echtem `PositionSizer`
- [ ] Integrationstest: Budget-Erschoepfung blockiert Cycle-3 Entry
- [ ] Integrationstest: Tages-Metriken akkumulieren korrekt ueber mehrere Cycles
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — `OrderManagementService` und `PositionSizer` sind Pro-Pipeline-POJOs ohne direkten Datenbankzugriff. Cycle-Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `OrderManagementService` (Cycle-Erweiterung), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei Cycle-Reset waehrend offenem Trade? Konkurrente Cycle-Wechsel bei Multi-Pipeline?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Thread-Safety-Probleme beim Tages-Metriken-Zustand, Null-Safety bei CycleContext"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/07-oms-execution.md` (Abschnitt 11) + `docs/concept/06-risk-management.md` (Abschnitt 2.2, 3) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Budget-Logik, Cycle-Sizing-Faktor und Reset-Verhalten dem Konzept entsprechen. Insbesondere: Gewinne erhoehen Budget NICHT"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche Szenarien bei Multi-Cycle-OMS koennten in der Praxis auftreten, die im Konzept nicht behandelt sind?"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. wie Budget-Threshold konfiguriert wird)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- Konzept-Kernregel: "Gewinne erhoehen das Budget NICHT" (`docs/concept/06-risk-management.md`, Abschnitt 3)
- Der Sizing-Faktor wird vom GlobalRiskManager ueber AccountRiskState kommuniziert
- Cycle-Wechsel wird von odin-core orchestriert (ODIN-025), OMS ist nur ausfuehrend
