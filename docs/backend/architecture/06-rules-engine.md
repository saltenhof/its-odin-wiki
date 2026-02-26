# ODIN — Kapitel 6: Rules Engine & Decision Loop

Version: 0.3 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2 — Loop-State-Scoping (P0.1), LLM-als-Gate (P0.2), Intent-vs-Maintenance (P0.3), Reject-Ownership (P0.4), Config-Keys (P1.2), vetoActive-Quelle (P1.3), targetPrice-Sicherung (P1.4), Regime-Rangfolge-Begruendung (P1.5), LLM-Refresh-Ownership (P1.6), Freshness-Definition (P1.7)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt die deterministische Rules Engine und den Decision Loop — das Herzstueck der Entscheidungslogik von ODIN. Die Rules Engine kombiniert quantitative Indikatoren mit LLM-Features zu deterministischen Entry/Exit-Entscheidungen. Der Decision Loop orchestriert den gesamten Entscheidungszyklus pro Decision-Bar.

**Modulzuordnung:** `odin-brain` (Package: `de.odin.brain.rules`, `de.odin.brain.arbiter`)

**Zentrale Abhaengigkeiten:**
- `MarketSnapshot` (von odin-data) — Immutable Snapshot als Entscheidungsgrundlage (Kap 2)
- `IndicatorResult` (von KPI-Engine, Kap 4) — Alle technischen Indikatoren
- `LlmAnalysis` (von LlmAnalysisStore, Kap 5) — Letzter validierter LLM-Output (asynchron bereitgestellt)
- `MarketClock` (Port aus odin-api) — Sole Time Source, TTL-Berechnung
- `EventLog` (Port aus odin-api) — Persistierung aller Entscheidungen (non-droppable)
- `RunContext` (aus odin-api) — runId, mode, tradingDate

---

## 2. Designprinzipien

| Prinzip | Bedeutung |
|---------|----------|
| Determinismus | Gleicher Input → gleiches Ergebnis. Keine Zufallskomponente. Alle Zeitvergleiche gegen MarketClock |
| Testbarkeit | Jede Regel als Unit-Test formulierbar. Vollstaendige Inputs (IndicatorResult + LlmAnalysis + PipelineState) als Records injizierbar |
| Nachvollziehbarkeit | Jede Entscheidung mit vollstaendigem Kontext im EventLog persistiert (Kap 0, §10). Kein separater Audit-Trail — das EventLog ist die Single Source of Truth |
| Konfigurierbarkeit | Alle Schwellenwerte externalisiert (Properties), nicht hartcodiert |

> **Eiserne Regel (Analyst-Prinzip, Kap 0 §3.4, Kap 5 §2):** Das LLM liefert ausschliesslich strukturierte Features — nie Handlungsanweisungen. Kein LLM-Output darf direkt einen Entry oder Exit triggern. Die Rules Engine nutzt LLM-Features (regime, regimeConfidence, patternCandidates) als **Input-Signal** neben KPI-Werten, trifft aber die Entscheidung deterministisch.

---

## 3. Decision Loop (pro Decision-Cycle)

### Trigger

Der primaere Trigger fuer den Decision Loop ist der **Bar-Close** eines Decision-Bars (3m oder 5m, parametrisierbar via `odin.data.decision-bar-timeframe-s`; Kap 0, §6; Kap 2; Kap 4). Ticks und 1m-Bar-Closes aktualisieren den Buffer und Trailing-Stops, loesen aber keinen vollstaendigen Decision-Cycle aus. Der 1m-Event-Detektor emittiert bei extremen Moves (> 6% Spike) ein `MonitorEvent` — dieser triggert keinen Decision-Cycle, kann aber einen asynchronen LLM-Refresh ausloesen.

### Bar-Close-Barrier (Determinismus-Garantie)

Im Sim-Modus (Decision-synced Pace) gibt der SimulationRunner das naechste Event erst frei, wenn der aktuelle Decision-Cycle vollstaendig abgeschlossen ist — Bar-Close-Barrier (Kap 0, §3.3). Im Live-Modus ist die Barrier implizit durch die single-threaded Ausfuehrung pro Pipeline gewaehrleistet.

### Threading-Modell

Der Decision Loop laeuft **single-threaded pro Pipeline** (Kap 0, §6). Es gibt keine parallele Ausfuehrung von Schritten innerhalb eines Decision-Cycles. Verschiedene Pipelines fuehren ihren Decision Loop unabhaengig und parallel aus.

### Flow

