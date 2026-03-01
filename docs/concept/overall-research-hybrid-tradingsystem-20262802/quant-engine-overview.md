# ODIN Quant-Engine — Architekturübersicht

> **Stand:** 2026-02-28
> **Gültigkeit:** Beschreibt den tatsächlich implementierten Code-Stand (odin-brain)
> **Normative Quellen:** Konzept Kap. 4 (KPI), Kap. 5 (LLM), Kap. 6 (Rules), Kap. 8 (Symmetric Hybrid Protocol)

---

## 1. Systemüberblick

Die Quant-Engine ist das analytische Herzstück von ODIN. Sie verarbeitet Marktdaten zu einer
deterministischen Handelsentscheidung — verstärkt durch LLM-Kontext, aber nie von ihm abhängig.

### Architekturprinzip: Symmetrischer Hybrid

ODIN ist kein reines Algo-Trading-System. Quant und LLM sind gleichberechtigte Partner:

- **Quant-Engine**: Deterministische KPI-Berechnung, 7-Gate-Kaskade, Veto-Recht
- **LLM (Tactical Parameter Controller)**: Lagebeurteilung, Subregime-Diagnose, taktische Steuerparameter (bounded Enums)

**Beide liefern einen strukturierten Vote.** Der Decision Arbiter kombiniert beide Votes nach dem
**Dual-Key-Protokoll**: Ein Entry erfordert Zustimmung beider Seiten. Exits können von einer Seite
ausgelöst werden (Either-Key-Prinzip). Hard-Risk-Exits (Stop-Loss, Kill-Switch, Forced Close)
sind vollständig LLM-unabhängig und nicht verhandelbar.

### Big Picture: Decision Cycle

```
MarketSnapshotEvent (Bar-Close, 3m oder 5m)
        │
        v
1. KpiEngine.onSnapshot()
   ├── BarSeries aktualisieren (1m, Decision, 5m)
   ├── Indikatoren berechnen (EMA, RSI, ATR, ADX, Bollinger)
   └── IndicatorResult erzeugen (warmupComplete-Flag)
        │
        v
2. RulesEngine.evaluate()
   ├── RegimeResolver (KPI primär, LLM sekundär)
   ├── Pattern-FSMs aktualisieren (4 Patterns parallel)
   ├── ExitRules prüfen (Prio 1-7, Exit > Entry)
   └── EntryRules prüfen (nur wenn State=OBSERVING + Warmup)
        │ → RulesResult (intent + regime + confidence)
        v
3. GateCascadeEvaluator (7 Gates, Short-Circuit)
   └── SPREAD → VOLUME → RSI → EMA_TREND → VWAP → ATR → ADX
        │ → GateCascadeResult (passed / failedGate)
        v
4. DecisionArbiter.decide()  [Dual-Key Symmetric Hybrid Protocol]
   ├── QuantVote aufbauen (aus RulesResult + GateCascadeResult)
   ├── LlmVote aufbauen (aus LlmAnalysisStore — async bereitgestellt)
   ├── Fuse: Regime, ExitBias, TrailMode, SizeModifier etc.
   └── TradeIntent erzeugen (oder NO_ACTION)
        │ → DecisionResult (FusionResult + optionaler TradeIntent)
        v
5. Risk Gate (odin-execution): Position Sizing, R/R, globale Limits
        │
        v
6. OMS (odin-execution): Order erzeugen, Stop platzieren, Fill verwalten
```

**Threading:** Decision Loop läuft single-threaded pro Pipeline. LLM-Calls laufen asynchron
im Hintergrund (Single-Flight-Semantik). Der Arbiter liest den zuletzt validierten LLM-Output
aus dem `LlmAnalysisStore` (AtomicReference, monotoner Update).

---

## 2. KPI-Engine

**Klasse:** `de.its.odin.brain.kpi.KpiEngine`
**Abhängigkeiten:** `MarketSnapshot` (odin-data), ta4j, `BrainProperties`

### 2.1 Indikatoren-Katalog

