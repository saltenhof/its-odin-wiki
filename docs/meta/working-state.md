# Working State — Architekturkonzept ODIN

Stand: 2026-02-15

---

## Aktueller Stand

### Alle Kapitel 0–10: FERTIG (v0.1/v0.2)

Alle 11 Kapitel sind geschrieben und haben ChatGPT R1 durchlaufen. Kapitel 0 hat zusaetzlich R2.

| Kapitel | Status | ChatGPT |
|---------|--------|---------|
| 0 — Gesamtuebersicht | v0.2 | R1+R2+CrossReview GO |
| 1 — Modularchitektur | v0.1 | R1 GO |
| 2 — Echtzeit-Datenpipeline | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 3 — Broker-Anbindung | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 4 — KPI-Engine | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 5 — LLM-Integration | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 6 — Rules Engine | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 7 — OMS | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 8 — Datenmodell | v0.1 | R1 GO m.B. → Fixes eingearbeitet + CrossReview |
| 9 — Frontend | v0.1 | R1 GO m.B. → Fixes eingearbeitet |
| 10 — Deployment | v0.1 | R1 GO m.B. → Fixes eingearbeitet + CrossReview |

### Uebergreifende Architekturentscheidungen (aus Reviews)

- Kill-Switch-Ownership: odin-data ESCALATEt, odin-core entscheidet
- Regime-Bestimmung: KPI deterministisch (primaer), LLM sekundaer
- Risk-Limits: Global ueber alle Pipelines (AccountRiskState in odin-core)
- VWAP: Source-of-Truth ausschliesslich in odin-data
- Currency/Exchange-Awareness: Contract-Waehrung + Exchange-TZ durchgaengig
- Crash-Recovery: GTC-Stops beim Broker + Startup-Reconciliation
- Decision Loop: Single-threaded pro Pipeline, KPI vor Rules
- DDD-Modulschnitt: Entities gehoeren in ihre Domaenen-Module, odin-persistence = reine Infrastruktur
- Drain-Mechanismus: Zwei Schreibpfade — direkte Domain-Persistierung + AuditEventDrainer nur fuer EventRecord
- Library-Regel: Domain-spezifisch (gekapselt) vs. Infrastruktur (cross-cutting)
- Transaktionsgrenzen: Pro-Pipeline-POJOs nutzen TransactionTemplate statt @Transactional

### Was als naechstes zu tun ist

**PHASE: Tiefenueberarbeitung aller Kapitel — ABGESCHLOSSEN (2026-02-15)**

Alle 11 Kapitel (0→10) wurden linear mit ChatGPT in mehreren Iterationen ueberarbeitet. Jedes Kapitel hat mindestens 2 Review-Runden durchlaufen und GO erhalten. Insgesamt 30 Review-Iterationen ueber alle Kapitel.

**Querschnittsthemen (durchgehend eingearbeitet):**
- Backtest/Simulation: Tagesweise Replay, BrokerSimulator, MarketClock (Live/Sim), RunContext mit mode-Flag
- Cross-Kapitel-Konsistenz: instrumentId, Pipeline-States (Kap 6 FSM), OMS-States (Kap 7), brokerOrderId, EventLog non-droppable
- Lean-Bereinigung: Audit-Chain/HMAC entfernt, WebSocket entfernt, EMERGENCY-Level entfernt, pipelineId eliminiert
- Neue Konzepte: TradingRun-Outcome (COMPLETED/ABORTED/ERROR), Position-Safety-Invariante, EventLog-Overflow-Semantik

**Cross-Kapitel-Gesamtreview Runde 1 (2026-02-15): ABGESCHLOSSEN — FINAL GO**

Alle 11 Kapitel als Gesamtheit reviewed (merge.txt → ChatGPT). Ergebnis:
- 1 KRITISCHES Finding (P0): Widerspruch "nicht intraday restartable" vs WinSW Auto-Restart → behoben durch Safe-Mode-Konzept (Kap 0 §10, Kap 8 §6, Kap 10 §4/§5/§6)
- 0 WICHTIGE Findings die echte Cross-Kapitel-Widersprueche waren
- Abgelehnte Vorschlaege: DecisionCycleId (Over-Engineering), Risk/LLM Budget Fairness (Implementierungsdetail)
- 5 Konsistenz-Nacharbeiten im Safe-Mode-Fix: ERROR-Semantik bei Mehrfach-Restart, Startup-Sequenz-Reihenfolge, State-Recovery-Filter, DAY_STOPPED-Semantik-Trennung, Runbook-Ergaenzung
- ChatGPT: FINAL GO

**Cross-Kapitel-Gesamtreview Runde 2 (2026-02-15): ABGESCHLOSSEN — FINAL GO**

Erneute Pruefung mit frischen Augen. Ergebnis:
- 1 Finding P0 (WICHTIG): Spool-Droppability in Kap. 10 pauschal "non-droppable" vs. eventklassenabhaengige Policy in Kap. 2 → Formulierung praezisiert
- 1 Finding P1 (WICHTIG): Safe-Mode ohne Broker-Verbindung (IB Gateway manueller Login) → DEGRADED Safe-Mode eingefuehrt (Dashboard/Forensik verfuegbar, Reconciliation pending)
- 1 Finding P2 (HINWEIS): Spool In-Memory Forensik-Limit → bereits in Kap. 2 dokumentiert, kein Fix noetig
- 5 Mitzug-Stellen in Kap. 8 und 10 konsequent nachgezogen (Reconciliation konditional, Dashboard pending)
- Abgelehnter Vorschlag: Tick-Queue vs. EventLog-Dropping Einzeiler (ueberfluessig, Kontext klar)
- ChatGPT: FINAL GO

