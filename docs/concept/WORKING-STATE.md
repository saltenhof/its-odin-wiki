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

## Aktueller Working State

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
```

### Git-Status

- Letzter Commit der Konsolidierung: `237c4b4`
- Review-Einarbeitung: `dfd22ac`, `ff01709`, `59da165`, `3d3db4e` (4 parallele Agents)
- Nachkorrekturen (verwaiste Referenzen): ausstehend (wird mit diesem WORKING-STATE committed)
- Wiki-Repo: `T:\codebase\its_odin\its-odin-wiki`
- mkdocs build erfolgreich

### Offene Punkte

**Keine offenen Punkte.** Alle P0- und P1-Findings aus dem Gemini/ChatGPT-Review sind eingearbeitet und verifiziert.

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
- **Neu:** LLM liefert keine Preise — OMS bestimmt Entry-Preis deterministisch aus Struktur-Levels
- **Neu:** Trade/Cycle/Tranche-Hierarchie: Aufstocken = Tranche, nicht Cycle
- **Neu:** Datenverfuegbarkeit konfigurationsabhaengig (OHLCV_ONLY / BASIC_QUOTES / L2_DEPTH)
- **Neu:** Gate-Kaskade statt gewichtetem Quant-Score
- **Neu:** V1 nur US-Markt (NYSE, NASDAQ)
- **Neu:** Pre-Market Instrument Profiling durch LLM in WARMUP-Phase
- **Neu:** Instrument-Selektion extern durch Operator (kein Universe Ingestion Gate)

### Wichtige Architektur-Referenzen

- Backend: `T:\codebase\its_odin\its-odin-backend` (10 Maven-Module, Java 21, Spring Boot)
- Frontend: `T:\codebase\its_odin\its-odin-ui` (React, TypeScript, Vite)
- Wiki: `T:\codebase\its_odin\its-odin-wiki` (MkDocs Material)
- CLAUDE.md: `T:\codebase\its_odin\CLAUDE.md` (vollstaendige Projekt-Konventionen)