| Indikator | Timeframe | ta4j-Klasse | Verwendung |
|-----------|-----------|-------------|-----------|
| EMA(9) | Decision-Bar (3m oder 5m) | `EMAIndicator` | Trendrichtung (Entry/Exit), Kreuzung mit EMA21 |
| EMA(21) | Decision-Bar (3m oder 5m) | `EMAIndicator` | Trendrichtung (Entry/Exit), Kreuzung mit EMA9 |
| EMA(50) | 5-Min (fix) | `EMAIndicator` | Chart-Anzeige, Regime-Kontext |
| EMA(100) | 5-Min (fix) | `EMAIndicator` | Chart-Anzeige, Regime-Kontext |
| RSI(14) | 5-Min (fix) | `RSIIndicator` | Überkauft/Überverkauft (Entry-Gate, Veto) |
| ATR(14) | 5-Min (fix) | `ATRIndicator` | Volatilität, Stop-Distanz, Trailing-Stop |
| Bollinger Bands(20, 2σ) | 5-Min (fix) | `BollingerBandsMiddle/Upper/LowerIndicator` | Überhitzungserkennung (Exhaustion-Detektion) |
| ADX(14) | 5-Min (fix) | `ADXIndicator` | Trendstärke-Messung (Gate + Regime) |
| +DI(14) | 5-Min (fix) | `PlusDIIndicator` | Direktionale Kraft (Regime-Feinsteuerung) |
| -DI(14) | 5-Min (fix) | `MinusDIIndicator` | Direktionale Kraft (Regime-Feinsteuerung) |
| VWAP | — | Konsumiert aus `MarketSnapshot.vwap()` | Trend-Alignment, Entry-Gate, Veto |
| Anchored VWAP | — | Konsumiert aus `MarketSnapshot.anchoredVwap()` | Chart-Anzeige, LLM-Kontext |
| ATR-Decay-Ratio | Abgeleitet aus ATR(14) | Eigenimplementierung | LLM-Kontext, Volatilitätsabbau |
| Volume-Ratio | Abgeleitet aus Decision-Bars | Eigenimplementierung (SMA-20) | Volume-Gate, Exhaustion-Erkennung |

**Trennung der Timeframes:** EMA(9)/EMA(21) laufen auf Decision-Bars (konfigurierbar 3m oder 5m).
RSI, ATR, ADX, Bollinger laufen **immer auf 5-Min-Bars** — unabhängig vom Decision-Timeframe.
Das garantiert konsistente Risikoindikatoren unabhängig von der Decision-Bar-Granularität.

**VWAP ist Source-of-Truth in odin-data.** Die KPI-Engine berechnet VWAP nicht selbst,
sondern konsumiert ihn aus dem MarketSnapshot.

### 2.2 Berechnung und Timing

**Klassen:** `KpiEngine`, `BarSeriesManager`, `IndicatorCalculationService`

Bei jedem Decision-Bar-Close (`MarketSnapshotEvent`) läuft die KPI-Engine synchron:

1. Neue Bars aus dem Snapshot extrahieren (Delta über `barEndTime`, Duplikat-Schutz)
2. BarSeries für 1m, Decision-Timeframe und 5m aktualisieren
3. ta4j berechnet alle angehängten Indikatoren (lazy evaluation)
4. `IndicatorResult`-Record erzeugen (immutable, alle Werte + `warmupComplete`-Flag)

**Warmup:** Indikatoren gelten erst als valide nach `period + 1` Bars (RSI, ATR) bzw.
`2 × period` Bars (ADX). Während Warmup: `NaN`-Werte + `warmupComplete = false`.
Die Rules Engine blockiert bei `warmupComplete = false` alle Entry-Intents.

**SOD-Reset:** Jeder Handelstag beginnt mit leeren BarSeries (`KpiEngine.reset()`).

**IndicatorResult** (Record in odin-api):
```
runId, instrumentId, barTime (MarketClock),
ema9, ema21, ema50, ema100,
rsi14_5m, atr14_5m,
bollingerUpper_5m, bollingerMiddle_5m, bollingerLower_5m,
adx14_5m, plusDi14, minusDi14,
atrDecayRatio, volumeRatio,
vwap, anchoredVwap,
warmupComplete, flags
```

---

## 3. Rules Engine

**Klasse:** `de.its.odin.brain.rules.RulesEngine`
**Enthält:** `RegimeResolver`, `EntryRules`, `ExitRules`, 4 Pattern-FSMs

### 3.1 FSM-Architektur

Die Pipeline-FSM (in odin-core) steuert den übergeordneten Lebenszyklus:

| State | Decision-Loop-Verhalten |
|-------|------------------------|
| `INITIALIZING` | Kein Decision Loop |
| `WARMUP` | Loop aktiv, `warmupComplete=false` → nur Exits |
| `OBSERVING` | Loop aktiv, Entry- und Exit-Prüfung |
| `POSITIONED` | Loop aktiv, Exit-Prüfung priorisiert |
| `PENDING_FILL` | Loop eingeschränkt, Repricing |
| `DAY_STOPPED` / `FORCED_CLOSE` / `EOD` | Kein Decision Loop |

Innerhalb der `RulesEngine` verwalten vier **Pattern-State-Machines** das Erkennen von
Intraday-Setups. Diese laufen parallel und sind unabhängig voneinander.

### 3.2 Regime-Bestimmung

**Klasse:** `de.its.odin.brain.rules.RegimeResolver`

