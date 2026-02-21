# ODIN (Orderflow Detection & Inference Node) — Architekturkonzept Playbook

Dieses Playbook ist die **Kontinuitaetsdatei** fuer die kapitelweise Entwicklung des IT-Architekturkonzepts.
Es wird bei jedem Context-Reset als Einstiegspunkt verwendet, um nahtlos weiterzuarbeiten.

Stand: 2026-02-15

---

## Guardrails (vom Benutzer vorgegeben)

| Bereich | Vorgabe |
|---------|---------|
| Sprache | Java 21 |
| Framework | Spring Boot |
| Broker | Interactive Brokers via IB-Gateway |
| Broker-API | TWS API **10.40** (neueste Version). **ACHTUNG:** Modellwissen oft veraltet — nicht auf Modellwissen verlassen, sondern API-Doku konsultieren |
| Chart-KPIs | ta4j |
| Datenbank | PostgreSQL |
| Frontend | React |
| LLM-Anbindung | Primaer: Claude Agent SDK. Alternativ: OpenAI API (pay-per-use) |
| Betrieb | On-Prem, direkt unter Windows |
| Multi-Instrument | 2–3 Instrumente parallel handelbar, vollstaendig isolierte Pipelines pro Instrument, globaler Risk-Manager |
| Performance | Hochgradige Parallelisierung, End-to-End auf schnelle Reaktionsgeschwindigkeit optimiert (Daten-Subscription → KPI-Berechnung → Entscheidung) |

## Fachkonzept-Referenz

Das fachliche Konzept liegt in `intraday-agent-concept.md` (v1.5). Das Architekturkonzept setzt dieses fachliche Konzept in eine technische Architektur um.

## Vorgehensweise

1. **Gesamtkonzept** auf hoher Flughoehe (Kapitel 0) — grobe Parameter, Systemkontext, Kapitelplan
2. **Einzelkapitel** werden nacheinander mit ChatGPT im Sparring durchgearbeitet
3. Jedes Kapitel wird als eigene Markdown-Datei angelegt
4. Nach Abschluss eines Kapitels: Zusammenfassung hier im Playbook, Verweis auf Datei

## Kapitelplan

| # | Kapitel | Datei | Status | Zusammenfassung |
|---|---------|-------|--------|-----------------|
| 0 | Gesamtuebersicht & Systemkontext | `architecture/00-system-overview.md` | FERTIG (v0.7) | Tiefenreview + Cross-Review + DDD-Modulschnitt, FINAL GO |
| 1 | Modularchitektur & Package-Struktur | `backend/architecture/01-module-architecture.md` | FERTIG (v0.5) | Tiefenreview + DDD-Modulschnitt + ChatGPT-Review, GO |
| 2 | Echtzeit-Datenpipeline | `backend/architecture/02-realtime-pipeline.md` | FERTIG (v0.4) | Tiefenreview: 3 Iterationen, GO |
| 3 | Broker-Anbindung (IB Gateway, TWS API 10.40) | `backend/architecture/03-broker-integration.md` | FERTIG (v0.3) | Tiefenreview: 3 Iterationen (REWORK+R2+R3), GO |
| 4 | KPI-Engine (ta4j, Parallelisierung) | `backend/architecture/04-kpi-engine.md` | FERTIG (v0.3) | Tiefenreview: 3 Iterationen (REWORK+R2+R3), GO |
| 5 | LLM-Integration (Claude SDK, OpenAI) | `backend/architecture/05-llm-integration.md` | FERTIG (v0.3) | Tiefenreview: 3 Iterationen (REWORK+R2+R3), GO |
| 6 | Rules Engine & Decision Loop | `backend/architecture/06-rules-engine.md` | FERTIG (v0.3) | Tiefenreview: 2 Iterationen (REWORK+R2), GO |
| 7 | Order Management System (OMS) | `backend/architecture/07-oms.md` | FERTIG (v0.3) | Tiefenreview: 4 Iterationen (REWORK+R2+R3+R4), GO |
| 8 | Datenmodell & Persistenz (PostgreSQL) | `backend/architecture/08-data-model.md` | FERTIG (v0.6) | Tiefenreview + Cross-Review R1+R2 + DDD-Modulschnitt + ChatGPT-Review, FINAL GO |
| 9 | React-Frontend | `frontend/architecture/09-frontend.md` | FERTIG (v0.3) | Tiefenreview: 2 Iterationen (REWORK+R2), GO |
| 10 | Deployment & Betrieb (Windows, On-Prem) | `backend/architecture/10-deployment.md` | FERTIG (v0.6) | Tiefenreview + Cross-Review R1+R2 + DDD-Modulschnitt, FINAL GO |
| — | Guardrail: Modulstruktur | `backend/guardrails/module-structure.md` | FERTIG (v1.2) | ChatGPT R1+R2 GO + DDD-Modulschnitt |
| — | Guardrail: Entwicklungsprozess | `backend/guardrails/development-process.md` | FERTIG (v1.2) | DDD-Modulschnitt |
| — | Guardrail: Frontend | `frontend/guardrails/frontend.md` | FERTIG (v1.0) | Erstversion, Feature-basierte Struktur, TS-Regeln, SSE/REST |

