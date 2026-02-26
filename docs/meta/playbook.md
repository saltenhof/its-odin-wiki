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



## Session 2 — Implementierungsphase (2026-02-21)

### 16. Playbook-Update: Verbindliche Execution-Regeln

Das Playbook (`docs/meta/playbook.md`) wurde um einen neuen Abschnitt **"User-Story-Umsetzung: Execution-Regeln"** ergaenzt (Commit `adc1216` im Wiki-Repo).

**Neue verbindliche Regeln:**

| Regel | Inhalt |
|-------|--------|
| **Orchestrator-Rolle** | Primary Claude implementiert nichts selbst — nur Koordination, binaere Signale empfangen |
| **Agent-Typen** | Implementierungs-Agent (Code committed) vs. QS-Agent (DoD pruefen, Findings schreiben) |
| **Parallelisierung** | Max. 3 Agents gleichzeitig (Impl. + QS zaehlen zusammen), alle im Background-Mode |
| **Implement → QS → Loop** | Zyklusstruktur: Impl. → QS → PASS/FAIL, max. 3 Runden, dann Eskalation |
| **QS-Reporting** | QS schreibt `qa-report-rN.md` (N = Rundennummer), binaeres Signal an Primary |
| **Commit-Disziplin** | Commit nur durch QS-Agent bei PASS — nicht durch Implementierungs-Agent |
| **Hard Gate** | Nach 3x FAIL: Primary bricht ab, informiert User mit Findings-Pfad |
| **Prompt-Struktur** | 7 Pflichtbestandteile (story.md, Spezifikation, CLAUDE.md, Konzept, Architektur, QS-Report, Runde) |

### 17. Abgeschlossene Stories (QS PASS) — Session 2

9 Stories wurden erfolgreich implementiert, durch QS-Agents geprueft und auf `its-odin-backend/main` committed.

| Story | Modul | Commits | Wesentliche Aenderungen |
|-------|-------|---------|------------------------|
| **ODIN-001** | odin-api | `802ae31`, `cc68ba2` | SubRegime-Modell: Enum + Integration in RegimeState. Integrationstests, ChatGPT-Review, Gemini-Review, Failsafe-Plugin-Fix |
| **ODIN-003** | odin-api | `4ddb4b1` | Gate-Cascade-Modell: GateEvaluation-Record, GateCascadeResult, fromEvaluations-Factory. NPE-Fix, Integrationstests, ChatGPT- + Gemini-Review |
| **ODIN-004** | odin-api | (nach `4ddb4b1`) | DegradationMode-Enum: DEGRADED_EXECUTION als 5. Modus ergaenzt (fehlte in Wave-0-Impl.), Integrationstests |
| **ODIN-008** | odin-data | `ced71f6`, `65c497f` | Session Boundary Detection: Power-Hour-Fenster auf 30 Min korrigiert (15:15-15:45, war 60 Min), Failsafe-Plugin in odin-data pom.xml ergaenzt, SessionPhase JavaDoc-Fix |
| **ODIN-009** | odin-data | `9adbfd2` | Pattern Feature Recognition: PatternFeatureCalculator, 6 Features (GapUp/Down, VolumeSpike, MomentumBurst, NarrowRange, HigherLow), 237 Unit-Tests |
| **ODIN-015** | odin-brain | `6b2596c`, `3385748` | Exhaustion Detection: ExhaustionDetector 3-Pillar-Modell (A AND B AND C statt OR), RSI-Shift-Register-Bug-Fix (falscher Index bei Lookback) |
| **ODIN-020** | odin-execution | `f5e53e5`, `7d3e7c7` | Repricing Policy: RepricingManager, Cancel+Replace-Logik, Interval-Doubling (1min→2min→4min...) |
| **ODIN-044** | odin-persistence | `2751864` | Cycle-Tabelle: Flyway-Migration V0XX, CycleEntity, CycleRepository, Zonky-Integrationstests |
| **ODIN-045** | odin-audit | `523987b` | Hash-Chain-Schema: EventRecordEntity mit hash_hex und previous_hash_hex Spalten (updatable=false), ChatGPT-Edge-Case-Tests |

**Gesamtfortschritt:** 9 von 50 Stories abgeschlossen = **18%**

