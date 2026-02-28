# ODIN-028: Flash Crash Detection und Kill-Switch Enhancement

**Modul:** odin-core
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-005 (Monitor Events), ODIN-026 (Degradation Manager)

---

## Context

Der aktuelle Kill-Switch reagiert auf DQ-Eskalation und manuellen Trigger. Das Konzept definiert zusaetzliche automatische Trigger: Flash Crash (>5% in <1 Min + Volume >3x), Heartbeat-Timeout (3x30s), und Tages-Drawdown (realized + unrealized). Diese muessen in den KillSwitchService integriert werden.

## Scope

**In Scope:**
- Flash-Crash-Detektion: >5% Preisbewegung in <1 Minute + Volume > 3x Durchschnitt
- Tages-Drawdown-Pruefung: Echtzeit (realized + unrealized) gegen 10% Hard-Stop
- Heartbeat-Integration: 3 x 30s ohne Heartbeat -> Kill-Switch
- Kill-Switch-Latenz SLO: < 2 Sekunden von Trigger bis Order-Submission
- Kill-Switch-Aktion: Stornieren + Market-Close + DAY_STOPPED + EMERGENCY Alert

**Out of Scope:**
- Flash-Crash-Detection in odin-data (die `CrashDetectionGate` erkennt, odin-core entscheidet)
- Degradation-Mode-Logik (das ist ODIN-026)
- Manuelle Kill-Switch-Ausloesung via API (existiert bereits)

## Acceptance Criteria

- [ ] Flash-Crash: Preis aendert sich >5% in <1 Minute UND Volume > 3x Durchschnitt -> Kill-Switch
- [ ] Tages-Drawdown: Summe(realized + unrealized) > 10% Tageskapital -> Kill-Switch
- [ ] Heartbeat: 3 aufeinanderfolgende verpasste Heartbeats (90s total) -> Kill-Switch
- [ ] Kill-Switch-Latenz: < 2 Sekunden (messbar via EventLog-Timestamps)
- [ ] Kill-Switch-Aktion: 1. Cancel All, 2. Market-Close All, 3. DAY_STOPPED, 4. EMERGENCY Alert
- [ ] Kill-Switch ist idempotent (mehrfacher Aufruf schadet nicht)
- [ ] Kein Re-Entry nach Kill-Switch fuer den Rest des Tages

## Technical Details

**Dateien:**
- `odin-core/src/main/java/de/its/odin/core/service/KillSwitchService.java` (Erweiterung)
- `odin-core/src/main/java/de/its/odin/core/service/GlobalRiskManager.java` (Drawdown-Check)

**Konfiguration:**
- `odin.core.kill-switch.flash-crash-threshold-pct` (Default: 0.05)
- `odin.core.kill-switch.flash-crash-volume-multiplier` (Default: 3.0)
- `odin.core.kill-switch.drawdown-threshold-pct` (Default: 0.10)
- `odin.core.kill-switch.heartbeat-miss-count` (Default: 3)
- `odin.core.kill-switch.heartbeat-interval-seconds` (Default: 30)

**Patterns:**
- Kill-Switch-Ownership: "odin-data eskaliert, odin-core entscheidet" (CLAUDE.md)
- Kill-Switch ist idempotent: Ein Flag `killSwitchTriggered` verhindert Doppel-Ausfuehrung
- Latenz-SLO-Messung: Zeitstempel bei Trigger-Empfang und bei Order-Submission ins EventLog

## Concept References

- `docs/concept/06-risk-management.md` -- Abschnitt 4 "Kill-Switch (Normativ)"
- `docs/concept/06-risk-management.md` -- Abschnitt 4.2 "Ausloeser (automatisch)"
- `docs/concept/06-risk-management.md` -- Abschnitt 4.3 "Aktion"
- `docs/concept/06-risk-management.md` -- Abschnitt 4.5 "Kill-Switch-Latenz (SLO)"
- `docs/concept/11-edge-cases.md` -- Abschnitt 1.1 "Flash Crash Detection"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Singleton-Instanziierung
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `T:\codebase\its_odin\CLAUDE.md` -- "Kill-Switch-Ownership: odin-data eskaliert, odin-core entscheidet"

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (alle Schwellen als Konstanten oder Config)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (Kill-Switch-Trigger-Typ als ENUM)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.kill-switch.*` fuer Konfiguration
- [ ] Port-Abstraktion: `BrokerGateway`- und `EventLog`-Interfaces aus `de.its.odin.api.port` verwenden

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Flash-Crash-Detection (Preis + Volume beide ueber Schwelle)
- [ ] Unit-Tests: Flash-Crash nicht ausgeloest wenn nur eine Bedingung erfuellt (Preis ODER Volume)
- [ ] Unit-Tests: Drawdown-Check (realized + unrealized summiert korrekt)
- [ ] Unit-Tests: Heartbeat-Timeout (3 verpasste Heartbeats -> Trigger)
- [ ] Unit-Tests: Idempotenz (zweiter Kill-Switch-Aufruf hat keine Wirkung)
- [ ] Unit-Tests: Latenz-Messung (via gemockte Clock und Event-Timestamps)
- [ ] Unit-Tests: Kein Re-Entry nach Kill-Switch
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `BrokerGateway`, `EventLog`, `MarketClock`
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `KillSwitchService` + `GlobalRiskManager` — Drawdown-Trigger feuert korrekt
- [ ] Integrationstest: Flash-Crash-Signal aus odin-data-Gate loest Kill-Switch in odin-core aus
- [ ] Integrationstest: Kill-Switch-Sequenz (Cancel -> Market-Close -> DAY_STOPPED -> Alert) in korrekter Reihenfolge
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — `KillSwitchService` und `GlobalRiskManager` sind Singletons ohne direkten Datenbankzugriff. Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `KillSwitchService` (erweitert), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn Cancel-All fehlschlaegt (Broker-Verbindung weg)? Was bei Drawdown-Trigger exakt an der Grenze (9.99%)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Thread-Safety bei Kill-Switch-Trigger-Flag, korrekte Atomaritaet bei Idempotenz-Guard, Latenz-SLO-Messung"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/06-risk-management.md` (Abschnitt 4) + `docs/concept/11-edge-cases.md` (Abschnitt 1.1) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Flash-Crash-Schwellen, Heartbeat-Logik, Drawdown-Berechnung und Kill-Switch-Sequenz dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche Kill-Switch-Szenarien koennten in der Praxis auftreten, die im Konzept nicht behandelt sind? (z.B. Kill-Switch waehrend EOD-Sequenz, Broker-Timeout beim Cancel-All)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Idempotenz-Implementierung, Latenz-Messstrategie)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- Kill-Switch-Ownership: "odin-data eskaliert, odin-core entscheidet" (CLAUDE.md)
- Flash-Crash-Detection: Die CrashDetectionGate in odin-data erkennt den Crash, die Kill-Switch-Entscheidung trifft odin-core
- Unrealisierte Verluste muessen periodisch aktualisiert werden (via PositionUpdate Events)