> Kapitelplan ist vorlaeufig. Kann sich im Laufe der Arbeit aendern (Kapitel splitten, zusammenlegen, neue hinzufuegen).

---

## Arbeitsprotokoll

### 2026-02-14: Projektstart

- Playbook angelegt
- Guardrails dokumentiert
- Kapitelplan erstellt
- Beginn mit Kapitel 0 (Gesamtuebersicht)

### 2026-02-14: Kapitel 0 abgeschlossen (v0.2)

- ChatGPT R1 Feedback eingearbeitet: Snapshot-Modell, IB Reconnect, LLM Safety Contract, SLOs, Operationelle Anforderungen
- ChatGPT R2 Feedback eingearbeitet: 5 Konsistenz-Glaettungen (LLM-Fallback, IB-Reconnect-Policy, SSE/WS, Windows Hardening, Kill-Switch-Semantik)
- A1-A6 vorlaeufig entschieden (Status VORLAEUFIG)
- ChatGPT R2 Urteil: GO
- Naechster Schritt: Kapitel 1 (Modularchitektur)

### 2026-02-14: Kapitel 1 abgeschlossen (v0.1)

- 9 Maven-Module + Parent POM definiert, zyklusfreier Abhaengigkeitsgraph
- Package-Konvention: `de.odin.{modul}`
- Instanziierungsmodell: Singleton (broker, core, persistence, app) vs. Pro-Pipeline (data, brain, execution)
- ChatGPT R1 Feedback eingearbeitet: Contract-Only Wiring, Persistence-Alignment, SSE/WS in odin-app, Pipeline-Scope Regel
- ChatGPT R1 Urteil: GO
- Naechster Schritt: Kapitel 2 (Echtzeit-Datenpipeline)

### 2026-02-14/15: Kapitel 2–10 geschrieben + ChatGPT R1 durchlaufen

- Alle 9 Kapitel (2–10) initial geschrieben und einzeln durch ChatGPT R1 Review geschickt
- Alle Kapitel: **GO MIT BEDINGUNGEN** → Bedingungen (P0, P1, P2) jeweils eingearbeitet
- Uebergreifende Fixes: Kill-Switch-Ownership (odin-core entscheidet), Regime-Bestimmung (KPI primaer, LLM sekundaer), Risk-Limits global (nicht pro Pipeline), VWAP Source-of-Truth (odin-data), Currency/Exchange-Awareness, Crash-Recovery mit GTC-Stops
- Naechster Schritt: A2+A4 finalisieren, M1+M2 klaeren, ggf. R2 fuer einzelne Kapitel

### 2026-02-15: Tiefenueberarbeitung aller Kapitel abgeschlossen