Die Regime-Bestimmung erfolgt in zwei Schritten:

**Primär (KPI-deterministisch):**

| Regime | KPI-Kriterien |
|--------|---------------|
| `TREND_UP` | EMA9 > EMA21 AND Preis > VWAP AND ADX > Schwelle |
| `TREND_DOWN` | EMA9 < EMA21 AND Preis < VWAP AND ADX > Schwelle |
| `RANGE_BOUND` | ADX < Schwelle AND Preis innerhalb VWAP ± ATR-Band |
| `HIGH_VOLATILITY` | ATR(aktuell) > ATR-Multiplier × ATR(20-Bar-SMA) |
| `UNCERTAIN` | Kein anderes Kriterium klar erfüllt |

**Sekundär (LLM als Konservativitätsfilter):**

| Situation | Ergebnis |
|-----------|----------|
| KPI-Regime == LLM-Regime | KPI-Regime gilt |
| KPI ≠ LLM, LLM-Confidence >= 0.5 | Konservativeres Regime gilt (Long-Only-Rangfolge: UNCERTAIN > TREND_DOWN > HIGH_VOL > RANGE_BOUND > TREND_UP) |
| KPI ≠ LLM, LLM-Confidence < 0.5 | KPI-Regime gilt allein |
| Kein LLM-Output | KPI-Regime gilt allein |

**Regime-Hysterese:** Ein Regimewechsel erfordert N konsekutive Decision-Bar-Closes mit
konsistentem neuem Regime (konfigurierbar: `regimeChangeConfirmations`, Default: 2).

**Subregime:** Der `RegimeResolver` bestimmt zusätzlich ein Subregime (z.B. `TREND_UP/EARLY`,
`TREND_UP/EXHAUSTION`). Pro Basisregime gibt es 4 Subregimes (5×4 = 20 Zustände). KPI primär,
LLM als sekundärer Hint. Subregime-Wechsel innerhalb eines Regimes gelten sofort ohne Hysterese.

### 3.3 Entry-Regelwerk

**Klasse:** `de.its.odin.brain.rules.EntryRules`

Entry-Prüfung läuft nur bei `state == OBSERVING` und wenn die Voraussetzungen erfüllt sind:
- `warmupComplete == true`
- `flags` enthält kein `CRASH_SIGNAL` oder `STALE_WARNING`
- `regimeConfidence >= regimeConfidenceMin` (konfigurierbar)

**Opening Pattern Gate:** Wenn aktiviert und das Opening Pattern `LOWER_HALF` zeigt → Entry blockiert.

**Regime-Confidence-Gate** (vor jeder Entry-Prüfung):

| regimeConfidence | Konsequenz |
|-----------------|-----------|
| < 0.5 | Regime gilt als UNCERTAIN → kein Entry |
| 0.5–0.7 | Entry mit erhöhter Gate-Kaskaden-Strenge |
| > 0.7 | Entry mit Standard-Gate-Strenge |

**Regime-spezifische Entry-Bedingungen:**

| Regime | KPI-Trigger-Bedingungen | S/R-Erweiterungen |
|--------|------------------------|-------------------|
| `TREND_UP` | EMA9 > EMA21 AND RSI < rsiMax AND Preis ≤ VWAP + vwapOffsetAtrFactor × ATR | LLM `entry_price_zone` als zusätzliche Bestätigung; Bounce-Signal bypasses VWAP-Check |
| `RANGE_BOUND` | RSI < rangeBoundRsiMax AND Preis nahe VWAP-Support (< rangeVwapSupportAtrFactor × ATR) | S/R-Proximity relaxiert VWAP-Constraint proportional zur Zone-Confidence |
| `HIGH_VOLATILITY` | Volume-Ratio > highVolVolumeMultiplier AND Spread < Schwelle | Erfordert regimeConfidence > highVolRegimeConfidenceMin |
| `TREND_DOWN` / `UNCERTAIN` | **Kein Entry** (Long-Only v1) | — |

**Pattern-Einträge** haben Vorrang vor Standard-Einträgen: Wenn ein Pattern-Signal mit
ausreichender Confidence aktiv ist, wird es als Entry verwendet (ohne Standard-Checks).

**S/R-Integration (SrEntryContext):** Die Entry-Rules erhalten drei Signale aus dem S/R-System:
- **Bounce-Signal** (`BounceEvent`): Aktives Bounce-Signal ≥ `bounceConfidenceMin` triggert
  direkt einen Entry und bypasses den VWAP-Proximity-Check. Der Stop wird tighter gesetzt
  (Faktor 0.5 auf `stopLossAtrFactor`).
- **S/R-Proximity**: Preis nahe einem Support-Level (< `srProximityAtrMax × ATR`) mit hoher
  Konfidenz (≥ 0.60) relaxiert den VWAP-Constraint.
