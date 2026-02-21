# Working State: Konzeptkonsolidierung ODIN

**Stand:** 2026-02-21
**Kontext:** Diese Datei dokumentiert den aktuellen Arbeitsstand der Konzeptkonsolidierung, damit eine neue Claude-Session nahtlos weiterarbeiten kann.

---

## Was wir getan haben

### 1. Analyse der Konzeptlandschaft (10 Dokumente)

Der User hatte ueber mehrere Iterationen eine gewachsene Konzeptlandschaft aus 10 Dokumenten:

| Dokument | Rolle |
|----------|-------|
| `intraday-agent-concept.md` (FK v1.5) | Aeltestes, umfangreichstes Dokument (~1140 Zeilen). Strategielogik, LLM-Integration, Rules Engine, Anti-Halluzination, Security, Monitoring. Teilweise veraltet (Namespace `moses.*`, Stop-Faktor 1.5x statt 2.2x) |
| `strategy-optimization-sparring.md` | ChatGPT-Sparring: R-basierte Profit-Protection, LLM als Tactical Parameter Controller, Multi-Cycle-Day. Enthaelt bindende Stakeholder-Entscheidungen und Konflikttabelle K1-K19 |
| `odin-master-concept.md` (MC v1.1) | Konsolidierung FK + Sparring. Paradigmenwechsel: "KPI primaer, LLM sekundaer" bei Regime Detection. Neue Inhalte: WARMUP, Subregimes, TACTICAL_EXIT, Exhaustion-Detection |
| `odin-hybrid-intraday-unified-concept-v2_1.md` | "Symmetric Hybrid"-Ansatz: QuantEngine und LLM auf Augenhoehe. 4 Setups (A-D), Dual-Key-Decisions, OHLCV-only |
| `odin-hybrid-intraday-unified-build-spec-v3_0.md` (Build-Spec v3.0) | Technische Implementierungsspezifikation. 3-Timeframe (1m/3m/10m), Reliability-weighted Regime Fusion, Event-Detektoren, Challenger-Suite |
| `evaluationsdesign.md` | Empirischer Vergleich MC vs. Unified: 5 Ablations-Varianten, geometrische Metriken, Walk-Forward |
| `testplan.md` | Formaler Testplan mit Postgres-DDL, Given/When/Then Tests, SQL-Auswertungen |
| `open-points-feedback.md` | Stakeholder-Feedback 1: FusionLab F0-F3, Lambda/Mu, Power-Analyse, Infrastruktur-Bruecke |
| `open-points-feedabk-2.md` | Stakeholder-Feedback 2: Gate-Kaskade, Bucket-Schwellen, FusionLab-Schema, Loss-Cost-Matrix |
| `kompromiss.txt` | L2-Orderbuch-Kompromiss: Alpha/Regime aus OHLCV, L2 nur optional fuer Live-Execution |

### 2. Diskussion und Klaerungen

In der Session wurden folgende Themen im Dialog mit dem User geklaert:

**Regime Detection — Drei Ansaetze:**
- FK: LLM-primaer (LLM bestimmt Regime allein)
- MC: KPI-primaer, LLM-sekundaer (deterministische KPI-Regeln, Konservativitaets-Rangfolge)
- Build-Spec: Reliability-weighted Fusion (dynamische Gewichtung)
- **Entscheidung:** Empirisch loesen ueber Evaluationsdesign

**Stakeholder-Feedback zur Reliability-Kalibrierung:**
- Reliability NICHT "1x/Tag" (EOD) sondern auf Decision-Ebene (50-150 Datenpunkte/Tag statt 1)
- Korrelation loesen ueber EMA/Downweighting
- Regime-Sparsity loesen ueber hierarchisches Pooling mit Shrinkage (globaler Score + Regime-Adjustments)
- Damit entfaellt das Kaltstart-Problem (~5-10 Tage statt 100+)