- Alle 11 Kapitel (0→10) linear ueberarbeitet mit ChatGPT-Sparring
- Insgesamt 30 Review-Iterationen ueber alle Kapitel (2-4 pro Kapitel)
- Querschnittsthemen eingearbeitet: instrumentId (kein pipelineId), MarketClock sole time source, Pipeline-States (Kap 6 FSM), Sim-Semantik (Live/Sim dual-mode), EventLog non-droppable
- Lean-Bereinigung: Audit-Chain/HMAC entfernt, WebSocket entfernt, EMERGENCY-Level entfernt
- Neue Konzepte: TradingRun-Outcome (COMPLETED/ABORTED/ERROR), Position-Safety-Invariante, EventLog-Overflow-Semantik (Threshold-ESCALATE), DAY_STOPPED vs HALTED Trennung
- Cross-Kapitel-Fixes: orderId→brokerOrderId (Kap 7+8), LLM Cache-Key mit provider (Kap 5+8), Kill-Switch-Semantik ACCEPTED/REJECTED/NOOP (Kap 9+10)
- Alle Kapitel: GO

### 2026-02-15: Cross-Kapitel-Gesamtreview Runde 1 — FINAL GO

- Alle 11 Kapitel als Gesamtheit (merge.txt) an ChatGPT gesendet
- Fokus: Cross-Kapitel-Konsistenz, Datenfluss-Brueche, State-Machine-Konsistenz, Interface-Konsistenz
- **1 KRITISCHES Finding (P0):** Widerspruch "nicht intraday restartable" (Kap 0) vs WinSW Auto-Restart (Kap 10)
  - Loesung: Safe-Mode-Konzept — WinSW startet Prozess neu, Startup-Recovery erkennt Crash und sperrt Trading
  - 5 Konsistenz-Nacharbeiten: ERROR-Semantik bei Mehrfach-Restart, Startup-Sequenz-Reihenfolge, State-Recovery-Filter, DAY_STOPPED-Semantik-Trennung, Runbook-Ergaenzung
- **0 weitere KRITISCHE oder WICHTIGE Cross-Kapitel-Widersprueche**
- Abgelehnte Vorschlaege: DecisionCycleId (Over-Engineering), Risk/LLM Budget Fairness (Implementierungsdetail)
- Geaenderte Kapitel: 00 (v0.5→v0.6), 08 (v0.3→v0.4), 10 (v0.3→v0.4)
- ChatGPT: FINAL GO

### 2026-02-15: Cross-Kapitel-Gesamtreview Runde 2 — FINAL GO

- Erneute Pruefung mit frischen Augen (merge.txt → ChatGPT, neuer Kontext)
- Fokus: Cross-Kapitel-Konsistenz, Safe-Mode-Konsistenz, Datenfluss, State-Machines, Interfaces
- **1 Finding P0 (WICHTIG):** Spool-Droppability in Kap. 10 pauschal "non-droppable" → widerspricht Kap. 2 eventklassenabhaengiger Policy
  - Fix: Formulierung praezisiert — "eventklassenabhaengig, Verweis auf Kap 2 §8"
- **1 Finding P1 (WICHTIG):** Safe-Mode bei Broker-Unreachability — ODIN bricht ab statt Forensik bereitzustellen
  - Fix: DEGRADED Safe-Mode eingefuehrt (Dashboard/Forensik verfuegbar, Reconciliation pending)
  - 5 Mitzug-Stellen in Kap. 8 und 10 nachgezogen (Voraussetzungen-Tabelle, Startup Schritt 5+6, Safe-Mode §5, Runbook)
- **1 Finding P2 (HINWEIS):** Spool In-Memory Forensik-Limit → bereits in Kap. 2 dokumentiert, kein Fix noetig
- Abgelehnter Vorschlag: Tick-Queue vs. EventLog-Dropping Einzeiler (Kontext bereits klar)
- Geaenderte Kapitel: 08 (v0.4→v0.5), 10 (v0.4→v0.5)
- ChatGPT: FINAL GO
- **Architekturkonzept ist review-komplett (2 Runden Cross-Review)**