- **Opening Pattern Gate**: `LOWER_HALF` blockiert Entry, `UPPER_HALF` bestätigt.

### 3.4 Exit-Regelwerk

**Klasse:** `de.its.odin.brain.rules.ExitRules`

Exits werden in **fester Prioritätsreihenfolge** geprüft. Erste Übereinstimmung gewinnt:

| Prio | Trigger | LLM-Abhängigkeit | Mechanismus |
|------|---------|------------------|-------------|
| 1 | Kill-Switch | Keine | odin-core Signal |
| 2 | Forced Close (EOD) | Keine | MarketClock: RTH-Ende − forcedCloseBufferMin |
| 3 | Stop-Loss | Keine | Preis ≤ Entry − stopLossAtrFactor × entryATR |
| 4 | Trailing Stop | Keine | Preis fällt um trailingStopAtrFactor × entryATR vom Intraday-High. **Highwater-Mark**: Trail kann nur steigen, nie fallen. **entryATR eingefroren**: ATR-Wert bei Entry-Zeitpunkt fixiert |
| 5 | Regime-Wechsel | Wünschenswert | Bestätigter Regimewechsel zu TREND_DOWN oder UNCERTAIN bei Long-Position |
| 6 | Tactical Exit (LLM) | Ja (mit Guards) | LLM `exit_bias ∈ {EXIT_SOON, EXIT_NOW}` + min. Gewinn (0.5R) + Exhaustion-Bestätigung + min. Haltedauer (3 Bars) + Anti-Drift-Limit (max 1 TACTICAL_EXIT pro Position) |
| 7 | Gewinnmitnahmen (R-Tranchen) | Keine | OMS-intern, R-basierte Scaling-Out-Profile |

**Exhaustion-Detektor** (`ExhaustionDetector`): Drei-Säulen-Modell für KPI-basierte
Erschöpfungsbestätigung. Alle drei Säulen müssen aktiv sein:
- **Säule A (Extension):** RSI ≥ 78 ODER Close ≥ BB-Upper ODER VWAP-Extension ≥ 2.5×ATR
- **Säule B (Climax):** Volume-Ratio ≥ 2.5 ODER Bar-Range ≥ 1.8×ATR
- **Säule C (Rejection/Momentum-Loss):** BB-Re-Entry ODER VWAP-Snapback ODER RSI-Drop ≥ N Punkte ODER EMA-Spread schrumpft

### 3.5 Risk Management

**R-Floor Profit Protection** (`ProfitProtectionEngine`): R-basierte Stufentabelle setzt
einen Mindest-Stop-Floor. Der effectiveStop = `max(trailingStop, rFloor)` — der höhere Wert gewinnt.

| Profil | Aktivierungschwellen |
|--------|---------------------|
| `MODERATE` | 1.0R → BE, 1.5R → Floor +0.5R, 2.0R → +1.0R, 3.0R → +2.0R, 4.0R → +3.0R |
| `AGGRESSIVE` | 0.5 R früher als MODERATE (aggressivere Absicherung) |
| `RELAXED` | 0.5 R später als MODERATE |

Das aktive Profil wählt der LLM via `profit_protection_profile` — der Arbiter übernimmt
das **konservativere** der beiden Votes (Quant vs. LLM).

### 3.6 Pattern-State-Machines

Vier Pattern-FSMs laufen parallel. Nur ein Pattern darf gleichzeitig eine Position halten.

| Pattern-FSM | Klasse | Setup-Beschreibung |
|-------------|--------|--------------------|
| Flush → Reclaim → Run | `FlushReclaimRunFsm` | Starker Abwärtssweep → Reclaim über VWAP/MA → Trend-Continuation |
| Coil → Breakout | `CoilBreakoutFsm` | Kompression (fallende Hochs + steigende Tiefs) → Breakout mit ATR-Bestätigung |
| Opening Consolidation | `OpeningConsolidationFsm` | RTH-Opening-Pattern: Upper/Lower-Half-Bestätigung |
| Re-Entry / Correction | `ReEntryCorrectionFsm` | Pullback nach initialer Bewegung, re-entry nach strukturellen Tiefs |

Alle State-Transitions werden durch **KPI-Kriterien** ausgelöst — LLM-`patternCandidates`
schlägt Patterns vor, aktiviert aber keine Transitions allein (Analyst-Prinzip).

---

## 4. LLM-Integration

**Paket:** `de.its.odin.brain.llm`
**Zentrale Klassen:** `LlmAnalystOrchestrator`, `PromptBuilder`, `LlmAnalysisStore`,
`SchemaValidator`, `PlausibilityChecker`, `ConsistencyChecker`, `CircuitBreaker`