**Gate-Kaskade fuer Indifferenz-Fall:**
- Gate 0 (Safety): Non-Inferiority-Check (MDD, ES95, TailLossFreq)
- Gate 1: Signifikanter P&L-Vorteil → Unified gewinnt
- Gate 2: Indifferenz → FusionLab als Proxy → Bucket-Robustheit → Default MC
- Unified-Veto als "Default-Upgrade" bei Indifferenz

**FusionLab als eigenstaendiges Instrument:**
- F0 (MC-Regel), F1 (Reliability EOD), F2 (Reliability Decision-Level), F3 (Reliability + Pooling)
- Ground-Truth: 30-Min-ex-post-Labeling
- Loss-Cost-Matrix: asymmetrisch (GT=TREND_DOWN, Pred=TREND_UP = 3.0)

**Alle 5 identifizierten Luecken im Evaluationsdesign wurden geschlossen** (Fusion isoliert, Schwellenwerte, Power-Analyse, Infrastruktur-Bruecke, Datenbedarf).

### 3. Konsolidierung

- Alle 10 Quelldokumente ins Archiv verschoben (`docs/concept/archiv/`)
- Claude-Agent hat 12 thematische Konzeptdateien erstellt (~4.600 Zeilen, ~230 KB)
- ChatGPT hat parallel eine eigene Konsolidierung erstellt (~730 Zeilen eigener Inhalt + ~4.900 Zeilen Verbatim-Kopien)

### 4. Vergleich Claude vs. ChatGPT

Ein Opus-Agent hat systematisch 68 Themen verglichen:
- Claude: ~58 vollstaendig, ~7 oberflaechlich, ~3 fehlend
- ChatGPT: ~18 vollstaendig, ~28 oberflaechlich, ~22 fehlend
- **Claude-Version als Source of Truth gewaehlt**

4 Elemente aus der ChatGPT-Version wurden in die Claude-Dateien integriert:
1. JSON-Schema-Beispiele (QuantOutput, LlmOutput, TradeIntent) → `08-symmetric-hybrid-protocol.md`
2. Traceability-Header (Quellverweise) → alle 12 Dateien
3. Cold-Start-Policy → `02-regime-detection.md`
4. Bucket "Trend Day Grind Up" → `09-backtesting-evaluation.md`

### 5. Finale Strukturierung

Die 12 finalen Dateien wurden aus dem Unterordner direkt nach `docs/concept/` verschoben. Beide Konsolidierungs-Unterordner (Claude + ChatGPT) ins Archiv.

### 6. Nachtrag: Datenqualitaet und Daten-Hygiene

Bei einer Stichprobenpruefung der offenen Punkte wurde festgestellt, dass "Backtesting-Datenqualitaet" (Survivorship Bias, Corporate Actions, Timezone/Kalender) in der Konsolidierung verloren gegangen war. Inhalt existierte in den Archiv-Quellen (Unified-Concept v2.1 Abschnitt 12.2, Build-Spec v3.0 Abschnitt 49.4), fehlte aber in den finalen Dateien.

**Nachgetragen** als neuer Abschnitt 6 "Datenqualitaet und Daten-Hygiene" in `09-backtesting-evaluation.md`:
- 6.1 Corporate Actions (Split-Adjustierung, Dividenden, Konsistenz)
- 6.2 Survivorship Bias (Delistete Aktien, Limitation dokumentieren)
- 6.3 Timezone und Kalender (US/Eastern, NYSE-Kalender, Halts)

### 7. Unabhaengiges Review durch Gemini und ChatGPT (2026-02-21)

Alle 12 Konzeptdateien wurden unabhaengig von **Gemini 3.1 Pro** und **ChatGPT GPT-5.2 Thinking** reviewed. Beide Reviews waren ausfuehrlich (je 3-4 Runden mit Nachfragen).

**Gesamtbewertung:**

| Kriterium | Gemini | ChatGPT |
|-----------|--------|---------|
| Architektur-Qualitaet | 9/10 | Oberes Quartil fuer Retail/Small-Team |
| Risk-Management | 9.5/10 | "Ungewoehnlich detailliert" |
| Rendite-Chance | 6.5/10 | "Moeglich, aber nicht Default-Outcome" |