### 18. Ausstehende Remediation (QS nicht bestanden)

Zwei Stories aus Wave 0 haben QS-Pruefung in Session 2 nicht vollstaendig durchlaufen. Code ist committed, aber Findings aus dem Wave-0-QS-Bericht wurden noch nicht behoben.

| Story | Modul | Status | Naechste Aktion |
|-------|-------|--------|-----------------|
| **ODIN-005** | odin-api | Code vorhanden, systematische Findings offen | Dedizierter Remediation-Agent: `qa-report-r1.md` lesen, alle Findings beheben, erneut QS |
| **ODIN-006** | odin-api | Code vorhanden, systematische Findings offen (u.a. `totalCyclesCompleted` wird nie inkrementiert) | Dedizierter Remediation-Agent: `qa-report-r1.md` lesen, Bug + Prozess-Findings beheben, erneut QS |

### 19. Gesamtfortschritt und naechste Schritte

#### Fortschritt nach Wave

| Wave | Stories | Abgeschlossen | Status |
|------|---------|--------------|--------|
| Wave 0 (Foundation) | 7 | 5 | ODIN-001,003,004,044,045 PASS; ODIN-005,006 Remediation ausstehend |
| Wave 1 (Independent Domain) | 10 | 4 | ODIN-008,009,015,020 PASS; Rest noch nicht gestartet |
| Wave 2-6 | 33 | 0 | Noch nicht gestartet |

#### Naechste Schritte fuer Session 3

1. **Remediation ODIN-005 und ODIN-006** (Wave 0 abschliessen)
   - Je einen dedizierten Remediation-Agenten starten
   - Quellen: `qa-report-r1.md`, `story.md`, User-Story-Spezifikation
   - Wave 0 ist erst vollstaendig wenn alle 7 Stories PASS

2. **Wave 1 fortsetzen** (parallel zu Remediation, andere Module)
   - ODIN-002 (LLM Tactical Parameter Schema, odin-api, abhaengig von ODIN-001 — BEREIT)
   - ODIN-023 (FX-Conversion Sizing, odin-execution, keine Abhaengigkeiten — BEREIT)
   - ODIN-029 (Daily Performance Recording, odin-core, keine Abhaengigkeiten — BEREIT)
   - ODIN-032 (Trade Intent HMAC, odin-audit, keine Abhaengigkeiten — BEREIT)
   - ODIN-035, ODIN-036 (Backtest Governance + Cost Model, keine Abhaengigkeiten — BEREIT)

3. **Execution-Regelkonformitaet sicherstellen**
   - Strikt ein Agent pro Story
   - QS-Agent nach jedem Implementierungs-Agent (kein Ueberspringen)
   - ChatGPT-Sparring fuer Test-Edge-Cases ist Pflicht
   - Gemini-Review alle 3 Dimensionen (Code-Bugs, Konzepttreue, Praxis-Gaps)

### 20. Git-Status nach Session 2

**Wiki-Repo** (`its-odin-wiki`):
- Playbook Execution-Regeln: `adc1216`
- WORKING-STATE Update: dieser Commit

**Backend-Repo** (`its-odin-backend/main`):
- ODIN-001 (SubRegime): `802ae31`, `cc68ba2`
- ODIN-003 (Gate Cascade): `4ddb4b1`
- ODIN-004 (DegradationMode): nach `4ddb4b1`
- ODIN-008 (Session Boundaries): `ced71f6`, `65c497f`
- ODIN-009 (Pattern Features): `9adbfd2`
- ODIN-015 (Exhaustion Detection): `6b2596c`, `3385748`
- ODIN-020 (Repricing Policy): `f5e53e5`, `7d3e7c7`
- ODIN-044 (Cycle Table): `2751864`
- ODIN-045 (Hash Chain Schema): `523987b`
- ODIN-005 und ODIN-006: Code committed, Remediation ausstehend

---

## Session 3 — Implementierungsphase (2026-02-21)

### 21. Abgeschlossene Stories (QS PASS) — Session 3

4 Stories wurden erfolgreich implementiert, durch QS-Agents geprueft und auf `its-odin-backend/main` committed.