### 4.1 Wann wird der LLM aufgerufen?

Der LLM-Aufruf ist **asynchron** — er blockiert den Decision-Cycle nie.

**Periodisch (volatilitätsabhängig, MarketClock-basiert):**

| Volatilität (ATR vs. Daily-ATR) | Intervall |
|--------------------------------|-----------|
| > 120% Daily-ATR | Alle 3 Minuten |
| 80–120% | Alle 5 Minuten |
| < 80% | Alle 15 Minuten |

**Event-getriggert (zusätzlich):**
- Entry-Approaching: Preis nähert sich einer Opportunity-Zone oder einem OMS-Struktur-Level
- Volumen-Spike (> 2×), VWAP-Durchbruch
- Stop-Loss-Annäherung (< 0.5×ATR)
- State-Transition der Pipeline-FSM
- Crash/Spike-Events, Exhaustion-Climax auf 1m

**Single-Flight:** Nie mehr als ein laufender LLM-Call pro Pipeline. Weitere Trigger
innerhalb eines laufenden Calls werden koalesziert.

### 4.2 Input (was bekommt der LLM?)

**Dreistufiges Context Window** (`PromptBuilder`):

| Tier | Inhalt | Aktualisierung |
|------|--------|---------------|
| Tier 1: System Prompt | Rollendefinition, vollständiges Bounded-Enum-Schema, Safety Contract (keine Preise/Quantities/Orders), Konfidenz-Kalibrierung, Prompt-Version | Fix pro Prompt-Version |
| Tier 2: Dynamischer Kontext | Session-State, bisherige Trades, offene Position, Opening-Pattern-Ergebnis, S/R-Levels (top 5 nach Konfidenz), aktive Bounce-Events (max 3) | Pro LLM-Aufruf |
| Tier 3: Aktueller Snapshot | Letzte 15 × 1m-Bars, 24 × 5m-Bars, 20 Daily-Bars (komprimiert), aktuelle Indikatoren aus IndicatorResult (RSI, ATR, EMA, ADX, Bollinger), VWAP, Volumen-Anomalien, MarketClock-Zeit + verbleibende Handelszeit | Pro LLM-Aufruf |

**Safety Contract:** Der System-Prompt verbietet dem LLM explizit, numerische Preise,
Stückzahlen, Stop-Levels oder Order-Typen zu emittieren. Nur bounded Enums aus dem Schema
sind im strukturierten Output-Bereich erlaubt.

### 4.3 Output (was liefert der LLM?)

Das LLM liefert `LlmTacticalOutput` — ein vollständig strukturiertes, valides Enum-Objekt
nach bestandenem 5-Schichten-Anti-Halluzinations-Pipeline:

1. **Schema-Validierung** (`SchemaValidator.validateTactical`): Strict-Schema, alle Enums geschlossen, keine `additionalProperties`
2. **Plausibilitätsprüfung** (`PlausibilityChecker.checkTactical`): ATR-relative Checks, Zeitvergleiche gegen MarketClock
3. **Konsistenzprüfung** (`ConsistencyChecker.checkTactical`): Feld-Kombinationen (z.B. TREND_DOWN + Entry-Zone ist inkonsistent)
4. **Freshness/TTL-Gate**: Caller-seitig über `LlmAnalystOrchestrator.isTacticalAnalysisFresh()`
5. **Fallback**: Bei REJECT → letzter valider Output beibehalten (falls innerhalb TTL), sonst Defaults

**Feature-Kategorien:**

| Kategorie | Felder | Nutzung |
|-----------|--------|---------|
| Decision-Features | `regime`, `regime_confidence`, `pattern_candidates`, `setup_type`, `tactic`, `urgency_level` | Input für RegimeResolver und Decision Arbiter |
| Kontext-Features | `opportunity_zones`, `hold_duration_bars` | Optionale Signale in EntryRules |
| Tactical Controls P0 | `exit_bias`, `trail_mode`, `profit_protection_profile`, `target_policy` | Taktische Steuerung im Arbiter |
| Tactical Controls P1 | `subregime`, `exhaustion_signal` | Regime-Feinsteuerung |
| Tactical Controls P2/P3 | `entry_timing_bias`, `scale_out_profile` | Erweiterte Steuerung |
| Logging-Only | `reasoning`, `key_observations`, `risk_factors`, `market_context_signals`, `explanation_short` | Nur Telemetrie, nie in Entscheidungslogik |

### 4.4 Structured Output Format (`LlmTacticalOutput`)