### 2026-02-15: Guardrail Frontend erstellt + WebSocket-Bereinigung

- `frontend/guardrails/frontend.md` v1.0 erstellt: DDD/Feature-basierte Projektstruktur, TypeScript-Regeln, State Management, Kommunikation, Komponenten-Regeln, Styling, Build/Tooling, Test-Strategie, Checkliste
- WebSocket-Referenzen in Kap 1 (3 Stellen) und Guardrail Dev Process (1 Stelle) korrigiert → konsistent mit Kap 9 ("Kein WebSocket, REST POST fuer Controls")
- Kapitelplan um alle 3 Guardrails ergaenzt

### 2026-02-15: DDD-Modulschnitt — odin-persistence aufgeloest

- **Motivation:** Zentrales odin-persistence widerspricht DDD — Domaene soll ihre Persistenz besitzen
- **Entscheidungen:**
  - Drain: Domaenen persistieren strukturierte Daten direkt, AuditEventDrainer nur fuer EventRecord
  - ConfigSnapshotEntity → odin-execution (Teil des TradingRun-Kontexts)
  - Library-Regel: Domain-spezifisch (TWS API, ta4j, LLM SDKs = gekapselt) vs. Infrastruktur (JPA, Logging = cross-cutting)
- **Neuer Modulschnitt: 10 Module** (vorher 9)
  - odin-audit NEU: EventRecordEntity, PostgresEventLog, AuditEventDrainer
  - odin-persistence REDUZIERT: nur DataSource-Config, JPA-Infra, Flyway
  - odin-brain ERWEITERT: + LlmCallEntity, DecisionLogEntity
  - odin-execution ERWEITERT: + TradingRunEntity, TradeEntity, FillEntity, ConfigSnapshotEntity
  - odin-core ERWEITERT: + PipelineStateEntity, DailyPerformanceEntity
- **Geaenderte Dateien:** Kap 0 (v0.6→v0.7), Kap 1 (v0.4→v0.5), Kap 8 (v0.5→v0.6), Kap 10 (v0.5→v0.6), Guardrail Module Structure (v1.1→v1.2), Guardrail Dev Process (v1.1→v1.2)
- **ChatGPT-Review der geaenderten Kapitel (Kap 1 + Kap 8):**
  - K1 (KRITISCH): sequenceNumber in EventRecord fehlte → eingebaut + Index
  - K2 (KRITISCH): "synchron im selben Drain-Batch" widersprach Zwei-Pfade-Modell → Konsistenzmodell umformuliert
  - K3 (KRITISCH): State-Recovery-Quelle unklar → PipelineStateRepository als primaere Quelle klargestellt
  - K4 (KRITISCH): Package-Naming fuer odin-audit inkonsistent → explizit in Kap 8 Modulzuordnung
  - W3 (WICHTIG): Cross-Domain-FK als skalare IDs → "kein JPA-Relation"-Hinweise ergaenzt
  - W4 (WICHTIG): Transaktionsgrenzen fuer Pro-Pipeline-POJOs → TransactionTemplate-Hinweis in Kap 1
- **merge.txt** nach Review-Fixes neu generiert (6.437 Zeilen)

---

## User-Story-Umsetzung: Execution-Regeln (verbindlich, ab 2026-02-21)

Diese Regeln gelten fuer alle User-Story-Implementierungen und Remediations. Sie ergaenzen und praezisieren Abschnitt 7 der User-Story-Spezifikation (`docs/meta/user-story-specification.md`).

### Orchestrator-Rolle (Primary Claude)

Der Primary Claude **implementiert nichts selbst**. Er:
- Startet Agents mit klar definierten Auftraegen und Quellverweisen
- Empfaengt nur binaere Rueckmeldungen (Pass / Fail / Eskalation)
- Haelt seinen Kontext sauber — keine Implementierungsdetails, keine Dateiinhalte