**Konsens:** Architektur und Risk-Management sind stark. Alpha-Generierung aus OHLCV-only Long-only ist die groesste Herausforderung. LLM-Nutzen liegt realistisch bei Drawdown-/Tail-Risk-Reduktion und besserer No-Trade-Selektion, nicht bei "magischer Alpha-Generierung".

### 8. Einarbeitung der Review-Findings (2026-02-21)

Basierend auf den Reviews wurden **4 P0-Entscheidungen** (kritische Inkonsistenzen), **6 P1-Entscheidungen** (wichtige Luecken) und **1 neue Konzepterweiterung** im Dialog mit dem User getroffen und in die Konzeptdateien eingearbeitet.

#### P0-Entscheidungen (kritische Inkonsistenzen behoben)

**P0-1: LLM liefert keine Preise — OMS bestimmt Entry-Preis deterministisch**
- `entry_price_zone` und `target_price` aus dem LLM-Output-Schema entfernt
- Neue LLM-Felder: `setup_type` (Enum), `tactic` (WAIT_PULLBACK / BREAKOUT_FOLLOW / NO_TRADE), `urgency_level` (LOW / MEDIUM / HIGH)
- Safety-Contract konsistent: "LLM gibt keine numerischen Preise oder Orderdetails aus — weder im Freitext noch in strukturierten Feldern"
- OMS bestimmt Entry-Preis aus Struktur-Levels (Pre-Market H/L, Prior Day C/H/L, erste 1m-Bar, Gap-Fill, runde Zahlen) mit Taktik-zu-Level-Mapping
- Betroffene Dateien: `04-llm-integration.md`, `07-oms-execution.md`, `08-symmetric-hybrid-protocol.md`, `11-edge-cases.md`

**P0-2: Normative Begriffshierarchie Trade / Cycle / Tranche**
- **Trade** = Alles von erster Eroeffnung bis vollstaendiger Schliessung einer Position in einem Instrument
- **Cycle** = Erneuter Trade im selben Instrument am selben Tag (Position komplett geschlossen → neu eroeffnet)
- **Tranche** = Teilstueck innerhalb eines Trades (Aufstocken = neue Tranche, NICHT neuer Cycle)
- Limits: `maxCyclesPerInstrumentPerDay` (Default 3), `maxTranchesPerTrade` (konfigurierbar)
- Betroffene Dateien: `00-overview.md`, `03-strategy-logic.md`, `06-risk-management.md`, `07-oms-execution.md`, `08-symmetric-hybrid-protocol.md`

**P0-3: Datenverfuegbarkeit ist konfigurationsabhaengig**
- Drei Stufen ueber `odin.data.subscriptions`: OHLCV_ONLY / BASIC_QUOTES / L2_DEPTH
- Jedes Feature hat Proxy-Fallback fuer niedrigere Stufen
- Data/Feature-Matrix in `01-data-pipeline.md` (Backtest Standard/Recording × Live OHLCV/BASIC/L2)
- Harte Regel: "Alpha-Signale duerfen nur auf Daten basieren, die im Backtest desselben Laufs verfuegbar waren"
- Recording-Modus: Live-Daten aufzeichnen fuer reichere Backtest-Datenbasis ueber Zeit
- Backtest-Runs mit Datenstufe getaggt (`data_tier` in DB-Schema)
- Betroffene Dateien: `00-overview.md`, `01-data-pipeline.md`, `09-backtesting-evaluation.md`

**P0-4: Quant-Score durch Gate-Kaskade ersetzt**
- Kein gewichteter `quant_score` mehr — stattdessen harte Ja/Nein-Gates pro Indikator
- 7 Gates: Spread, Volume, RSI, EMA-Trend, VWAP, ATR, ADX
- Hard-Veto (kein Override) vs. Soft-Gate (Schwelle durch Regime-Confidence moduliert)
- QuantOutput-Schema: `gates_passed` (boolean) + `failed_gates` (Set von Enums) statt `quant_score` (float)
- Betroffene Dateien: `03-strategy-logic.md`, `08-symmetric-hybrid-protocol.md`