Das vollständige Feld-Schema (bounded Enums):

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `regime` | Enum: OPENING_VOLATILITY, TREND_UP, TREND_DOWN, RANGE_BOUND, HIGH_VOLATILITY, BREAKOUT_ATTEMPT, EXHAUSTION_RISK, RECOVERY, UNCERTAIN | LLM-diagnostiziertes Marktregime |
| `regime_confidence` | Float 0.0–1.0 | Konfidenz der Regime-Einschätzung |
| `pattern_candidates` | Object[] (max 2): `{pattern: Enum, confidence: Float, phase: String}` | Erkannte Patterns; Pattern-Enums: FLUSH_RECLAIM_RUN, COIL_BREAKOUT, OPENING_CONSOLIDATION, PULLBACK_REENTRY, NONE |
| `setup_type` | Enum: OPENING_CONSOLIDATION, FLUSH_RECLAIM, COIL_BREAKOUT, PULLBACK_REENTRY, RECOVERY_TRADE, NONE | Erkannter Setup-Typ |
| `tactic` | Enum: WAIT_PULLBACK, BREAKOUT_FOLLOW, NO_TRADE | Taktische Empfehlung |
| `urgency_level` | Enum: LOW, MEDIUM, HIGH | Zeitkritikalität |
| `exit_bias` | Enum: HOLD, NEUTRAL, EXIT_SOON, EXIT_NOW | Aktiviert TACTICAL_EXIT-Check in ExitRules |
| `trail_mode` | Enum: WIDE, NORMAL, TIGHT | Multiplikator auf base `trailingStopAtrFactor` (1.5×, 1.0×, 0.75×) |
| `profit_protection_profile` | Enum: OFF, STANDARD, AGGRESSIVE | R-Stufentabelle für ProfitProtectionEngine |
| `target_policy` | Enum: KEEP, CAP_AT_R_MULTIPLE, TRAIL_ONLY | Target-Order-Steuerung im OMS |
| `subregime` | Enum: 20 Subregimes (4 pro Basisregime) | Regime-Feinsteuerung |
| `exhaustion_signal` | Enum: NONE, EARLY_WARNING, CONFIRMED | LLM-Einschätzung zur Erschöpfung |
| `entry_timing_bias` | Enum: ALLOW_NOW, WAIT_PULLBACK, WAIT_CONFIRMATION, ALLOW_RE_ENTRY, SKIP | Entry-Guard-Modifikator |
| `scale_out_profile` | Enum: OFF, CONSERVATIVE, STANDARD, AGGRESSIVE | OMS-Tranchen-Profil |

**Freshness-Regeln (MarketClock-basiert):**

| Kontext | Max-Alter | Reaktion |
|---------|-----------|---------|
| Entry-Entscheidung | 120s (`odin.llm.entry-freshness-max-s`) | Entry blockiert |
| Position-Management | 900s (`odin.llm.management-freshness-max-s`) | Letzten validen Output verwenden |
| Kein Output | — | Default: `regimeConfidence = 0.0`, Entries blockiert |

**Simulation:** `CachedAnalyst` liefert gecachte LLM-Responses für reproduzierbare Backtests.
Cache-Key: `Hash(canonicalizedRequestPayload + promptVersion + modelId)`. Der `Δt_model`-Parameter
simuliert realistische Antwortlatenz über MarketClock.

**Circuit Breaker:** 3 konsekutive Timeouts/Fehler → Quant-Only-Modus (neue Entries blockiert).
> 30 Min Ausfall → Trade-Halt. Bestehende Positionen laufen mit ihren Stops weiter.

---

## 5. Decision Arbiter

**Klassen:** `de.its.odin.brain.arbiter.DecisionArbiter`, `FusionResult`, `QuantVote`, `LlmVote`
**Protokoll:** Symmetric Hybrid Protocol (Dual-Key)

### 5.1 Quant + LLM Zusammenspiel

Der Arbiter kombiniert zwei strukturierte Votes zu einem deterministischen Ergebnis:

**QuantVote** (aus `RulesResult` + `GateCascadeResult`):

| Feld | Herkunft |
|------|---------|
| `vote` | RulesResult: STRONG_ENTRY, ALLOW_ENTRY, NO_TRADE, HOLD, EXIT_SOON, EXIT_NOW |
| `regime` / `regimeConfidence` | RegimeResolver |
| `gatesPassed` / `failedGates` | GateCascadeEvaluator (7 Gates) |
| `hardVetoFlags` | WARMUP_INCOMPLETE, DQ_VIOLATION, etc. |

**LlmVote** (aus `LlmTacticalOutput` im `LlmAnalysisStore`):

| Feld | Herkunft |
|------|---------|
| `vote` | LLM-deriviert aus `exit_bias`, `tactic`, `entry_timing_bias` |
| `regime` / `regimeConfidence` | LLM-Output |
| `exitBias`, `trailMode`, `profitProtectionProfile`, `targetPolicy` | LLM Tactical Controls |
| `entryTiming`, `sizeModifier` | LLM Tactical Controls |