### Agent-Typen

| Typ | Aufgabe | Ergebnis |
|-----|---------|---------|
| **Implementierungs-Agent** | Story implementieren oder Findings beheben | Code committed + gepusht |
| **QS-Agent** | DoD pruefen, Findings dokumentieren | Datei geschrieben + binaeres Signal |

### Parallelisierung

- **Maximal 3 Agents gleichzeitig** — Implementierungs- und QS-Agents zaehlen zusammen
- Alle Agents laufen im **Background-Mode** (Primary bleibt responsiv)
- Verschiedene Module koennen parallel bearbeitet werden; gleiches Modul sequenziell

### Implement → QS → Loop

```
fuer jede Story:
  runde = 1
  LOOP:
    starte Implementierungs-Agent (Background)
    warte auf Abschluss
    starte QS-Agent (Background)
    warte auf binaeres Signal:
      PASS → commit (durch QS-Agent) → Story abgeschlossen → BREAK
      FAIL und runde < 3 → runde++ → LOOP (neuer Implementierungs-Agent)
      FAIL und runde == 3 → ESKALATION → Primary unterbricht, User einbeziehen
```

### QS-Agent: Pflichten

1. **Alle DoD-Kriterien** pruefen (gemaess `user-story-specification.md`)
2. **Findings** in Datei schreiben: `temp/userstories/<STORY-ID>/qa-report-r<N>.md`
   - N = Rundennummer (r1, r2, r3)
   - Format: Strukturierte Maengelliste mit Schwere (Bug / Konzepttreue / Prozess / Minor)
3. **Nur binaeres Signal** an Primary zurueckgeben: `PASS` oder `FAIL`
4. Bei `PASS`: Story committen und pushen (einmaliger Commit mit aussagekraeftiger Message)
5. Bei `FAIL`: Findings-Datei ist bereits geschrieben — kein Detail an Primary

### Implementierungs-Agent bei Remediation

- Liest `qa-report-r<N-1>.md` (Findings der vorherigen Runde)
- Behebt alle Findings vollstaendig
- Committe und pushe nach Abschluss

### Prompt-Struktur (Pflichtbestandteile fuer jeden Agent)

Jeder Agent-Prompt muss enthalten:
1. Verweis auf `story.md` (Story-Definition + Akzeptanzkriterien)
2. Verweis auf `user-story-specification.md` (vollstaendige DoD)
3. Verweis auf `CLAUDE.md` (Coding-Regeln, Architektur)
4. Verweis auf relevante Konzeptdateien (aus `docs/concept/`)
5. Verweis auf relevante Architekturkapitel (aus `docs/backend/architecture/`)
6. Verweis auf `qa-report-r<N>.md` (bei Remediation-Agents)
7. Explizite Rundennummer (damit QS-Agent den richtigen Dateinamen waehlt)

### Hard Gate

Nach **3 erfolglosen Runden** (QS-Agent meldet 3x FAIL) wird der Zyklus unterbrochen.
Primary Claude informiert den User mit:
- Story-ID
- Anzahl Runden
- Pfad zur letzten Findings-Datei (`qa-report-r3.md`)
- Empfehlung fuer naechste Schritte

Der User entscheidet dann, ob und wie der Zyklus fortgesetzt wird.

---

## Hinweise fuer Context-Recovery

Wenn du nach einem Context-Reset hier landest:

1. **Lies dieses Playbook** — es enthaelt den aktuellen Stand
2. **Pruefe den Kapitelplan** — welches Kapitel ist IN ARBEIT?
3. **Lies das aktuelle Kapitel** — dort steht der letzte Arbeitsstand
4. **Lies das Fachkonzept** (`intraday-agent-concept.md` v1.5) wenn noetig
5. **Arbeitsweise:** ChatGPT-Sparring via `chatgpt_send` MCP-Tool, Sub-Agent-Modus
6. **Sprache:** Konzept-Markdown auf Deutsch, technische Begriffe/Code auf Englisch