#### P1-Entscheidungen (wichtige Luecken geschlossen)

**P1-1: Kein Universe Ingestion Gate**
- Bewusste Designentscheidung: Instrumente werden vom Operator handverlesen uebergeben
- Kein automatischer Screening-/Validierungsfilter
- Dokumentiert in `00-overview.md`

**P1-2: Partial Fills als bekannte V1-Limitation**
- Im Backtest nicht modelliert (immer vollstaendiger Fill)
- Bei populaeren High-Beta-Aktien zu RTH praktisch irrelevant
- Im Live-Monitoring beobachten
- Dokumentiert in `07-oms-execution.md` und `09-backtesting-evaluation.md`

**P1-3: Crash-Recovery als Zielbild, niedrige Prio**
- V1-Schutz: GTC-Stops beim Broker + Operator sitzt davor
- Zielbild (spaetere Entwicklungsstufe): Externer Watchdog + Operator-Alerting (hoechste Prio) + optionaler Position-Management-Only Recovery Mode
- Alerting bei Prozess-Tod hat hoehere Prio als automatischer Restart
- Dokumentiert in `11-edge-cases.md` (V1 vs. Zielbild getrennt) und `10-observability.md`

**P1-4: Incomplete-Bar-Policy als harte Regel**
- Bei 1m-Decision-Interrupt: Letzter abgeschlossener 3m/10m-State wird eingefroren
- Nur dedizierte 1m-Features der triggernden Bar werden frisch berechnet
- Keine Neuberechnung von EMAs/Baendern auf unfertigen Bars
- Harte Anforderung fuer Backtest-Fidelity
- Dokumentiert in `01-data-pipeline.md`

**P1-5: Post-Fill Exit Split fuer Bracket-Orders**
- Ein einziger Entry (eine Order)
- Nach Fill: OMS splittet in separate OCA-Exit-Gruppen pro Tranche
- OCA-Gruppe je mit Target (Limit) + Stop (Stop-Market)
- Bei Target-Fill storniert IB serverseitig nur den zugehoerigen Stop — keine Race Condition
- Dokumentiert in `07-oms-execution.md`

**P1-6: V1 nur US-Markt**
- V1 fokussiert ausschliesslich auf US-Maerkte (NYSE, NASDAQ)
- EU-Maerkte als spaetere Erweiterung vorgesehen, nicht Teil der V1-Spezifikation
- Begruendung: Kostenstruktur macht US-Intraday fuer Privatpersonen deutlich attraktiver
- Dokumentiert in `00-overview.md`

#### Neue Konzepterweiterung

**Pre-Market Instrument Profiling**
- In WARMUP-Phase analysiert das LLM die historischen Intraday-Verlaeufe des Instruments
- 5 Dimensionen: Gap-Verhalten, Recovery-Tendenz, Reversal-Zeitfenster, Volatilitaetsprofil, Trendkontinuitaet
- Output: Strukturiertes Instrument-Profil mit Enums + Begruendung
- Deterministische Validierung durch QuantEngine: Bei Widerspruch gilt quantitative Evidenz
- Profil fliesst als statischer Kontext in alle Intraday-LLM-Calls ein
- Dokumentiert in `04-llm-integration.md` (Abschnitt 8)

### 9. Verifikation der Einarbeitung (2026-02-21)

4 parallele Verifikations-Agents haben alle 12 Konzeptdateien geprueft:

| Entscheidung | Status | Anmerkung |
|-------------|--------|-----------|
| P0-1 | ✅ Vollstaendig | 2 verwaiste `entry_price_zone`-Referenzen in 11-edge-cases.md nachkorrigiert |
| P0-2 | ✅ Vollstaendig | 2 "Round-Trip"-Referenzen in 06-risk-management.md nachkorrigiert |
| P0-3 | ✅ Vollstaendig | Keine Inkonsistenzen |
| P0-4 | ✅ Vollstaendig | Keine `quant_score`-Referenzen mehr |
| P1-1 bis P1-6 | ✅ Alle vorhanden | Korrekt dokumentiert |
| Pre-Market Profiling | ✅ Vorhanden | Vollstaendig mit Schema und Validierung |

---

## Phase 2: User Stories und Implementierung (2026-02-21)

### 10. User-Story-Erstellung

Das konsolidierte Konzept (12 Dateien) wurde in **50 User Stories** zerlegt, verteilt ueber 10 Maven-Module und 7 Execution Waves.

**Pfad:** `T:\codebase\its_odin\temp\userstories\`

| Modul | Stories | IDs |
|-------|---------|-----|
| odin-api (Foundation) | 6 | ODIN-001 bis ODIN-006 |
| odin-data | 3 | ODIN-007 bis ODIN-009 |
| odin-brain | 10 | ODIN-010 bis ODIN-019 |
| odin-execution | 5 | ODIN-020 bis ODIN-024 |
| odin-core | 6 | ODIN-025 bis ODIN-030 |
| odin-audit | 2 | ODIN-031, ODIN-032 |
| odin-backtest | 4 | ODIN-033 bis ODIN-036 |
| odin-app | 3 | ODIN-037 bis ODIN-039 |
| odin-frontend | 4 | ODIN-040 bis ODIN-043 |
| odin-persistence | 2 | ODIN-044, ODIN-045 |
| **Phase 2** | 5 | ODIN-046 bis ODIN-050 |

**Groessenverteilung:** 14 S, 21 M, 12 L, 3 XL
**Kritischer Pfad:** ODIN-001 → ODIN-002 → ODIN-017 → ODIN-019 (~12.5 Tage)
**Execution Plan:** `T:\codebase\its_odin\temp\userstories\EXECUTION-PLAN.md`

### 11. User-Story-Spezifikation im Wiki

Eine verbindliche, generische Spezifikation fuer alle User Stories wurde erstellt und im Wiki verankert:

**Pfad:** `docs/meta/user-story-specification.md`

Enthaelt:
- Story-Struktur und Namenskonventionen
- Vollstaendige Definition of Done (8 Bereiche: 2.1 Code-Qualitaet bis 2.8 Abschluss)
- Drei-dimensionales Gemini-Review (Code-Bugs, Konzepttreue, Praxis-Gaps)
- ChatGPT-Test-Sparring-Pflicht (Grenzfaelle, Edge Cases)
- Integrationstests mit realen Implementierungen (nicht alles gemockt)
- Embedded Postgres/Zonky fuer DB-Stories
- Live-aktualisierte Protokolldatei (protocol.md)
- **Agent-Execution-Regeln** (Abschnitt 7 — siehe unten)

### 12. Erste Implementierungswelle (Wave 0 + Wave 1 Teilmenge)

**Umgesetzt:** 7 Stories durch 3 parallele Agents

| Agent | Stories | Ergebnis |
|-------|---------|----------|
| Agent 1 | ODIN-001, 003, 004, 005, 006 (5× odin-api) | Code + 71 Unit-Tests committed |
| Agent 2 | ODIN-044, 045 (2× odin-persistence) | Code + 32 Unit-Tests + 20 Integrationstests (Zonky) committed |
| Agent 3 | ODIN-008 (1× odin-data) | Code + 38 Unit-Tests + 4 Integrationstests committed |

**Alle 7 Stories committed und gepusht** auf `its-odin-backend/main`.

### 13. QS-Gate: Alle 7 Stories durchgefallen

Nach der Implementierung wurden **dedizierte QS-Agents pro Story** eingesetzt. Ergebnis: **Alle 7 Stories NICHT BESTANDEN.**

#### Systematische Fehler (alle Stories betroffen)

| Problem | Betroffene Stories |
|---------|-------------------|
| **ChatGPT-Sparring nicht durchgefuehrt** | Alle 7 |
| **Integrationstests fehlen oder nicht im Build** | 001, 003, 004, 005, 006 (komplett fehlend), 008 (Failsafe-Plugin fehlt) |
| **Gemini-Review unvollstaendig** (Dimension 3 fehlt) | 001, 003, 004, 005, 006 |

#### Story-spezifische Findings

| Story | Finding | Schwere |
|-------|---------|---------|
| ODIN-004 | `DEGRADED_EXECUTION` als 5. Mode fehlt (Konzept 11-edge-cases.md Abschnitt 6.1) | Bug |
| ODIN-005 | `runId`-Nullable-Entscheidung und `MonitorEventListener`-Port-Entscheidung fehlen | Dokumentation |
| ODIN-006 | `totalCyclesCompleted` im GlobalRiskManager wird nie inkrementiert | Bug |
| ODIN-008 | Power-Hour Default 60min statt 30min laut Konzept (03-strategy-logic.md: 15:15-15:45) | Konzepttreue |
| ODIN-008 | Failsafe-Plugin fehlt in odin-data pom.xml (Integrationstests laufen nicht im Build) | Build |
| ODIN-045 | `HASH_HEX_LENGTH` Konstante steht zwischen Instanzfeldern statt am Klassenanfang | Minor |

#### Qualitaetsranking der Agents

1. **Agent 2** (Persistence, 2 Stories) — Beste Qualitaet: Nur ChatGPT fehlte, Integrationstests + Zonky + Gemini 3D alles vorhanden
2. **Agent 3** (Session Boundaries, 1 Story) — Gut, aber Failsafe vergessen + Konzept-Abweichung
3. **Agent 1** (API Models, 5 Stories) — Code und Unit-Tests gut, aber 3 Prozessschritte systematisch uebersprungen

### 14. Lesson Learned: Context-Compaction zerstoert DoD-Compliance

**Ursache:** Agent 1 hatte 5 Stories in einem Kontext. Mit jeder Aktion wuchs der Kontext, Context-Compaction loeschte Teile der Instruktion, und die "weichen" DoD-Schritte (Integrationstests, ChatGPT-Sparring, Gemini Dim 3) wurden systematisch uebersprungen.

**Neue verbindliche Regeln** (im Wiki verankert unter `docs/meta/user-story-specification.md`, Abschnitt 7):

| Regel | Beschreibung |
|-------|-------------|
| **7.1 Eine Story = Ein Agent** | Nie mehrere Stories in einem Agent buendeln. Jeder Agent erhaelt frischen Kontext mit vollstaendiger Instruktion |
| **7.2 Fail-Fast** | Wenn ein DoD-Schritt blockiert ist (z.B. ChatGPT nicht erreichbar): sofort abbrechen, Fehler melden — nicht ueberspringen |
| **7.3 QS-Gate** | Nach jedem Implementierungs-Agent ein dedizierter QS-Agent pro Story |
| **7.4 Prompt-Struktur** | 6 Pflichtbestandteile: story.md, Spezifikation, CLAUDE.md, Konzeptdateien, Guardrails, explizite DoD-Anweisung |
| **7.5 Parallelisierung** | Verschiedene Module parallel, gleiches Modul sequenziell oder Worktree |

### 15. ChatGPT-Pool: Funktionstest bestanden

Der ChatGPT-Pool wurde mit 12 zufaelligen Java-Dateien aus odin-api getestet. Ergebnis:
- Pool funktioniert (Acquire → Send → Release)
- Antwortzeit: ~140 Sekunden fuer 12 Dateien
- ChatGPT lieferte ausfuehrliches Review (Code-Qualitaet, Immutability, Contracts, fachliche Auffaelligkeiten)
- **Die Agents haetten ChatGPT nutzen koennen — sie haben es nur nicht getan**

---

## Aktueller Working State

### Naechste Schritte

**Schritt 1: Nacharbeit der 7 bestehenden Stories** (Remediation)

Fuer jede der 7 Stories wird ein **neuer, dedizierter Agent** gestartet, der:
1. Die bestehende Implementierung vorfindet (Code ist committed)
2. Den QS-Bericht (`qa-report.md`) als Maengelliste liest
3. Die fehlenden DoD-Schritte nachholt:
   - ChatGPT-Sparring fuer Test-Edge-Cases einholen und umsetzen
   - Fehlende Integrationstests schreiben
   - Fehlende Gemini-Review-Dimensionen nachholen
   - Protocol.md vervollstaendigen
4. Story-spezifische Bugs fixt (ODIN-004, 005, 006, 008, 045)
5. Erneut committet und pusht

**Schritt 2: Weitere Stories aus Wave 0 + Wave 1** (neue Implementierung)

Nach erfolgreicher Remediation werden die naechsten Stories umgesetzt — strikt ein Agent pro Story:
- ODIN-002 (LLM Tactical Schema, abhaengig von ODIN-001) — API-Foundation
- ODIN-009 (Pattern Feature Recognition) — odin-data, keine Abhaengigkeiten
- ODIN-015 (Exhaustion Detection) — odin-brain, keine Abhaengigkeiten
- ODIN-020 (Repricing Policy) — odin-execution, keine Abhaengigkeiten
- Weitere Wave-1-Stories ohne Abhaengigkeiten

**Schritt 3: QS-Gate pro Story, dann naechste Wave**

Jede neue Story durchlaeuft den vollstaendigen Zyklus: Implementierung → QS-Pruefung → ggf. Nacharbeit → Abnahme.

### Dateistruktur

```
docs/concept/
├── 00-overview.md              ← Systemuebersicht, Scope, Constraints, Begriffshierarchie, US-only V1
├── 01-data-pipeline.md         ← Datenstufen (OHLCV/BASIC/L2), Feature-Matrix, Incomplete-Bar-Policy
├── 02-regime-detection.md      ← 5 Regime, Subregimes, Fusion, FusionLab, Cold-Start
├── 03-strategy-logic.md        ← FSM, Decision Loop, 4 Setups, Gate-Kaskade, Multi-Cycle-Day
├── 04-llm-integration.md       ← Tactical Controller (Enums, keine Preise), Instrument-Profiling, Anti-Halluzination
├── 05-stops-profit-protection.md ← ATR-Freeze, R-Protection, Trailing, Exhaustion
├── 06-risk-management.md       ← 3 Verteidigungslinien, Position Sizing, Kill-Switch
├── 07-oms-execution.md         ← Entry-Preis via Struktur-Levels, Post-Fill Exit Split, OCA-Gruppen
├── 08-symmetric-hybrid-protocol.md ← Dual-Key, Arbiter, Gate-Kaskade statt Score, JSON-Schemata
├── 09-backtesting-evaluation.md ← Evaluationsdesign, Datenstufen-Tagging, Datenqualitaet, Gate-Kaskade
├── 10-observability.md         ← Monitoring, Audit, Dashboard, SLOs, Prozess-Tod-Alerting
├── 11-edge-cases.md            ← 34 Edge Cases, Degradation Modes, Crash-Recovery (V1 vs. Zielbild)
├── WORKING-STATE.md            ← Diese Datei
└── archiv/                     ← Alle 10 Quelldokumente + beide Konsolidierungsversionen