```
Bar-Close Event (Decision-Bar: 3m oder 5m)
  │
  ├── Voraussetzung: Bar-Close-Barrier (vorheriger Cycle abgeschlossen)
  │
  1. Hard-Stop-Pruefung
  │  → Tagesverlust >= Hard-Stop-Limit? → DAY_STOPPED (sofort, alle Orders cancelled)
  │  → Max Trades/Tag erreicht? → Keine neuen Entries (Exits weiterhin moeglich)
  │  → Cooling-Off aktiv? → Entry blockiert (Exits weiterhin moeglich)
  │
  2. MarketSnapshot erzeugen (odin-data, immutable, MarketClock-Timestamp)
  │
  3. KPI-Engine: Indikatoren berechnen → IndicatorResult
  │  (RSI, ATR, EMA, ADX, Bollinger, VWAP aus Snapshot — verfuegbar fuer ALLE nachfolgenden Schritte)
  │  Warmup-Guard: warmupComplete-Flag pruefen (siehe §3.1)
  │
  4. LlmAnalysis aus LlmAnalysisStore lesen (AtomicReference, pro Pipeline)
  │  → Freshness pruefen: barCloseTime - availableFromMarketTime < entryFreshnessMax?
  │    (barCloseTime = MarketClock-Zeit des aktuellen Bar-Close, nicht MarketClock.now())
  │  → Stale oder nicht vorhanden → Default: regime = UNCERTAIN, regimeConfidence = 0.0
  │  (LLM-Call selbst laeuft ASYNCHRON im Hintergrund — kein Blocking, Kap 5 §8)
  │  → Store-Read-Fehler (sollte bei AtomicReference nicht auftreten) → wie LLM_STALE behandeln
  │
  5. Regime-Bestimmung (KPI primaer, LLM sekundaer — siehe §4)
  │
  6. Rules Engine: Entry/Exit-Bedingungen pruefen
  │  (nutzt IndicatorResult + LlmAnalysis-Features + PipelineState + Regime)
  │  → Ein-Intent-pro-Bar: Maximal ein TradeIntent pro Decision-Cycle (§3.2)
  │  → Tie-Break: Exit > Entry (Kap 4, §5)
  │
  7. Quant Validation: Gesamtscore + Hard-Veto (Kap 4)
  │  (finaler Gate nach Rules, prueft Gesamtbild)
  │
  8. Decision Arbiter: Finale Entscheidung (§7)
  │  (kombiniert Rules-Intent + Quant-Score + LlmAnalysis-Metadaten)
  │
  9. Risk Gate (odin-execution): Position Sizing, Limits, R/R-Pruefung
  │  → Exposure > Global-Limit? → Reject
  │  → R/R < Minimum? → Reject
  │  → Budget erschoepft? → Reject (final fuer heute)
  │
  10. OMS: Order erzeugen oder Intent verwerfen
  │   EventLog-Eintrag (non-droppable): Entscheidung + vollstaendiger Kontext
  │
  └── Bar-Close-Barrier freigeben (naechster Cycle darf starten)
```

### 3.1 Warmup-Guard

Der Decision-Cycle laeuft in allen **Loop-aktiven States** (WARMUP, OBSERVING, POSITIONED, PENDING_FILL) — auch waehrend der Warmup-Phase (Kap 4, §6). In nicht-Loop-aktiven States (INITIALIZING, DAY_STOPPED, FORCED_CLOSE, EOD) laeuft kein Decision-Cycle (siehe §9). Die `warmupComplete`-Flag aus dem `IndicatorResult` wirkt als **Guard in der Rules Engine**:

| warmupComplete | Auswirkung |
|---------------|------------|
| `false` | Alle Entry-Regeln blockiert. Exit-Regeln aktiv (Stop-Loss, Forced-Close). Pattern-State-Machines pausiert |
| `true` | Normaler Regelbetrieb |

> **Kein separater Warmup-Modus.** In Loop-aktiven States durchlaeuft der Decision-Cycle alle Schritte — die Rules Engine filtert bei `warmupComplete = false` lediglich Entry-Intents heraus. Das verhindert Sonderpfade und ist in der Simulation deterministisch reproduzierbar.

### 3.2 Ein-Intent-pro-Bar-Garantie

Pro Decision-Cycle wird **maximal ein TradeIntent pro Pipeline** (= pro Instrument) erzeugt (Kap 4, §5). Ein `TradeIntent` ist eine **Entry- oder Exit-Entscheidung** — eine High-Level-Handelsabsicht, die an das Risk Gate weitergeleitet wird.