### 5.2 Fusion: Konservativität als Leitprinzip

Der Arbiter fusioniert die Parameter nach dem Prinzip: **der konservativere Wert gewinnt.**

| Parameter | Fusionsregel |
|-----------|-------------|
| `fusedRegime` | Konservativeres Regime (Rangfolge: UNCERTAIN > TREND_DOWN > HIGH_VOL > RANGE_BOUND > TREND_UP) |
| `sizeModifier` | Kleinere Position (MINIMAL < REDUCED < FULL) |
| `profitProtectionProfile` | Engeres Profil (AGGRESSIVE < MODERATE < RELAXED) |
| `exitBias` | Aggressiverer Exit |
| `trailMode` | Engerer Trail (`effectiveTrailFactor = min(llm_factor, profit_protection_factor)`) |
| `targetPolicy` | LLM-bevorzugt, Quant kann überschreiben |
| `entryTiming` | LLM-bevorzugt |

### 5.3 Veto-Logik und Konfliktauflösung

**Dual-Key-Regel (Entry):** Entry erfordert BEIDE Keys:

| Bedingung | Ergebnis |
|-----------|----------|
| Quant-HardVeto aktiv | Entry blockiert (unabhängig vom LLM) |
| LLM `SAFETY_VETO` | Entry blockiert |
| QuantVote ALLOW/STRONG + GateCascade bestanden + LlmVote ALLOW/STRONG | Entry möglich |
| QuantVote ALLOW/STRONG + GateCascade bestanden + LlmVote NO_TRADE | Entry blockiert |
| QuantVote NO_TRADE + LlmVote ALLOW/STRONG | Entry blockiert |
| Kein LLM-Output (QUANT_ONLY) | Entry blockiert, bestehende Positionen weiter verwaltet |

**Either-Key-Regel (Exit):** Exit kann von einer Seite ausgelöst werden:
- `EXIT_NOW` von einer Seite → Pending-Exit wird gesetzt, Bestätigung innerhalb N Bars
- Hard-Risk-Exits (Prio 1–3) sind nicht verhandelbar — kein Arbiter-Konsens nötig

**Regime-Confidence-Schwellen:**

| regimeConfidence | Gate-Kaskaden-Modus |
|-----------------|---------------------|
| < 0.5 | Kein Entry (`elevated=true`, aber categorically blocked) |
| 0.5–0.7 | `elevated=true` → stricter Gate-Schwellen |
| > 0.7 | `elevated=false` → Standard-Gate-Schwellen |

**7-Gate-Kaskade** (`GateCascadeEvaluator`): Short-Circuit bei erstem Fehler:

| Gate | Typ | Bedingung |
|------|-----|-----------|
| SPREAD | Hard-Veto | Bid-Ask-Spread zu breit |
| VOLUME | Hard-Veto | Volume-Ratio zu niedrig |
| RSI | Hard-Veto | RSI in extremer Zone |
| EMA_TREND | Moduliert | EMA-Trend-Misalignment (strenger bei elevated) |
| VWAP | Moduliert | Preis zu weit über VWAP (ATR-Faktor strenger bei elevated) |
| ATR | Moduliert | Intraday-ATR-Decay-Ratio zu niedrig (strenger bei elevated) |
| ADX | Moduliert | ADX unter Trendstärke-Schwelle (strenger bei elevated) |

**TradeIntent-Erzeugung** (Record in odin-api): Der Arbiter reichert den TradeIntent mit allen
Fusion-Ergebnissen an: `entryPrice`, `stopLevel`, `targetPrice`, `quantScore`, `fusedRegime`,
`patternState`, `llmAnalysisId`, `sizeModifier`, `trailMode`, `profitProtectionProfile`,
`targetPolicy`, `entryTiming`. HMAC-Signierung durch `IntentSigner` verhindert Manipulation.

---

## 6. Datenfluss: End-to-End