docs/meta/
├── user-story-specification.md ← Verbindliche DoD + Agent-Execution-Regeln (Abschnitt 7)
├── playbook.md                 ← Architekturkonzept-Playbook
├── working-state.md            ← Meta Working State
└── concept-changelog.md        ← Aenderungslog

temp/userstories/
├── EXECUTION-PLAN.md           ← 50 Stories, 7 Waves, Abhaengigkeitsgraph
├── ODIN-001_api-subregime-model/ bis ODIN-050_PHASE2-advanced-crash-recovery/
│   ├── story.md                ← Story-Definition
│   ├── protocol.md             ← Protokolldatei (ggf. noch unvollstaendig)
│   └── qa-report.md            ← QS-Bericht (fuer bereits gepruefte Stories)
```

### Git-Status

**Wiki-Repo** (`its-odin-wiki`):
- Konzeptkonsolidierung: `237c4b4` bis `3d3db4e`
- User-Story-Spezifikation: `0ce5db3`
- Agent-Execution-Regeln: `171f936`

**Backend-Repo** (`its-odin-backend`):
- Wave 0 odin-api (ODIN-001,003,004,005,006): `d225a82`
- Wave 0 persistence (ODIN-044,045): committed (nach d225a82)
- Wave 1 session boundaries (ODIN-008): committed (nach persistence)
- **Status:** Code ist committed aber QS nicht bestanden — Nacharbeit steht aus

### Offene Punkte

1. **Remediation der 7 durchgefallenen Stories** — ChatGPT-Sparring, Integrationstests, Gemini Dim 3, story-spezifische Bugs
2. **Wave 1 + Wave 2 Stories** — Umsetzung gemaess Execution Plan, strikt ein Agent pro Story
3. **ChatGPT-Pool Timing** — Erster Versuch lieferte leere Antwort (Pool antwortete bevor ChatGPT fertig war). Zweiter Versuch funktionierte. Ggf. Timeout-Konfiguration des Pool-Service pruefen

### Stakeholder-Entscheidungen (bindend, muessen erhalten bleiben)

- ATR-Freeze bei Entry, Faktor 2.2x (ersetzt alte 1.5x)
- Trailing Stop = Highwater-Mark (darf nur steigen)
- KPI primaer, LLM sekundaer bei Regime Detection (empirisch zu validieren)
- ODIN ist KEIN reines Algo-System — LLM MUSS Entscheidungsgewalt haben
- L2 nur fuer Live-Execution, nicht fuer Alpha/Regime/Setups
- Optimierungen muessen allgemeingueltig sein (kein Overfitting)
- Runner-Trail bleibt, Profit-Protection als Floor
- TRAIL_ONLY als OMS-Override
- Alte Backtests loeschen (keine Event-Logs, unbrauchbar)
- LLM liefert keine Preise — OMS bestimmt Entry-Preis deterministisch aus Struktur-Levels
- Trade/Cycle/Tranche-Hierarchie: Aufstocken = Tranche, nicht Cycle
- Datenverfuegbarkeit konfigurationsabhaengig (OHLCV_ONLY / BASIC_QUOTES / L2_DEPTH)
- Gate-Kaskade statt gewichtetem Quant-Score
- V1 nur US-Markt (NYSE, NASDAQ)
- Pre-Market Instrument Profiling durch LLM in WARMUP-Phase
- Instrument-Selektion extern durch Operator (kein Universe Ingestion Gate)
- **Neu:** Eine Story = Ein Agent (Context-Compaction-Regel)
- **Neu:** Fail-Fast bei blockierten DoD-Schritten
- **Neu:** QS-Gate nach jedem Implementierungs-Agent

### Wichtige Architektur-Referenzen

- Backend: `T:\codebase\its_odin\its-odin-backend` (10 Maven-Module, Java 21, Spring Boot)
- Frontend: `T:\codebase\its_odin\its-odin-ui` (React, TypeScript, Vite)
- Wiki: `T:\codebase\its_odin\its-odin-wiki` (MkDocs Material)
- CLAUDE.md: `T:\codebase\its_odin\CLAUDE.md` (vollstaendige Projekt-Konventionen)
- User-Story-Spezifikation: `T:\codebase\its_odin\its-odin-wiki\docs\meta\user-story-specification.md`
- Execution Plan: `T:\codebase\its_odin\temp\userstories\EXECUTION-PLAN.md`