**Abgrenzung zu OMS-Maintenance:** Trailing-Stop-Anpassungen, Scaling-Out-Tranchen-Berechnung und Repricing offener Orders sind **OMS-interne Operationen** (Kap 7), keine TradeIntents. Sie zaehlen nicht gegen das Ein-Intent-pro-Bar-Limit und laufen im OMS unabhaengig vom Decision Loop. Trailing-Stops werden sogar auf Tick-Ebene nachgefuehrt (nicht nur bei Bar-Close).

**Tie-Break-Regel bei Kollision** (Exit-Signal und Entry-Signal im selben Cycle): Exit > Entry. Begruendung: Kapitalerhalt hat Vorrang. Der Exit wird ausgefuehrt, ein potentieller Entry wird im naechsten Cycle neu evaluiert.

---

## 4. Regime-Bestimmung

### Primaer: KPI-basiert (deterministisch)

Die Rules Engine bestimmt das Regime **primaer** anhand von KPI-Werten aus dem `IndicatorResult`:

| Regime | KPI-Kriterien (Schwellenwerte konfigurierbar, §13) |
|--------|--------------|
| TREND_UP | EMA(9) > EMA(21) AND Preis > VWAP AND ADX > `adx-trend-threshold` |
| TREND_DOWN | EMA(9) < EMA(21) AND Preis < VWAP AND ADX > `adx-trend-threshold` |
| RANGE_BOUND | ADX < `adx-trend-threshold` AND Preis innerhalb VWAP ± `range-vwap-band-atr-factor` x ATR |
| HIGH_VOLATILITY | ATR(aktuell) > `high-volatility-atr-multiplier` x ATR(20-Bar-SMA) |
| UNCERTAIN | Keines der obigen Kriterien klar erfuellt |

### Sekundaer: LLM-Bestaetigung

Das LLM liefert `regime` + `regimeConfidence` im `LlmAnalysis`-Record (Kap 5, §5). Die Rules Engine nutzt das als **Bestaetigung oder Widerspruch**:

| Situation | Ergebnis |
|-----------|---------|
| KPI-Regime == LLM-Regime | KPI-Regime gilt (bestaetigt) |
| KPI-Regime != LLM-Regime, LLM-Confidence >= 0.5 | **Konservativeres Regime** gilt (z.B. KPI = TREND_UP, LLM = RANGE_BOUND → RANGE_BOUND) |
| KPI-Regime != LLM-Regime, LLM-Confidence < 0.5 | KPI-Regime gilt allein (LLM zu unsicher) |
| Kein LLM-Output oder stale | KPI-Regime gilt allein |

> **"Konservativer" bedeutet:** Weniger Entry-freundlich im Kontext eines **Long-Only-Systems** (v1). Rangfolge (konservativster zuerst): UNCERTAIN > TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > TREND_UP. Begruendung: UNCERTAIN und TREND_DOWN blockieren Long-Entries vollstaendig; HIGH_VOLATILITY erlaubt Entries nur unter strengen Auflagen; RANGE_BOUND erlaubt limitierte Entries; TREND_UP ist der Entry-freundlichste Zustand.

### Regime-Wechsel-Erkennung

1. KPI-Indikatoren zeigen Richtungswechsel (z.B. EMA-Kreuzung, VWAP-Durchbruch)
2. **Confirmation Lag (Regime-Hysterese):** Erst nach **zwei aufeinanderfolgenden Decision-Bar-Closes** (3m oder 5m) mit konsistentem neuen Regime. Die fruehere 10m-Confirmation-Loop entfaellt — die Hysterese wird vollstaendig ueber konsekutive Decision-Bars abgebildet
3. Bei LLM-Widerspruch: konservatives Regime (wie oben)
4. Event-Trigger (VWAP-Durchbruch, Volumen-Spike) → Rules Engine emittiert einen **LLM-Refresh-Request** an den `LlmAnalystOrchestrator` (fire-and-forget, async). Der Orchestrator entscheidet ob ein Call abgesetzt wird (Single-Flight, Kap 5 §7). Der Request traegt `runId` + `barCloseTime` als Correlation
5. Bei bestaetigtem Wechsel: Pruefung ob bestehende Position zum neuen Regime passt

---

## 5. Entry-Regeln pro Regime

### Regime-Confidence-Gate (vor Entry-Pruefung)

Vor jeder Entry-Pruefung wird die `regimeConfidence` aus der aktuellen `LlmAnalysis` bewertet (konsistent mit Kap 4, §8 und Kap 5, §8):

| regimeConfidence | Konsequenz |
|-----------------|-----------|
| < 0.5 | Regime gilt als UNCERTAIN → kein Entry (unabhaengig vom Quant-Score) |
| 0.5–0.7 | Entry nur bei quant_score >= 0.70 (erhoehte Schwelle) |
| > 0.7 | Entry bei quant_score >= 0.50 (Standard-Schwelle) |

