# ODIN-024: TRAIL_ONLY Override im OMS

**Modul:** odin-execution
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** ODIN-002 (LLM Tactical Schema, insb. TargetPolicy)

---

## Context

Das Konzept definiert TRAIL_ONLY als OMS-Override: Das LLM kann ueber `targetPolicy=TRAIL_ONLY` alle Tranchen-Targets deaktivieren und die gesamte Position gegen den Trail laufen lassen. Aktuell submitted die OMS immer Target-Orders pro Tranche.

## Scope

**In Scope:**
- OMS-Flag `trailOnlyActive` das Target-Order-Submission unterdrueckt
- Wenn `targetPolicy == TRAIL_ONLY`: Keine Target-Orders, nur Stop/Trail
- Aktivierung: Ueber LLM Tactical Output oder manuell via Control-Endpoint
- Deaktivierung: Nur durch neuen LLM-Call mit anderem targetPolicy
- Bestehende Targets muessen storniert werden wenn TRAIL_ONLY aktiviert wird

**Out of Scope:**
- Aenderungen am LLM-Adapter oder Prompt-Schema (das ist ODIN-002)
- Aenderungen am `TrailingStopManager` selbst (das ist ODIN-021)
- Frontend-Controls fuer TRAIL_ONLY (separates Frontend-Ticket)

## Acceptance Criteria

- [ ] Wenn `trailOnlyActive == true`: `submitTargetOrdersForTranches()` wird nicht aufgerufen
- [ ] Wenn TRAIL_ONLY nach Entry aktiviert wird: Bestehende Target-Orders werden storniert
- [ ] Trail schuetzt gesamte Position (nicht nur Runner-Tranche)
- [ ] EventLog: TRAIL_ONLY_ACTIVATED / TRAIL_ONLY_DEACTIVATED
- [ ] Anti-Missbrauch: TRAIL_ONLY kann nur aktiviert werden wenn Position im Gewinn (> 0.5R)

## Technical Details

**Dateien:**
- `odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java` (Erweiterung)

**Patterns:**
- `trailOnlyActive` als atomarer Boolean-State in OMS
- Stornierung bestehender Targets ueber `BrokerGateway`-Port (Cancel-Requests)
- Aktivierungs-Guard: `currentPnlInR >= TRAIL_ONLY_MIN_PROFIT_R` (0.5R als Konstante)

## Concept References

- `docs/concept/07-oms-execution.md` -- Abschnitt 6 "TRAIL_ONLY Override"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 5.1 "Runner-Trail bleibt, Profit-Protection als Floor"
- `docs/concept/04-llm-integration.md` -- Abschnitt 3 "target_policy: TRAIL_ONLY"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `T:\codebase\its_odin\CLAUDE.md` -- "TRAIL_ONLY als OMS-Override — LLM kann Tranchen-Targets deaktivieren"

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `TRAIL_ONLY_MIN_PROFIT_R = 0.5`)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (TargetPolicy als ENUM)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.execution.oms.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen `BrokerGateway`- und `EventLog`-Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: TRAIL_ONLY aktiviert -> keine Target-Orders werden submitted
- [ ] Unit-Tests: Aktivierung storniert bestehende Target-Orders
- [ ] Unit-Tests: Anti-Missbrauch-Guard (nur aktivierbar wenn Position > 0.5R im Gewinn)
- [ ] Unit-Tests: Deaktivierung via neuem LLM-Call mit anderem targetPolicy
- [ ] Unit-Tests: EventLog-Events werden korrekt emittiert
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `BrokerGateway` und `EventLog` Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: OMS-Lifecycle mit TRAIL_ONLY-Aktivierung nach Entry, danach kein Target-Submit
- [ ] Integrationstest: Reaktivierung normaler Target-Orders nach TRAIL_ONLY-Deaktivierung
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — TRAIL_ONLY ist ein In-Memory-OMS-Flag ohne Datenbankzugriff. Events werden ueber `EventLog`-Port geschrieben (in Tests gemockt).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: OMS TRAIL_ONLY-Erweiterung, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei TRAIL_ONLY-Aktivierung wenn Target bereits teilweise gefuellt? Bei Deaktivierung nach Tranche-Fill?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions beim Flag-Zustand, korrekte Reihenfolge Cancel -> Flag-Setzen"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/07-oms-execution.md` (Abschnitt 6) + `docs/concept/05-stops-profit-protection.md` (Abschnitt 5.1) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob TRAIL_ONLY-Semantik, Anti-Missbrauch-Guard und Event-Typen dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche praktischen Szenarien bei TRAIL_ONLY koennten auftreten, die im Konzept nicht behandelt sind? (Partial Fills, gleichzeitige LLM-Empfehlungen)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Implementierung des Anti-Missbrauch-Guards)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- TRAIL_ONLY ist eine User-Stakeholder-Entscheidung (siehe MEMORY.md)
- Die gesamte Position laeuft gegen den Trail — das kann mehr Gewinn bringen bei starken Trends, aber auch mehr Slippage bei Reversals