| Story | Modul | Commit | Wesentliche Aenderungen |
|-------|-------|--------|------------------------|
| **ODIN-005** | odin-api | (R2 Remediation) | MonitorEventIntegrationTest (4 ITs), Jakarta Validation Annotations auf MonitorEvent ergaenzt, ChatGPT- und Gemini-Reviews vollstaendig, 5 Open Points dokumentiert. Build: 83 UT + 30 IT, SUCCESS |
| **ODIN-006** | odin-api | (R2 Remediation) | Critical Bug gefixt: `totalCyclesCompleted` wurde nie inkrementiert → `GlobalRiskManager.onCycleCompleted()` implementiert. CycleTrackingIntegrationTest (7 ITs), 4 neue CycleContext-Invarianten aus ChatGPT-Review, 8 Open Points dokumentiert. Build: 88 UT + 37 IT, SUCCESS |
| **ODIN-002** | odin-api | (nach ODIN-006) | `LlmTacticalOutput` Record + 7 Enums: `LlmAction`, `ExitBias`, `TrailMode`, `ProfitProtectionProfile`, `TargetPolicy`, `EntryTiming`, `SizeModifier`. `LlmAnalysis` mit `@Deprecated(forRemoval=true)` markiert. 36 UT + 13 IT, SUCCESS |
| **ODIN-007** | odin-data | `cc5de96` | `MonitorEventDetector` mit 9 Detektoren: `CRASH_DOWN`, `SPIKE_UP`, `EXHAUSTION_CLIMAX`, `STRUCTURE_BREAK_1M`, `DQ_STALE`, `DQ_OUTLIER`, `DQ_GAP`, `VWAP_CROSS_1M`, `EMA_CROSS_1M`. In `DataPipelineService` integriert. Critical Bug gefixt: SMA-Self-Inclusion-Fehler. 268 UT + 11 IT, SUCCESS |

**Gesamtfortschritt:** 13 von 50 Stories abgeschlossen = **26%**

### 22. Wave-Status nach Session 3

| Wave | Stories | Abgeschlossen | Status |
|------|---------|--------------|--------|
| Wave 0 (Foundation, 7 Stories) | 7 | 7 | **VOLLSTAENDIG** — ODIN-001,003,004,005,006,044,045 alle PASS |
| Wave 1 (Independent Domain) | ca. 10-15 | 6+ | ODIN-002,007,008,009,015,020 PASS; Rest noch nicht gestartet |
| Wave 2-6 | 33 | 0 | Noch nicht gestartet |

**Wave 0 ist vollstaendig abgeschlossen.**

### 23. Technische Details der Session-3-Stories

#### ODIN-005 Remediation (R2): MonitorEvent-Erweiterungen

- Jakarta Validation Annotations (`@NotNull`, `@NotBlank`, `@Valid`) auf `MonitorEvent` und Untertypen nachgetragen
- `MonitorEventIntegrationTest` als neue `*IntegrationTest`-Klasse: 4 Integrationstests (Serealisierung, Validierung, Feld-Constraints, polymorphe Events)
- Failsafe-Plugin fuer odin-api `pom.xml` aktiviert, damit ITs im Maven-Build ausgefuehrt werden
- ChatGPT-Sparring abgeschlossen (3 Runden), Gemini-Review alle 3 Dimensionen (Code-Bugs, Konzepttreue, Praxis-Gaps)
- 5 Open Points aus den Reviews in `protocol.md` dokumentiert

#### ODIN-006 Remediation (R2): CycleTracking-Korrektur

- **Kritischer Bug:** `GlobalRiskManager.onCycleCompleted()` war nicht implementiert — `totalCyclesCompleted` blieb dauerhaft 0
- Fix: `onCycleCompleted(String instrumentId)` implementiert, inkrementiert Counter und triggert Limit-Pruefung
- 4 neue Invarianten aus ChatGPT-Review in `CycleContext` aufgenommen (z.B. `cycleNumber >= 1`, `maxCycles >= 1`)
- `CycleTrackingIntegrationTest`: 7 Integrationstests (Counter-Inkrementierung, Limit-Enforcement, Grenzbedingungen)
- 8 Open Points aus den Reviews in `protocol.md` dokumentiert