> **Default bei fehlendem/stale LLM:** `regimeConfidence = 0.0` → Entries blockiert (Kap 5, §8).

### Entry-Bedingungen pro Regime

**LLM als Mandatory Gate (nicht Trigger):** Ein frischer LLM-Output (`regimeConfidence > 0`) ist eine **Vorbedingung** fuer Entries (Policy-Gate). Ohne frischen LLM-Output ist `regimeConfidence = 0.0` (Kap 5, §8) und das Regime-Confidence-Gate blockiert den Entry — unabhaengig von den KPI-Signalen. Das ist kein Widerspruch zum Analyst-Prinzip: das LLM **triggert** keinen Entry (keine Handlungsanweisung), es liefert ein **notwendiges Input-Feature** (regimeConfidence), das die deterministische Rules Engine auswertet.

Alle **Trigger-Bedingungen** nutzen ausschliesslich KPI-Werte (aus `IndicatorResult`) und Marktdaten (aus `MarketSnapshot`). LLM-Kontext-Features (opportunity_zones, entry_price_zone) stehen als **optionale Kontextsignale** zur Verfuegung — sie verschaerfen oder lockern Schwellen, triggern aber nie eigenstaendig:

| Regime | KPI/Markt-Bedingung | LLM-Kontext (optional, verschaerft/lockert) |
|--------|---------------------|----------------------------------------------|
| TREND_UP | EMA(9) > EMA(21) AND RSI < `rsi-max` AND Preis <= VWAP + `vwap-offset-atr-factor` x ATR(14) | Wenn `entry_price_zone` vorhanden: Preis muss zusaetzlich innerhalb Zone liegen |
| TREND_DOWN | **Default: Kein Entry.** Aggressive Mode (Config, Default: OFF): RSI < `aggressive.rsi-threshold` AND Volume > `aggressive.volume-multiplier` x SMA(20) | Halbe Zielgroesse, max `aggressive.max-trades-per-day` Trades/Tag |
| RANGE_BOUND | RSI < `range-rsi-max` AND Preis nahe VWAP-Support (innerhalb `range-vwap-support-atr-factor` x ATR) | Wenn `opportunity_zones` mit type=ENTRY vorhanden: Zone als zusaetzliche Bestaetigung |
| HIGH_VOLATILITY | Volume > `high-vol-volume-multiplier` x SMA(20) AND Spread < Spread-Threshold | Nur bei LLM-Output mit regimeConfidence > `high-vol-regime-confidence-min` |
| UNCERTAIN | **Kein Entry.** Beobachten und warten | — |

> **Abgrenzung zu v0.1:** In v0.1 war "Preis in LLM-Opportunity-Zone" eine direkte Entry-Bedingung. Das ist entfernt — Entry wird ausschliesslich durch KPI/Markt-Signale getriggert. LLM-Zones dienen nur als optionale Bestaetigung (Analyst-Prinzip, Kap 5 §2).

---

## 6. Exit-Regeln

### Prioritaetsordnung

Exit-Trigger werden in **fester Rangfolge** geprueft. Hoeherpriore Trigger uebersteuern niedrigere:

| Prio | Trigger | Bedingung | LLM-Abhaengigkeit | Aktion |
|------|---------|-----------|-------------------|--------|
| 1 | Stop-Loss | Preis <= Entry - Stop-ATR-Factor x ATR(14) | **Keine** (Broker-seitig, GTC-Stop) | Sofortiger Exit. Kein Decision Loop noetig |
| 2 | Forced Close | MarketClock.now() > RTH-Ende - forcedCloseBufferMin | **Keine** (MarketClock-basiert) | Zeitbasierter Exit. EOD-Flat-Constraint |
| 3 | Trailing-Stop | Preis faellt um Trailing-ATR-Factor x ATR vom Intraday-High | **Keine** (ATR-basiert) | Exit. Nachfuehrung auf Tick-Ebene durch OMS (Kap 7), nicht durch Decision Loop |
| 4 | Regime-Wechsel | Neues Regime (2x Bar-Close bestaetigt), Position passt nicht | **Wuenschenswert** | Beschleunigter Exit. Timeout 10s → Quant-Only-Exit |
| 5 | Urgency CRITICAL | LLM urgency = CRITICAL bei offener Position | **Nur als Signal** | Rules Engine re-evaluiert mit IndicatorResult. Exit **nur** wenn Quant-Signale stuetzen (RSI-Reversal, EMA-Kreuzung). CRITICAL allein erzwingt keinen Exit (Analyst-Prinzip) |
| 6 | Zeitbasiert | Halte-Dauer > holdDurationBars (aus Config oder LLM-Kontext) | **Kontext** | Exit |
| 7 | Gewinnmitnahmen | R-basierte Tranchen (Scaling-Out, Kap 7) | **Keine** | Stufenweiser Exit. Tranchen-Berechnung und -Ausfuehrung durch OMS (Kap 7) |