```
[Bar-Close Event empfangen]
        │
        v
[KpiEngine.onSnapshot(snapshot)]
  → BarSeries.update(delta)
  → ta4j Indikatoren (EMA9, EMA21, RSI14, ATR14, ADX14, Bollinger)
  → Derived Values (volumeRatio, atrDecayRatio)
  → IndicatorResult (immutable Record)
        │
        v
[RulesEngine.evaluate(snapshot, indicatorResult, llmAnalysis, state, ...)]
  → RegimeResolver.resolve(indicatorResult, llmAnalysis, currentPrice)
       → KPI-Regime (deterministisch) + LLM-Konservativitätsfilter
       → Subregime + Regime-Hysterese (2-Bar-Confirmation)
       → ResolvedRegime (regime + confidence)
  → Pattern-FSMs: FlushReclaimRun, CoilBreakout, OpeningConsolidation, ReEntryCorrection
       → PatternSignal (wenn State-Machine-Transition ausgelöst)
  → ExitRules.checkExit(...) [wenn POSITIONED/PENDING_FILL/FORCED_CLOSE]
       → Prioritäten 1-7 durchlaufen
       → TrailingStop-Highwatermark aktualisieren
       → R-Floor-Highwatermark aktualisieren
  → EntryRules.checkEntry(...) [wenn OBSERVING + Voraussetzungen]
       → Opening Pattern Gate
       → Bounce-Signal-Gate
       → Regime-Confidence-Gate
       → Regime-spezifische Bedingungen + S/R-Kontext
  → RulesResult (intent: ENTRY/EXIT/NO_ACTION + regime + confidence)
        │
        v
[GateCascadeEvaluator.evaluate(indicatorResult, snapshot, elevated)]
  → 7 Gates in Reihe (Short-Circuit bei erstem Fail)
  → GateCascadeResult (passed / failedGate / evaluations)
        │
        v
[DecisionArbiter.decide(rulesResult, indicatorResult, tacticalOutput, snapshot, ...)]
  → QuantVote aufbauen (rulesResult + gateCascadeResult)
  → LlmVote aufbauen (tacticalOutput aus LlmAnalysisStore)
       → wenn null → QUANT_ONLY Fallback
  → fuse(quantVote, llmVote):
       → Regime-Konservativität: min(quant.regime, llm.regime) nach Rang
       → Parameter-Fusion: konservativerer Wert gewinnt
       → Dual-Key Entry: BEIDE Keys müssen ALLOW/STRONG sein
       → Either-Key Exit: EIN Key reicht für EXIT
  → FusionResult (action + fusedParams + winnerSide)
  → TradeIntent erzeugen (wenn action == ENTER/EXIT)
       → HMAC-Signierung (IntentSigner)
  → DecisionEvent ins EventLog (non-droppable)
        │
        v
[Risk Gate (odin-execution)]
  → Position-Sizing (ATR-basiert, SizeModifier aus Fusion)
  → Globale Limits prüfen (AccountRiskState)
  → R/R-Prüfung
        │
        v
[OMS (odin-execution)]
  → Order erzeugen (Bracket: Entry + Stop + Target(s))
  → Scaling-Out-Tranchen (ScaleOutProfile aus Fusion)
  → Fill-Handling, Stop-Nachführung

[Parallel — asynchron im Hintergrund]
LlmAnalystOrchestrator.triggerAsync(snapshot, indicatorResult, ...)
  → CallCadencePolicy: Soll jetzt ein Call abgesetzt werden?
  → CircuitBreaker: Ist das System im Quant-Only-Modus?
  → PromptBuilder: System Prompt + User Message (3-tier Context Window)
  → Provider-Client (Claude/OpenAI/Cached): HTTP-Call
  → 5-Layer Validation (Schema → Plausibilität → Konsistenz)
  → LlmAnalysisStore.updateMonotonic(validatedOutput)
       → AtomicReference: nur wenn requestMarketTime >= currentMarketTime
```

**Wichtige Invarianten:**
- `MarketClock` ist die einzige Zeitquelle — kein `Instant.now()` im Trading-Codepfad
- Alle Entscheidungs-Records sind immutable (Java Records)
- EventLog-Einträge sind non-droppable (Live: async Spool, Sim: synchron)
- TradeIntents sind HMAC-signiert (Integritätsschutz)
- LLM-Output im `LlmAnalysisStore` ist monoton (kann nie auf älteren Stand zurückfallen)
- Pro Decision-Cycle maximal ein TradeIntent pro Pipeline (Ein-Intent-pro-Bar-Garantie)

---

## 7. Was ist noch Stub / nicht vollständig implementiert?

Zum Stand 2026-02-28 sind alle Kernkomponenten implementiert und compilierbar. Folgende Aspekte
sind konzipiert aber noch nicht vollständig integriert oder aktiviert:

| Komponente | Status |
|------------|--------|
| News-Input | Konzipiert (`odin.llm.news.enabled=false`), Code-Infrastruktur vorbereitet |
| Pre-Market Instrument Profiling (Warmup-LLM) | Konzipiert in Kap. 8 des Konzepts, Code in `InstrumentProfiler.java` |
| Re-Entry / ALLOW_RE_ENTRY Pfad | Implementiert in `ReEntryCorrectionFsm`, `FLAT_INTRADAY`-State noch in Arbeit |
| Multi-Instrument Koordination (GlobalRiskManager) | odin-core Level, implementiert |
| Vision-Modul (Chart-Screenshot an LLM) | Für spätere Version geplant, nicht implementiert |