#### ODIN-002: LLM Tactical Parameter Schema

- `LlmTacticalOutput` als immutabler Record (ersetzt das alte `LlmAnalysis`-Modell)
- 7 neue Enums definieren das komplette LLM-Taktik-Vokabular:
  - `LlmAction`: `NO_TRADE`, `ENTER_LONG`, `ADD_TRANCHE`, `REDUCE_POSITION`, `EXIT_FULL`, `TRAIL_ONLY`
  - `ExitBias`: `HOLD`, `TAKE_PARTIAL`, `EXIT_ON_WEAKNESS`, `EXIT_AGGRESSIVE`
  - `TrailMode`: `ATR_TRAIL`, `EMA_TRAIL`, `HIGHLOW_TRAIL`, `FIXED_STOP`
  - `ProfitProtectionProfile`: `NONE`, `CONSERVATIVE`, `MODERATE`, `AGGRESSIVE`
  - `TargetPolicy`: `USE_TARGETS`, `TRAIL_ONLY`, `HYBRID`
  - `EntryTiming`: `IMMEDIATE`, `WAIT_PULLBACK`, `WAIT_CONFIRMATION`
  - `SizeModifier`: `FULL`, `REDUCED`, `MINIMAL`
- `LlmAnalysis` mit `@Deprecated(forRemoval=true)` markiert — Migrationsschutz fuer kuenftige Consumers
- 36 Unit-Tests + 13 Integrationstests, Failsafe-Plugin, ChatGPT- und Gemini-Reviews

#### ODIN-007: MonitorEventDetector

- 9 Detektoren in einer einzigen Klasse, klar getrennte Evaluierungs-Methoden
- **CRASH_DOWN / SPIKE_UP:** Prozentualer Preissprung in einer 1m-Bar ueber konfigurierbarem Schwellenwert
- **EXHAUSTION_CLIMAX:** Kombination aus extremem Volumen + Umkehr-Candle
- **STRUCTURE_BREAK_1M:** Unterbrechung des letzten Swing-Highs/-Lows
- **DQ_STALE / DQ_OUTLIER / DQ_GAP:** Datenqualitaets-Ereignisse direkt aus DQ-Gate-Ergebnissen
- **VWAP_CROSS_1M / EMA_CROSS_1M:** Preis-Kreuzungen ueber/unter VWAP bzw. EMA
- Integration in `DataPipelineService`: nach DQ-Gate-Pruefung, vor KPI-Berechnung aufgerufen
- **Bug-Fix:** SMA-Self-Inclusion — SMA-Berechnung schloss die aktuelle Bar faelschlicherweise in die Lookahead-Pruefung ein
- 268 Unit-Tests + 11 Integrationstests; Commit `cc5de96`

### 24. Naechste Kandidaten fuer Session 4

| Story | Modul | Groesse | Abhaengigkeiten | Prioritaet |
|-------|-------|---------|-----------------|-----------|
| ODIN-010 | odin-brain | M | ODIN-002 (PASS) | Hoch — Subregime-Resolver, erste odin-brain Story |
| ODIN-011 | odin-brain | M | ODIN-010 | Mittel — nach ODIN-010 |
| ODIN-023 | odin-execution | S | keine | Hoch — FX-Conversion Sizing |
| ODIN-029 | odin-core | M | keine | Hoch — Daily Performance Recording |
| ODIN-032 | odin-audit | S | keine | Mittel — Trade Intent HMAC |
| ODIN-035 | odin-backtest | M | keine | Mittel — Backtest Governance |
| ODIN-036 | odin-backtest | M | keine | Mittel — Backtest Cost Model |

### 25. Git-Status nach Session 3

**Wiki-Repo** (`its-odin-wiki`):
- WORKING-STATE Update Session 3: dieser Commit

**Backend-Repo** (`its-odin-backend/main`):
- ODIN-005 (MonitorEvent Remediation R2): committed
- ODIN-006 (CycleTracking Remediation R2): committed
- ODIN-002 (LLM Tactical Schema): committed
- ODIN-007 (MonitorEventDetector): `cc5de96`
- Build-Stand: 268 UT + 37 IT gesamt, alle GREEN