> **Prio 1–3** sind LLM-unabhaengig und funktionieren auch im Quant-Only-Modus (Circuit Breaker aktiv, Kap 5 §9). **Prio 4–6** benoetigen oder bevorzugen LLM-Kontext. **Prio 7** ist rein regelbasiert.

> **Forced Close Timing:** Der Zeitpunkt wird als `RTH-Ende - forcedCloseBufferMin` berechnet, wobei RTH-Ende aus dem Exchange-Kalender der MarketClock stammt (Kap 0, §3.2). Kein hartcodierter Zeitwert.

---

## 7. Pattern-State-Machines

Die Rules Engine verwaltet konkrete Intraday-Setups als Pattern-State-Machines. Jedes Pattern durchlaeuft definierte Zustaende mit expliziten Uebergangsbedingungen.

### Pattern-Aktivierung

Das LLM liefert `patternCandidates` (Enum + Confidence + Phase) im `LlmAnalysis`-Record (Kap 5, §5). Die Rules Engine aktiviert eine Pattern-State-Machine **nur wenn:**
1. Pattern-Confidence >= `odin.rules.pattern.min-confidence` (Default: 0.5)
2. **Deterministische KPI-Feature-Erkennung** (aus IndicatorResult/MarketSnapshot) den Zustandsuebergang **unabhaengig bestaetigt**

> **Analyst-Prinzip:** Das LLM schlaegt Patterns vor (Kontext-Feature), aber die State-Machine-Uebergaenge werden durch KPI-Kriterien getriggert — nicht durch den LLM-Vorschlag allein.

### Setup A: Flush → Reclaim → Run

```
IDLE ──→ FLUSH_DETECTED ──→ RECLAIM ──→ RUN ──→ EXIT ──→ IDLE
              │                  │          │
              └──(invalidiert)───┘──────────┘
```

| State | KPI/Markt-Erkennungskriterien | Uebergang |
|-------|-------------------------------|-----------|
| FLUSH_DETECTED | Bar-Range > flushRangeAtrFactor x ATR(14), Close nahe Low, Sweep unter Key-Level | → RECLAIM wenn Preis innerhalb reclaimMaxBars Bars > VWAP/MA-Zone |
| RECLAIM | Preis ueber VWAP + MA-Cluster, Hold min. 2 Bars | → RUN wenn Higher-Low + EMA(9) > EMA(21). **Starter-Position Entry** |
| RUN | Hoehere Tiefs, Pullbacks flach (< 0.5x ATR), Preis > Trend-MA | → EXIT bei Lower-Low oder Close unter MA-Zone. **Add-Position bei Bestaetigung** |
| EXIT | Strukturbruch oder Trailing-Stop | → IDLE |

**Invalidierungslevel:** Low des Flush. Unterschreitung → Setup sofort ungueltig → IDLE.

### Setup B: Coil → Breakout

```
IDLE ──→ COIL_FORMING ──→ COIL_MATURE ──→ BREAKOUT ──→ RUN ──→ EXIT ──→ IDLE
              │                  │              │          │
              └──(aufgeloest)────┘──────────────┘──────────┘
```

| State | KPI/Markt-Erkennungskriterien | Uebergang |
|-------|-------------------------------|-----------|
| COIL_FORMING | Fallende Hochs + steigende Tiefs, min. coilMinBars Bars, ATR faellt | → COIL_MATURE wenn Kontraktion > coilContractionThreshold |
| COIL_MATURE | Range < 60% Initial-Range, nahe Apex | → BREAKOUT bei Close ausserhalb Begrenzung + Range > breakoutRangeAtrFactor x ATR |
| BREAKOUT | Close oberhalb oberer Trendlinie + Tempo-Bestaetigung | → RUN. Entry bei Breakout oder Retest |
| RUN | Trend etabliert, Pullbacks halten Breakout-Level | → EXIT bei Reentry in Formation |
| EXIT | Rueckkehr in Formation oder Trailing-Stop | → IDLE |

**Fakeout-Schutz:** Am Apex kein Entry — nur bei klarer Bestaetigung (Close ausserhalb + ATR-Range).

### Pattern-Verwaltung