**DDD-Modulschnitt (2026-02-15): ABGESCHLOSSEN**

odin-persistence aufgeloest → Entities wandern in Domaenen-Module, neues odin-audit Modul, odin-persistence auf reine Infrastruktur reduziert. 10 statt 9 Maven-Module. ChatGPT-Review der geaenderten Kapitel durchlaufen, alle kritischen Findings eingearbeitet.

**Naechste Phase (offen):**
- Entscheidung: Implementierung starten oder weitere Verfeinerung

**Fortschritt:**
| Kapitel | Tiefenreview | Status |
|---------|-------------|--------|
| 0 — Gesamtuebersicht | FERTIG (v0.7) | 3 Iterationen + Cross-Review + DDD-Modulschnitt, FINAL GO |
| 1 — Modularchitektur | FERTIG (v0.5) | 3 Iterationen + DDD-Modulschnitt + ChatGPT-Review, GO |
| 2 — Echtzeit-Datenpipeline | FERTIG (v0.4) | 3 Iterationen, GO |
| 3 — Broker-Anbindung | FERTIG (v0.3) | 3 Iterationen (REWORK+R2+R3), GO |
| 4 — KPI-Engine | FERTIG (v0.3) | 3 Iterationen (REWORK+R2+R3), GO |
| 5 — LLM-Integration | FERTIG (v0.3) | 3 Iterationen (REWORK+R2+R3), GO |
| 6 — Rules Engine | FERTIG (v0.3) | 2 Iterationen (REWORK+R2), GO |
| 7 — OMS | FERTIG (v0.3) | 4 Iterationen (REWORK+R2+R3+R4), GO |
| 8 — Datenmodell | FERTIG (v0.6) | 3 Iterationen + Cross-Review R1+R2 + DDD-Modulschnitt + ChatGPT-Review, FINAL GO |
| 9 — Frontend | FERTIG (v0.3) | 2 Iterationen (REWORK+R2), GO |
| 10 — Deployment | FERTIG (v0.6) | 3 Iterationen + Cross-Review R1+R2 + DDD-Modulschnitt, FINAL GO |
| Guardrail Module Structure | FERTIG (v1.2) | DDD-Modulschnitt |
| Guardrail Development Process | FERTIG (v1.2) | DDD-Modulschnitt + WebSocket→REST Fix |
| Guardrail Frontend | FERTIG (v1.0) | Erstversion, Feature-basierte Struktur, TS-Regeln |

### Wichtige Arbeitsanweisung vom Benutzer

- **ChatGPT als Sparring-Partner:** Ebenbuertig, nicht als Autoritaet. Feedback kritisch bewerten
- **ChatGPT-Nutzung:** STRIKT SEQUENTIELL. Nie parallel. Max 20 Requests pro Kontext, dann neuer Chat
- **Kapitelweise vorgehen:** Nicht alles auf einmal
- **Playbook immer aktuell halten**
- **Lean halten:** Professionell aber nicht over-engineered

### Dateien

| Datei | Status |
|-------|--------|
| `intraday-agent-concept.md` | v1.5 (Multi-Instrument eingearbeitet) |
| `architecture/PLAYBOOK.md` | Aktuell (alle Kapitel FERTIG) |
| `architecture/00-system-overview.md` | v0.7 FERTIG (Tiefenreview + Cross-Review + DDD-Modulschnitt) |
| `architecture/01-module-architecture.md` | v0.5 FERTIG (Tiefenreview + DDD-Modulschnitt) |
| `architecture/02-realtime-pipeline.md` | v0.4 FERTIG (Tiefenreview) |
| `architecture/03-broker-integration.md` | v0.3 FERTIG (Tiefenreview) |
| `architecture/04-kpi-engine.md` | v0.3 FERTIG (Tiefenreview) |
| `architecture/05-llm-integration.md` | v0.3 FERTIG (Tiefenreview) |
| `architecture/06-rules-engine.md` | v0.3 FERTIG (Tiefenreview) |
| `architecture/07-oms.md` | v0.3 FERTIG (Tiefenreview) |
| `architecture/08-data-model.md` | v0.6 FERTIG (Tiefenreview + Cross-Review R1+R2 + DDD-Modulschnitt) |
| `architecture/09-frontend.md` | v0.3 FERTIG (Tiefenreview) |
| `architecture/10-deployment.md` | v0.6 FERTIG (Tiefenreview + Cross-Review R1+R2 + DDD-Modulschnitt) |
| `architecture/guardrail-module-structure.md` | v1.2 FERTIG (ChatGPT R1+R2 GO + DDD-Modulschnitt) |
| `architecture/guardrail-development-process.md` | v1.2 FERTIG (DDD-Modulschnitt + WebSocket→REST Fix) |
| `architecture/guardrail-frontend.md` | v1.0 FERTIG (Erstversion) |
| `architecture/WORKING-STATE.md` | Diese Datei |

### Guardrails (Kurzfassung)

- Java 21, Spring Boot, Maven Multi-Modul
- IB Gateway + TWS API 10.40
- ta4j, PostgreSQL, React Frontend
- Claude Agent SDK (primaer) + OpenAI API (alternativ, kein Runtime-Failover)
- On-Prem Windows, 2–3 Instrumente parallel, isolierte Pipelines
- Namespace: `odin.*`, Package: `de.odin.*`
- ODIN = Orderflow Detection & Inference Node (eigenstaendig, kein MOSES)
