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

Alle nachfolgenden Abschnitte von 7 bis 19 durchnummeriert (vorher 6-18).

---

## Aktueller Working State

### Dateistruktur

```
docs/concept/
├── 00-overview.md              ← Systemuebersicht, Scope, Constraints, Designphilosophie
├── 01-data-pipeline.md         ← OHLCV-only, L2-Kompromiss, Timeframes, DQ-Gates
├── 02-regime-detection.md      ← 5 Regime, Subregimes, Fusion, FusionLab, Cold-Start
├── 03-strategy-logic.md        ← FSM, Decision Loop, 4 Setups, Multi-Cycle-Day
├── 04-llm-integration.md       ← Tactical Parameter Controller, Schema, Anti-Halluzination
├── 05-stops-profit-protection.md ← ATR-Freeze, R-Protection, Trailing, Exhaustion
├── 06-risk-management.md       ← 3 Verteidigungslinien, Position Sizing, Kill-Switch
├── 07-oms-execution.md         ← Order State Machine, Bracket, Multi-Tranchen
├── 08-symmetric-hybrid-protocol.md ← Dual-Key, Arbiter, JSON-Schemata
├── 09-backtesting-evaluation.md ← Evaluationsdesign, Datenqualitaet, Challenger-Suite, Gate-Kaskade, DB-Schema
├── 10-observability.md         ← Monitoring, Audit, Dashboard, SLOs
├── 11-edge-cases.md            ← 34 Edge Cases, Degradation Modes
├── WORKING-STATE.md            ← Diese Datei
└── archiv/                     ← Alle 10 Quelldokumente + beide Konsolidierungsversionen
```

### Git-Status

- Letzter Commit der Konsolidierung: `237c4b4` — Finale Konsolidierung mit ChatGPT-Ergaenzungen
- Nachtrag Datenqualitaet: noch nicht committed
- Wiki-Repo: `T:\codebase\its_odin\its-odin-wiki`
- mkdocs.yml ist aktualisiert, `mkdocs build` erfolgreich

### Was als Naechstes ansteht

**Review-Schleife:** Der User will die 12 konsolidierten Dateien reviewen lassen (vermutlich mit neuen MCP-Plugins). Ziel ist sicherzustellen, dass kein Informationsverlust aufgetreten ist und die Qualitaet stimmt.

### Offene Punkte (Stichprobe geprueft 2026-02-21, verifiziert 2026-02-21)

**Alle erledigt:**

- ~~Instrument-Selektion~~ → `00-overview.md` (Instrumentauswahl) + `03-strategy-logic.md` (Abschnitt 11)
- ~~FX-Hedging~~ → `06-risk-management.md` (FX-Conversion) + `05-stops-profit-protection.md` (EUR/USD-Spot) + `09-backtesting-evaluation.md` (FX-Kosten)
- ~~Multi-Instrument-Korrelation~~ → `00-overview.md` bewusst als Out of Scope deklariert
- ~~Backtesting-Datenqualitaet~~ → `09-backtesting-evaluation.md` Abschnitt 6 (nachgetragen)
- ~~Quant-Score-Berechnungsformel~~ → `09-backtesting-evaluation.md` Abschnitt 7.3: Geometric Score mit λ=1.0, μ=2.0 explizit definiert
- ~~Gate-0-Schwellen~~ → `09-backtesting-evaluation.md` Abschnitt 13: MDD ×1.10, ES95 −0.0005, TailLossFreq +20% quantifiziert
- ~~Loss-Cost-Matrix~~ → `08-symmetric-hybrid-protocol.md` Abschnitt 8.3 + `02-regime-detection.md` Abschnitt 6.4: 5 Kernzellen definiert, Rest intentional offen fuer Walk-Forward-Kalibrierung
- ~~FusionLab Gate-2-Schwelle~~ → `02-regime-detection.md` Abschnitt 10-11: CI-Breite < 0.15 als Proxy, Quantifizierung ueber Walk-Forward-Plateau-Stabilisierung geplant

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

### Wichtige Architektur-Referenzen

- Backend: `T:\codebase\its_odin\its-odin-backend` (10 Maven-Module, Java 21, Spring Boot)
- Frontend: `T:\codebase\its_odin\its-odin-ui` (React, TypeScript, Vite)
- Wiki: `T:\codebase\its_odin\its-odin-wiki` (MkDocs Material)
- CLAUDE.md: `T:\codebase\its_odin\CLAUDE.md` (vollstaendige Projekt-Konventionen)