- Mehrere Patterns koennen **parallel ueberwacht** werden (verschiedene State-Machines laufen gleichzeitig)
- Nur ein Pattern darf gleichzeitig eine **Position halten** (Ein-Intent-pro-Bar-Garantie, §3.2)
- Bei warmupComplete = false: Pattern-State-Machines sind **pausiert** (keine Zustandsuebergaenge, §3.1)

### Position Building: Starter + Add (Setup A)

| Phase | Anteil | Zeitpunkt | Stop |
|-------|--------|-----------|------|
| Starter | starterPositionPercent der Zielgroesse | RECLAIM-State, Hold ueber MA/VWAP | Unter Flush-Low |
| Add | addPositionPercent der Zielgroesse | RUN-State bestaetigt (Higher-Low) | Unter letztes Higher-Low |

> Gesamtrisiko (Starter + Add) darf `odin.rules.position.max-risk-percent` pro Trade nicht ueberschreiten. Risk Gate (odin-execution) prueft final.

---

## 8. Decision Arbiter

Der `DecisionArbiter` ist die letzte Instanz vor dem Risk Gate. Er lebt in `de.odin.brain.arbiter` (Kap 1) und wird pro Pipeline instanziiert.

### Inputs

| Input | Quelle | Beschreibung |
|-------|--------|-------------|
| Rules-Intent | RulesEngine | Entry/Exit/NO_ACTION mit Begruendung (Regime, Regel-ID, Pattern-State) |
| QuantScore | QuantValidation (Kap 4) | overallScore, Einzel-Scores, vetoActive (gesetzt von Quant Validation bei Hard-Veto-Bedingung, z.B. NaN in Pflicht-KPI, Kap 4 §10), approved, effectiveThreshold |
| LlmAnalysis | LlmAnalysisStore (Kap 5) | Letzter validierter LLM-Output. Freshness bereits in Schritt 4 des Decision Loop geprueft |
| PipelineState | TradingPipeline (Kap 1) | Aktueller Zustand der Pipeline-FSM |

### Arbiter-Logik

```
if (rules.intent == NO_ACTION) → kein Trade
if (quant.vetoActive) → Trade blockiert (mit RejectReason QUANT_VETO)
if (quant.overallScore < quant.effectiveThreshold) → Trade blockiert (QUANT_SCORE_BELOW_THRESHOLD)
else → TradeIntent erzeugen
```

### TradeIntent-Erzeugung

Der `TradeIntent` (Record in odin-api, Kap 4) wird mit allen relevanten Daten angereichert:

| Feld | Quelle |
|------|--------|
| instrumentId | Pipeline |
| direction | Rules-Intent (LONG, v1: immer LONG) |
| entryPrice | MarketSnapshot.currentPrice (als Referenz, OMS bestimmt tatsaechlichen Order-Preis) |
| stopLevel | Rules-Engine (ATR-basiert, Pattern-spezifisch) |
| targetPrice | Rules-Engine (R-basiert, primaer). LlmAnalysis.targetPrice nur als Vorschlag — wird nur uebernommen wenn Plausibilitaetspruefung bestanden (Kap 5, §4: ATR-relative Pruefung) UND R/R-Verhaeltnis ausreichend. Sonst Fallback auf R-basiertes Target |
| quantScore | QuantScore.overallScore |
| regime | Aktuelles Regime |
| patternState | Aktiver Pattern-State (oder null) |
| llmAnalysisId | runId + requestMarketTime der genutzten LlmAnalysis (fuer Traceability) |
| runId | RunContext.runId |

Der TradeIntent wird an das Risk Gate in `odin-execution` weitergeleitet.

---

## 9. Flow pro Pipeline-State

Die Pipeline-FSM-Zustaende sind in `PipelineState` (Enum in odin-api, Kap 1) definiert:

| Pipeline-State | Decision-Loop-Verhalten |
|----------------|------------------------|
| INITIALIZING | Kein Decision Loop. Daten laden, Warmup beginnt |
| WARMUP | Decision Loop laeuft (§3.1). warmupComplete = false → Entries blockiert, Exits aktiv |
| OBSERVING | Decision Loop aktiv. Regime-Diagnose. Bei klarem Regime + Entry-Signal → TradeIntent |
| POSITIONED | Decision Loop aktiv. Exit-Regeln pruefen. Position-Management (Trailing-Stops, Scaling-Out) |
| PENDING_FILL | Decision Loop laeuft eingeschraenkt. Repricing-Zyklen (OMS, Kap 7). Keine neuen Entry-Intents |
| DAY_STOPPED | Kein Decision Loop. Handel eingestellt (Hard-Stop-Limit erreicht). Bestehende Stops bleiben |
| FORCED_CLOSE | Kein Decision Loop. Zwangs-Schliessung (EOD-Flat). OMS schliesst alle Positionen |
| EOD | Kein Decision Loop. Reconciliation, Tages-Report |

> **State-Uebergaenge:** INITIALIZING → WARMUP → OBSERVING ↔ POSITIONED (via Fill/Exit) → DAY_STOPPED oder FORCED_CLOSE → EOD. PENDING_FILL ist ein Unter-Zustand von OBSERVING (Entry-Order platziert, warte auf Fill).

---

## 10. LLM-Criticality-Klassifikation

| Aktion | LLM-Abhaengigkeit |
|--------|-------------------|
| Stop-Loss-Execution | **Keine** (Broker-seitig, GTC-Stop) |
| Forced Close (EOD-Flat) | **Keine** (MarketClock-basiert) |
| Kill-Switch | **Keine** (Notfall, odin-core) |
| Trailing-Stop-Anpassung | **Keine** (ATR-basiert) |
| Scaling-Out (Tranchen) | **Keine** (R-basiert) |
| Neuer Entry | **Erforderlich, frisch** (< entryFreshnessMax MarketClock-Sekunden, Kap 5 §4) |
| Regime-Wechsel-Pruefung | **Wuenschenswert** (LLM als Sekundaer-Bestaetigung, §4) |
| Exit bei Regime-Wechsel | **Wuenschenswert** (Timeout 10s → Quant-Only-Exit) |
| Position-Management | **Wuenschenswert** (< managementFreshnessMax, Kap 5 §4) |

---

## 11. Reject-Semantik

### Rejects aus Kap 6 (Rules Engine / Arbiter)

| Ablehnungsgrund | RejectReason-Code | Emitter | Reaktion | Retry? |
|-----------------|-------------------|---------|----------|--------|
| Quant Hard-Veto | QUANT_VETO | Arbiter | Intent verwerfen | Nein |
| Quant Soft-Block (Score < Schwelle) | QUANT_SCORE_BELOW_THRESHOLD | Arbiter | Intent verwerfen | Nein (naechster Cycle) |
| Warmup nicht abgeschlossen | WARMUP_INCOMPLETE | Rules Engine | Entry blockiert | Nein (automatisch nach Warmup) |
| LLM stale / nicht vorhanden | LLM_STALE | Rules Engine | Entry blockiert | Nein (wartet auf frischen LLM-Output) |
| regimeConfidence < 0.5 | REGIME_UNCERTAIN | Rules Engine | Entry blockiert | Nein (naechster Cycle) |

### Rejects aus Kap 7 (Risk Gate, odin-execution)

Die folgenden Rejects werden vom Risk Gate emittiert und sind in Kap 7 definiert. Sie sind hier der Vollstaendigkeit halber aufgefuehrt:

| Ablehnungsgrund | RejectReason-Code | Emitter | Reaktion | Retry? |
|-----------------|-------------------|---------|----------|--------|
| Position zu gross | RISK_EXPOSURE_EXCEEDED | Risk Gate | Intent verwerfen | Nein |
| Tages-Budget erschoepft | RISK_BUDGET_EXHAUSTED | Risk Gate | Intent verwerfen | Nein (final fuer heute) |
| R/R unzureichend | RISK_RR_INSUFFICIENT | Risk Gate | Intent verwerfen | Nein (naechster Cycle) |

### EventLog-Schema fuer Rejects

Jeder Reject wird als `DecisionEvent` ins EventLog geschrieben (non-droppable) mit:
- `emitter`: `RULES_ENGINE`, `ARBITER` oder `RISK_GATE`
- `runId` + `instrumentId` + `barCloseTime` als Kontext-Keys
- Vollstaendiger Entscheidungskontext (Regime, Scores, LlmAnalysis-Referenz)

> **Kein automatisches Retry.** Jeder abgelehnte Intent wird final verworfen. Der naechste Bar-Close-Trigger startet eine neue Evaluation mit frischen Daten.

> **EventLog-Modus:** In Simulation synchron (innerhalb Bar-Close-Barrier), in Live asynchron (non-blocking Spool, Kap 2, Kap 5 §11).

---

## 12. Simulation-Semantik

### Identischer Codepfad

Die Rules Engine und der Decision Loop sind in Live und Simulation **identisch** (Kap 0, §3.1). Unterschiede entstehen ausschliesslich durch die injizierten Port-Implementierungen:

| Aspekt | Live | Simulation |
|--------|------|------------|
| MarketClock | SystemMarketClock (Echtzeit) | SimClock (vom Runner gesteuert) |
| LlmAnalysis | Aus Provider-Client (async) | Aus CachedAnalyst (Kap 5 §10) |
| EventLog-Writes | Asynchron (non-blocking Spool) | Synchron (innerhalb Bar-Close-Barrier) |
| Decision-Trigger | Echtzeit Bar-Closes | Vom SimulationRunner freigegeben |
| Regime-Wechsel | MarketClock-basiert (Echtzeit) | SimClock-basiert (deterministische Zeitschritte) |

### RunContext-Nutzung

Die Rules Engine nutzt den `RunContext` (Kap 1, §5) fuer:
- `runId` als Join-Key in allen EventLog-Eintraegen
- `mode` zur Unterscheidung, ob EventLog-Writes synchron oder asynchron erfolgen (nicht die Rules-Logik selbst — die ist identisch)

### Determinismus-Garantien

| Bedingung | Garantie |
|-----------|---------|
| Gleiche MarketData + gleiche LlmAnalysis (CachedAnalyst) + gleiche Config | **Deterministisch reproduzierbar** |
| Gleiche MarketData + Live-LLM | **Best-Effort** (LLM-Output kann variieren, Kap 5 §10) |

---

## 13. Konfiguration

```properties
# odin-brain.properties (Rules-Abschnitt)

# Entry-Schwellenwerte
odin.rules.entry.vwap-offset-atr-factor=0.5
odin.rules.entry.regime-confidence-min=0.5
odin.rules.entry.regime-confidence-high=0.7
odin.rules.entry.quant-score-threshold-standard=0.50
odin.rules.entry.quant-score-threshold-elevated=0.70
odin.rules.entry.rsi-max=70
odin.rules.entry.range-rsi-max=40
odin.rules.entry.range-vwap-support-atr-factor=0.3
odin.rules.entry.high-vol-volume-multiplier=1.5
odin.rules.entry.high-vol-regime-confidence-min=0.7

# Regime-Bestimmung (KPI-Schwellenwerte)
odin.rules.regime.adx-trend-threshold=20
odin.rules.regime.high-volatility-atr-multiplier=1.5
odin.rules.regime.range-vwap-band-atr-factor=0.5

# Aggressive Mode (TREND_DOWN)
odin.rules.aggressive-mode.enabled=false
odin.rules.aggressive-mode.rsi-threshold=25
odin.rules.aggressive-mode.volume-multiplier=2.0
odin.rules.aggressive-mode.max-trades-per-day=2

# Exit-Schwellenwerte
odin.rules.exit.stop-loss-atr-factor=1.5
odin.rules.exit.trailing-stop-atr-factor=1.0
odin.rules.exit.regime-change-confirmations=2
odin.rules.exit.forced-close-buffer-min=30
odin.rules.exit.regime-change-timeout-s=10

# Pattern-State-Machines
odin.rules.pattern.min-confidence=0.5
odin.rules.pattern.flush-range-atr-factor=2.0
odin.rules.pattern.reclaim-max-bars=3
odin.rules.pattern.coil-min-bars=5
odin.rules.pattern.coil-contraction-threshold=0.4
odin.rules.pattern.breakout-range-atr-factor=1.5

# Position Building (Setup A)
odin.rules.pattern.starter-position-percent=40
odin.rules.pattern.add-position-percent=60

# Position Risk
odin.rules.position.max-risk-percent=3.0

# Freshness (Verweis auf Kap 5 / odin-brain.properties LLM-Abschnitt)
# odin.llm.entry-freshness-max-s=120
# odin.llm.management-freshness-max-s=900
```

---

## 14. Abhaengigkeiten und Schnittstellen

### Konsumiert

- `MarketSnapshot` (von odin-data, Kap 2) — Immutable Marktdaten-Snapshot
- `IndicatorResult` (von KPI-Engine, Kap 4) — Technische Indikatoren + warmupComplete-Flag
- `LlmAnalysis` (von LlmAnalysisStore, Kap 5) — LLM-Features (regime, regimeConfidence, patternCandidates, Kontext-Features)
- `MarketClock` (Port, odin-api) — Zeitreferenz fuer Forced-Close, Regime-Wechsel-Timing
- `RunContext` (odin-api) — runId, mode

### Produziert

- `TradeIntent` (Record in odin-api) — Vollstaendig angereicherte Handelsabsicht (an Risk Gate in odin-execution)
- `DecisionEvent` (ins EventLog) — Entscheidung mit vollstaendigem Kontext (Regime, Scores, Reject-Reason)

### Nicht in Scope

- Risk Gate / Position Sizing → Kap 7 (OMS)
- Order-Management / Fill-Handling → Kap 7 (OMS)
- Broker-Anbindung → Kap 3
- KPI-Berechnung → Kap 4
- LLM-Aufruf-Strategie / Caching → Kap 5
