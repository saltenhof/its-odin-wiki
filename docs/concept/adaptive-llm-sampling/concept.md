# Adaptive LLM Sampling — Need-Driven + Phase-Aware Call-Cadence

> **Status:** ENTWURF (ChatGPT-Review eingearbeitet)
> **Datum:** 2026-03-02
> **Kontext:** Erkenntnisse aus IREN-Backtest (23.02.2026): Erster LLM-Call erst 20 Minuten nach Handelsbeginn, verpasster Entry bei 09:45 EST trotz Quant-ALLOW_ENTRY-Signal.
> **Bezug:** 04-llm-integration.md §6 (LLM Call-Cadence), ODIN-084 (Market Data Level Gates)
> **Review:** ChatGPT (2026-03-02) — wesentliche Aenderungen: Need-Driven-Trigger als primaerer Mechanismus, ATR-normalisierte Event-Schwellen, Freshness-Klaerung

---

## 1. Problemstellung

### 1.1 Beobachtung

Im IREN-Backtest vom 23.02.2026 wurde der erste LLM-Call erst bei 14:50 UTC (09:50 EST) ausgeloest — **20 Minuten nach Handelsbeginn** (14:30 UTC / 09:30 EST). In dieser Zeitspanne hat das Quant-Modul bei 14:45 UTC ein klares Entry-Signal generiert (TREND_UP, price=40.13, volRatio=4.21), das wegen fehlender LLM-Analyse durch den Dual-Key-Mechanismus blockiert wurde.

Der tatsaechliche Entry erfolgte erst bei 14:50 UTC zu 40.64 — der Kurs war bereits **51 Cent** ueber dem verpassten Level. Bei einer Positionsgroesse von 200 Stueck entspricht das **102 USD entgangenem P&L** allein durch den verspaeteten Einstieg.

### 1.2 Ursache

Die `TimeBasedSamplingPolicy` ist mit `intervalBars=5` konfiguriert. Da im Backtest 5-Minuten-Bars verwendet werden und der `currentBarIndex` erst bei RTH-Beginn bei 0 startet, ergibt sich:

| Bar-Index | SimClock (UTC) | `(barIndex - 0) >= 5` | Ergebnis |
|-----------|---------------|------------------------|----------|
| 1 | 14:30 | false | Kein LLM-Call |
| 2 | 14:35 | false | Kein LLM-Call |
| 3 | 14:40 | false | Kein LLM-Call |
| 4 | 14:45 | false | **Entry-Signal verpasst** |
| 5 | 14:50 | true | Erster LLM-Call |

Im Live-Modus mit 3-Minuten-Decision-Bars wuerde der erste Call bei 14:42 UTC erfolgen — immer noch **12 Minuten** nach Handelsbeginn.

### 1.3 Warum die Opening-Phase kritisch ist

Die ersten 15-30 Minuten nach Handelseroeffnung sind statistisch die wichtigste Phase des Handelstages:

- **Hoechste Liquiditaet und Volatilitaet** — >30% des Tagesvolumens werden in den ersten 30 Minuten gehandelt
- **Gap-Resolution** — Overnight-Gaps werden aufgeloest (Gap-and-Go oder Gap-and-Fade)
- **Trendinitiierung** — Die Opening Range (OR) definiert haeufig die Tagesrichtung
- **High-Beta-Aktien** (wie IREN) zeigen ueberproportionale Bewegungen in dieser Phase
- **Institutionelle Ausfuehrungen** — VWAP-Orders am Tagesanfang erzeugen Momentum

Ein Trading-System, das diese Phase systematisch verpasst, laesst die profitabelsten Opportunitaeten ungenutzt.

### 1.4 Diskrepanz Konzept vs. Implementierung

Das bestehende Konzept in `04-llm-integration.md` §6.1 beschreibt bereits volatilitaetsabhaengige Intervalle:

| Volatilitaet | Periodisches LLM-Intervall |
|--------------|---------------------------|
| Hoch (> 120% Daily-ATR) | Alle 3 Minuten |
| Normal (80-120%) | Alle 5 Minuten |
| Niedrig (< 80%) | Alle 15 Minuten |

Diese volatilitaetsadaptive Logik wurde **nie implementiert**. Stattdessen existiert eine statische `TimeBasedSamplingPolicy` mit festem 5-Bar-Intervall. Die Opening-Phase hat konzeptionell gar keine eigene Behandlung.

---

## 2. Loesung: Zwei-Saeulen-Modell (Need-Driven + Phase-Aware)

Das Review hat ergeben, dass eine rein phasenbasierte Sampling-Strategie nur ein Workaround fuer das eigentliche Problem ist. Das Problem ist nicht "zu selten samplen", sondern **"Entry kann ohne frischen LLM-Kontext nicht stattfinden"**. Die Loesung optimiert daher auf **"LLM ist frisch, wenn es zaehlt"**, nicht auf eine fixe Uhr.

### 2.1 Saeule 1: Need-Driven Trigger (primaerer Mechanismus)

Bedarfsgesteuerte LLM-Calls werden durch konkrete Pipeline-Events ausgeloest:

| Trigger | Bedingung | Begruendung |
|---------|-----------|-------------|
| **SESSION_START** | Erster RTH-Snapshot (14:30 UTC / 09:30 EST) | LLM-Kontext muss ab dem ersten Bar verfuegbar sein |
| **ENTRY_CANDIDATE** | Quant-Module liefern ALLOW_ENTRY, aber `lastLlmAnalysis == null` oder TTL abgelaufen | Dual-Key blockiert Entry ohne LLM — sofort nachfordern statt warten |
| **STATE_TRANSITION** | Pipeline-FSM wechselt den State (OBSERVING → POSITIONED etc.) | Taktischer Kontext muss zum neuen State passen |
| **REGIME_CHANGE** | Regime-Hysterese (2× konsekutive Decision-Bars) liefert neues Regime | Regime-Wechsel erfordert neue LLM-Einschaetzung |
| **SIGNIFICANT_MOVE** | Intra-Bar-Bewegung > k × ATR(14) (ATR-normalisiert, siehe 2.3) | Ersetzt die bisherige fixe Prozent-Schwelle |

**Wichtig:** Diese Trigger folgen der bestehenden Single-Flight-Regel (04-llm-integration.md §6.4). Wenn bereits ein LLM-Call laeuft, werden weitere Trigger koalesziert — der naechste Call nach Abschluss des laufenden beruecksichtigt den aktuellsten Snapshot.

**ENTRY_CANDIDATE als Kern-Innovation:** Dies loest das beobachtete Problem direkt. Statt darauf zu hoffen, dass der naechste periodische Call rechtzeitig kommt, wird der LLM-Call **genau dann** ausgeloest, wenn er gebraucht wird. Im IREN-Beispiel: Bei 14:45 UTC meldet Quant ALLOW_ENTRY, das LLM ist nicht frisch → sofortiger LLM-Call → Ergebnis nach ~80s (Backtest) bzw. ~3-5s (Live) → Entry wird freigegeben.

### 2.1a Hard-Constraint: Mindestabstand zwischen LLM-Calls

Unabhaengig von allen Triggern (Need-Driven und Phase-Aware) gilt ein **absoluter Mindestabstand** zwischen zwei aufeinanderfolgenden LLM-Calls. Dieser Constraint ist eine physische Notwendigkeit:

**Backtest-Modus (Claude CLI):** Ein LLM-Call dauert durchschnittlich ~90 Sekunden. Ein Mindestabstand < 300s (5 Minuten) ergibt keinen Sinn, weil:

- Bei 90s Antwortzeit + 300s Mindestabstand ist das LLM zu ~30% der Zeit beschaeftigt — ein gesunder Auslastungsgrad
- Bei 90s + 180s waere die Auslastung ~50%, und die Ergebnisse waeren kaum frischer (nur 2 Bars Unterschied bei 3-Min-Bars)
- Bei 90s + 90s waere das LLM permanent belegt ohne Erholungszeit — jeder Trigger wuerde sofort koalesziert werden, was den Need-Driven-Mechanismus ad absurdum fuehrt

**Live-Modus (Claude API):** Die Antwortzeit sinkt auf ~3-5 Sekunden. Hier kann der Mindestabstand deutlich kuerzer sein, muss aber dennoch existieren, um Rate-Limits und Cost-Overrun zu vermeiden.

**Konfiguration:**

```properties
# Minimum cooldown between two LLM calls (seconds).
# No trigger (need-driven or phase-aware) can fire before this cooldown expires.
# Backtest (CLI, ~90s response): 300s prevents thrashing.
# Live (API, ~3-5s response): can be lower but never 0.
odin.brain.llm.sampling.min-interval-s=300
```

**Auswirkung auf die Trigger-Logik:**

```
shouldSample():
  1. IF timeSinceLastCall < minIntervalS → return false (Cooldown aktiv)
  2. IF needDrivenPolicy.shouldSample() → return true
  3. IF phaseAwarePolicy.shouldSample() → return true
  4. return false
```

Der Cooldown wird in Simulations-Zeit (SimClock) gemessen, nicht in Wanduhr-Zeit. Im Backtest "vergeht" die Simzeit schneller als die Wanduhr, deshalb ist der Cooldown dort effektiv laenger als 300 Wanduhr-Sekunden — das ist gewollt, weil die LLM-Ausfuehrung (CLI) real 90s dauert und damit den limitierenden Faktor darstellt.

**Sonderfall SESSION_START:** Der allererste LLM-Call (SESSION_START) ist vom Cooldown **ausgenommen** — er feuert immer beim ersten RTH-Bar, unabhaengig von vorherigen Calls (es gibt keine vorherigen).

**Sonderfall ENTRY_CANDIDATE:** Wenn ein Entry-Signal vorliegt und der Cooldown aktiv ist, wird der Trigger **nicht verworfen, sondern gequeued**. Sobald der Cooldown ablaeuft, wird der ENTRY_CANDIDATE-Trigger sofort ausgefuehrt. Das garantiert, dass kein Entry-Signal dauerhaft blockiert wird — es wird nur zeitlich verzoegert.

### 2.2 Saeule 2: Phase-Aware Background-Cadence (sekundaerer Mechanismus)

Als Ergaenzung zu den Need-Driven-Triggern sorgt eine phasenbasierte Hintergrund-Cadence fuer **Situational Awareness** — das LLM hat auch dann aktuelle Regime-Einschaetzungen, wenn kein konkretes Entry-Signal vorliegt. Diese Cadence ist die Baseline, die von Need-Driven-Triggern jederzeit uebersteuert werden kann.

| Phase | Zeitfenster (EST) | Cadence (Decision-Bars) | Effektive Frequenz (3-Min-Bars) | Begruendung |
|-------|-------------------|------------------------|--------------------------------|-------------|
| **OPENING** | 09:30 - 10:00 | Alle 2 Bars | ~6 Min (durch Cooldown 300s begrenzt) | Kritischste Phase, Gap-Resolution, OR-Bildung |
| **MORNING** | 10:00 - 11:30 | Alle 3 Bars | ~9 Min | Noch erhoehte Aktivitaet, Trend-Bestaetigung |
| **MIDDAY** | 11:30 - 14:00 | Alle 5 Bars | ~15 Min | Typische Mittagsruhe, niedriges Volumen |
| **POWER_HOUR** | 14:00 - 15:30 | Alle 3 Bars | ~9 Min | MOC-Orders, institutionelle Positionierung |
| **CLOSE** | 15:30 - 16:00 | Alle 2 Bars | ~6 Min (durch Cooldown 300s begrenzt) | EOD-Volatilitaet, Schlussauktion |

**Hinweis:** Die Cadence-Werte interagieren mit dem Mindestabstand (`min-interval-s=300`, siehe §2.1a). Bei 3-Min-Decision-Bars kann "Alle 2 Bars" (= 6 Min) nie unter den 300s-Cooldown fallen. Bei 5-Min-Backtest-Bars ist "Alle 2 Bars" (= 10 Min) ebenfalls sicher ueber dem Minimum. Die Cadence-Werte duerfen kleiner als der Cooldown sein — der Cooldown wirkt als harter Floor und verhindert Ueberbelastung.

**OPENING-Phase ohne Warmup-Delay:** Der erste LLM-Call wird sofort beim ersten RTH-Bar ausgeloest (SESSION_START-Trigger). Es gibt keinen 5-Bar-Warmup in der Opening-Phase.

**Warum kein Warmup noetig:** Die Indikatoren (RSI, VWAP, ATR, EMA) sind bereits durch die Pre-Market-Warmup-Phase der DataPipelineService mit 60-70 Bars vorgeladen. Der bisherige 5-Bar-Warmup der `TimeBasedSamplingPolicy` war eine kuenstliche, redundante Verzoegerung.

**Einschraenkung Warmup-Entfall (ChatGPT-Review):** Premarket-Warmup liefert zwar stabile Momentum-Indikatoren (RSI, EMA, ATR), aber RTH-spezifische Metriken (VWAP, Volume-Ratio, Opening-Range) koennen beim ersten Bar noch unzuverlaessig sein. Das LLM muss diesen Kontext in seiner Analyse beruecksichtigen — der System-Prompt enthaelt die Information "Erster RTH-Bar, Opening-Range noch nicht etabliert".

### 2.3 Event-Schwellen: ATR-normalisiert (nicht Prozent)

Das bestehende Konzept (04-llm-integration.md §10.2) stellt klar: **Fixe Prozent-Schwellen sind in hochvolatilen Regimen systematisch falsch.** Die Event-Schwellen werden daher in ATR-Einheiten formuliert statt in Prozent:

| Phase | Event-Threshold | Aequivalent bei ATR=0.40 |
|-------|----------------|--------------------------|
| OPENING | 0.4 × ATR(14) | ~$0.16 |
| MORNING | 0.6 × ATR(14) | ~$0.24 |
| MIDDAY | 1.0 × ATR(14) | ~$0.40 |
| POWER_HOUR | 0.6 × ATR(14) | ~$0.24 |
| CLOSE | 0.4 × ATR(14) | ~$0.16 |

**Vorteil:** Diese Schwellen skalieren automatisch mit der Volatilitaet des Instruments. Ein hochvolatiler Small-Cap mit ATR=2.00 hat andere absolute Schwellen als ein Large-Cap mit ATR=0.30, aber die relative Sensitivitaet bleibt konsistent.

### 2.4 Kostenbetrachtung

Die erhoehte Frequenz fuehrt zu mehr LLM-Calls, wird aber durch den 300s-Mindestabstand gedeckelt.

**Backtest-Modus (5-Min-Bars, Claude CLI, ~90s pro Call):**

| Phase | Bars | Calls (vorher) | Calls (neu) | Delta | Bemerkung |
|-------|------|-----------------|-------------|-------|-----------|
| OPENING (30 Min) | 6 | 1 | 3 | +2 | Alle 2 Bars = alle 10 Min (> 300s Cooldown) |
| MORNING (90 Min) | 18 | 3-4 | 6 | +2 | Alle 3 Bars = alle 15 Min |
| MIDDAY (150 Min) | 30 | 6 | 6 | 0 | Alle 5 Bars = alle 25 Min |
| POWER_HOUR (90 Min) | 18 | 3-4 | 6 | +2 | Alle 3 Bars = alle 15 Min |
| CLOSE (30 Min) | 6 | 1 | 3 | +2 | Alle 2 Bars = alle 10 Min |
| **Gesamt** | **78** | **~15-18** | **~24** | **+7-9** | |

Zusaetzlich: ~2-4 Need-Driven-Calls (ENTRY_CANDIDATE, STATE_TRANSITION) pro Tag, sofern Cooldown nicht blockiert. **Realistisches Gesamtvolumen: ~26-28 Calls/Tag.**

Bei ~90s pro Call: ~39-42 Min LLM-Zeit. Gesamtlaufzeit steigt von ~24 auf ~30-35 Minuten.

**Live-Modus (3-Min-Bars, Claude API, ~3-5s pro Call):**

| Phase | Bars | Calls (vorher) | Calls (neu) | Delta | Bemerkung |
|-------|------|-----------------|-------------|-------|-----------|
| OPENING (30 Min) | 10 | 2 | 5 | +3 | Alle 2 Bars = alle 6 Min (> 300s Cooldown) |
| MORNING (90 Min) | 30 | 6 | 10 | +4 | Alle 3 Bars = alle 9 Min |
| MIDDAY (150 Min) | 50 | 10 | 10 | 0 | Alle 5 Bars = alle 15 Min |
| POWER_HOUR (90 Min) | 30 | 6 | 10 | +4 | Alle 3 Bars = alle 9 Min |
| CLOSE (30 Min) | 10 | 2 | 5 | +3 | Alle 2 Bars = alle 6 Min |
| **Gesamt** | **130** | **~26** | **~40** | **+14** | |

Zusaetzlich: ~3-5 Need-Driven-Calls pro Tag. **Realistisches Gesamtvolumen: ~43-45 Calls/Tag.** Im Live-Modus koennte `min-interval-s` auf 120s gesetzt werden (API-Latenz nur 3-5s), was hoehere Frequenzen ermoeglicht — aber pragmatisch ist 300s auch hier ein sinnvoller Default.

Bei Claude Sonnet (API): ~$0.003-0.01 pro Call → zusaetzlich $0.04-0.14 pro Tag. Vernachlaessigbar gegenueber dem Handelsvolumen.

**Fazit:** Die Kostensteigerung ist moderat und wird durch den 300s-Cooldown begrenzt. Ein einziger verpasster Entry-Zeitpunkt wie beim IREN-Backtest (102 USD Slippage) uebersteigt die zusaetzlichen API-Kosten um ein Vielfaches.

---

## 3. Architektur

### 3.1 Klassenmodell

```
LlmSamplingPolicy (Interface)
├── TimeBasedSamplingPolicy            // bestehend — unveraendert
├── EventDrivenSamplingPolicy          // bestehend — wird ATR-normalisiert
├── CompositeSamplingPolicy            // bestehend — OR-Logik
├── NeedDrivenSamplingPolicy [NEU]     // ENTRY_CANDIDATE, STATE_TRANSITION, REGIME_CHANGE
├── PhaseAwareSamplingPolicy [NEU]     // Phasen-Cadence (delegiert an Time+Event)
└── AdaptiveSamplingPolicy [NEU]       // Orchestrator: Need-Driven OR Phase-Aware
    ├── NeedDrivenSamplingPolicy
    ├── PhaseAwareSamplingPolicy
    │   ├── MarketPhase (Enum)
    │   ├── PhaseConfig (Record)
    │   └── MarketClock (Dependency)
    └── SESSION_START-Logik (erster RTH-Bar)
```

**Separation of Concerns:**

- `NeedDrivenSamplingPolicy` — reagiert auf Pipeline-Zustand (Entry-Candidate, State-Transition). Erhaelt Zustandsinformation ueber den `MarketSnapshot` (der aktuelle Pipeline-State, aktive Signale etc. enthaelt).
- `PhaseAwareSamplingPolicy` — zeitbasierte Hintergrund-Cadence mit phasenspezifischen Intervallen. Delegiert intern an `TimeBasedSamplingPolicy` + `EventDrivenSamplingPolicy` mit dynamischen Parametern.
- `AdaptiveSamplingPolicy` — OR-Verknuepfung: feuert wenn **einer** der beiden Mechanismen triggert.

### 3.2 Phase-Bestimmung

Die Phase wird aus dem `MarketSnapshot` abgeleitet, nicht aus der Wanduhr. Im Backtest ist die SimClock die einzige Zeitquelle — der Snapshot enthaelt den Bar-Timestamp.

```java
public enum MarketPhase {
    OPENING,      // 09:30:00 - 09:59:59 EST
    MORNING,      // 10:00:00 - 11:29:59 EST
    MIDDAY,       // 11:30:00 - 13:59:59 EST
    POWER_HOUR,   // 14:00:00 - 15:29:59 EST
    CLOSE;        // 15:30:00 - 15:59:59 EST

    public static MarketPhase fromTimestamp(Instant barTimestamp, ZoneId exchangeZone) {
        LocalTime barTime = barTimestamp.atZone(exchangeZone).toLocalTime();
        // ... Zuordnung basierend auf Zeitgrenzen
    }
}
```

**Einschraenkung (ChatGPT-Review):** Phase-by-Uhrzeit ist als Default brauchbar, aber instrument- und tagabhaengig. Small-/Midcaps koennen "Opening Volatility" laenger als 30 Minuten haben. Fruehschluss- und Halttage sprengen jede fixe Einteilung. **Spaetere Erweiterung:** Phase-Ende nicht nur per Uhrzeit, sondern zusaetzlich per Zustand (z.B. "Opening endet wenn OR gebildet + VolRatio normalisiert"). Dies ist nicht Teil der initialen Implementierung, aber als Evolutionspfad vorgesehen.

### 3.3 Konfiguration

Neue Properties in `odin-brain.properties`:

```properties
# === Adaptive LLM Sampling ===

# Hard minimum cooldown between any two LLM calls (seconds).
# Prevents overloading the LLM provider. Applies to ALL triggers.
# Backtest (CLI, ~90s response time): 300s is the practical floor.
# Live (API, ~3-5s response time): can be lowered to 120s if desired.
# SESSION_START is exempt (first call of the day, no prior call to cooldown from).
odin.brain.llm.sampling.min-interval-s=300

# Need-Driven Trigger
odin.brain.llm.sampling.need-driven.enabled=true
odin.brain.llm.sampling.need-driven.session-start=true
odin.brain.llm.sampling.need-driven.entry-candidate=true
odin.brain.llm.sampling.need-driven.state-transition=true
odin.brain.llm.sampling.need-driven.regime-change=true

# Phase-Aware Background-Cadence
# Interval in decision-bars per phase (how often to call the LLM)
odin.brain.llm.sampling.phase.opening.interval-bars=1
odin.brain.llm.sampling.phase.morning.interval-bars=3
odin.brain.llm.sampling.phase.midday.interval-bars=5
odin.brain.llm.sampling.phase.power-hour.interval-bars=3
odin.brain.llm.sampling.phase.close.interval-bars=2

# Event-driven thresholds per phase (ATR-relative, replaces fixed %)
# Threshold = factor × ATR(14). Example: 0.4 × ATR(14)=0.40 → trigger at $0.16 move
odin.brain.llm.sampling.phase.opening.event-threshold-atr-factor=0.4
odin.brain.llm.sampling.phase.morning.event-threshold-atr-factor=0.6
odin.brain.llm.sampling.phase.midday.event-threshold-atr-factor=1.0
odin.brain.llm.sampling.phase.power-hour.event-threshold-atr-factor=0.6
odin.brain.llm.sampling.phase.close.event-threshold-atr-factor=0.4
```

### 3.4 Freshness-Klaerung

Das ChatGPT-Review hat eine Inkonsistenz identifiziert: `entry-freshness-max-s=1800` (30 Min) vs. die normative Regel in 04-llm-integration.md §12: "Entry < 120s".

**Klaerung:** Es gibt zwei verschiedene Freshness-Konzepte:

| Parameter | Wert | Bedeutung |
|-----------|------|-----------|
| `entry-freshness-max-s=1800` | 30 Min | **Maximale Toleranz** fuer den Sampling-Zyklus: Wie alt darf die LLM-Analyse maximal sein, bevor sie komplett verworfen wird (EXPIRED). |
| `freshness.full-max-age-s=120` | 2 Min | **FULL-Freshness**: LLM-Analyse gilt als "frisch" fuer Entry-Zwecke. Innerhalb dieser Schwelle: volle LLM-Confidence-Gewichtung. |
| `freshness.stale-max-age-s=300` | 5 Min | **STALE**: LLM-Analyse wird mit Abschlag (`stale-multiplier=0.70`) gewichtet. |
| `freshness.quant-elevated-max-age-s=600` | 10 Min | **QUANT_ELEVATED**: Quant-Gates werden verschaerft, LLM-Confidence stark reduziert (`0.40`). |

Die Graded-Freshness-Decay (ODIN-077) loest das Problem elegant: Statt hart "frisch oder nicht", gibt es einen Decay-Gradienten. Mit dem neuen adaptiven Sampling sind Opening-Phase-Cycles fast immer im FULL-Bereich, und der ENTRY_CANDIDATE-Trigger sorgt dafuer, dass bei konkretem Bedarf immer ein frischer Call ausgeloest wird.

### 3.5 LLM-Flipping-Risiko bei hoher Frequenz

Das ChatGPT-Review warnt vor taktischem "Flipping" bei Jede-Bar-Sampling: Das LLM koennte bei aufeinanderfolgenden Bars widersprüchliche Einschaetzungen liefern (z.B. `entry_timing_bias` wechselt zwischen ALLOW_NOW und WAIT_PULLBACK).

**Bestehende Mitigationen:**
- `trail_mode` Hysterese: Wechsel nur bei 2 konsekutiven gleichen Werten (04-llm-integration.md §13)
- Regime-Hysterese: 2× konsekutive Decision-Bars fuer Regime-Wechsel (bestehend)

**Zusaetzlich noetig:**
- `entry_timing_bias` Hysterese: analog zu `trail_mode`, Wechsel nur bei 2× konsekutiven gleichen Werten
- `tactic` Hysterese: WAIT_PULLBACK → BREAKOUT_FOLLOW nur bei 2× konsekutiv

Diese Hysterese-Erweiterungen werden als Teil der Implementierung umgesetzt.

### 3.6 Integration mit bestehendem Code

**PipelineFactory-Anpassung:**

```java
// Bisher:
LlmSamplingPolicy samplingPolicy = new CompositeSamplingPolicy(
    new TimeBasedSamplingPolicy(samplingConfig.timeIntervalBars()),
    new EventDrivenSamplingPolicy(samplingConfig.eventThresholdPct()));

// Neu:
LlmSamplingPolicy samplingPolicy = new AdaptiveSamplingPolicy(
    new NeedDrivenSamplingPolicy(needDrivenConfig),
    new PhaseAwareSamplingPolicy(phaseConfig, marketClock, exchangeZone));
```

**TradingPipeline — Anpassung noetig fuer ENTRY_CANDIDATE-Trigger:**

Der `resolveLatestLlmAnalysis()` Aufruf muss den aktuellen Quant-Zustand an die Policy weiterreichen. Entweder ueber den `MarketSnapshot` (der bereits Gate-Ergebnisse enthaelt) oder ueber einen zusaetzlichen Parameter. Die bevorzugte Variante ist ueber den Snapshot, da das `LlmSamplingPolicy`-Interface dann unveraendert bleibt.

**DataPipelineService — keine Aenderung noetig.** Die Snapshot-Dispatch-Logik bleibt identisch.

---

## 4. Decision-Bar-Aufloesung: 3 Minuten als optimaler Kompromiss

### 4.1 Status Quo

| Modus | Decision-Bar | Sampling (`intervalBars=5`) | LLM-Frequenz |
|-------|-----------|-----------------------------|--------------|
| Live | 3 Min | Alle 5 × 3min = 15 Min | ~26 Calls/Tag |
| Backtest | 5 Min | Alle 5 × 5min = 25 Min | ~16 Calls/Tag |

Der Backtest nutzt 5-Minuten-Bars, weil das `BarInterval`-Enum in `odin-backtest` nur `ONE_MINUTE` und `FIVE_MINUTES` kennt. Es fehlt eine `THREE_MINUTES`-Variante.

### 4.2 Empfehlung: 3-Minuten-Bars auch im Backtest

Die 3-Minuten-Bar bietet den besten Kompromiss aus:

- **Aufloesung**: Genuegend Granularitaet, um kurzfristige Moves in der Opening-Phase zu erkennen (5 Min sind zu grob fuer Gap-Resolution)
- **Datendichte**: Weniger Rauschen als 1-Minuten-Bars — einzelne Ticks oder Wicks werden geglaettet
- **Token-Effizienz**: Ein 3-Min-Bar hat ein deutlich besseres Signal-zu-Rausch-Verhaeltnis als ein 1-Min-Bar, bei nur 60% mehr Datenpunkten gegenueber 5-Min
- **Konsistenz Live/Backtest**: Identische Decision-Bar-Aufloesung in beiden Modi vermeidet Regime-Unterschiede

**Erforderliche Aenderung:** `BarInterval`-Enum um `THREE_MINUTES(180)` erweitern und in der Backtest-Pipeline unterstuetzen. Das Daten-Replay muss 3-Minuten-Bars aus den 1-Minuten-Rohdaten aggregieren koennen — die `BarAggregator`-Logik dafuer existiert bereits in der Live-Pipeline.

### 4.3 LLM-Prompt-Anpassung

Das aktuelle LLM-Prompt-Schema (04-llm-integration.md §9.4) liefert dem LLM Bars in verschiedenen Granularitaeten:

| Zeitfenster | Granularitaet |
|-------------|--------------|
| Letzte 15 Minuten | 1-Minuten-Bars |
| Letzte 2 Stunden | 5-Minuten-Bars |
| Rest des Tages | 15-Minuten-Bars |

Unabhaengig von der Decision-Bar-Aufloesung erhaelt das LLM also immer 1-Minuten-Bars fuer den unmittelbaren Kontext. Die Decision-Bar bestimmt nur die **Trigger-Frequenz**, nicht die Datenqualitaet im Prompt.

---

## 5. Implementierungsschritte

### 5.1 Phase 1 — Need-Driven + Phase-Aware Sampling (Kern)

1. `MarketPhase` Enum erstellen (in `odin-brain`)
2. `NeedDrivenSamplingPolicy` implementieren (ENTRY_CANDIDATE, STATE_TRANSITION, REGIME_CHANGE Trigger)
3. `EventDrivenSamplingPolicy` auf ATR-normalisierte Schwellen umstellen
4. `PhaseAwareSamplingPolicy` implementieren (delegiert an bestehende Policies mit phasenspezifischen Parametern)
5. `AdaptiveSamplingPolicy` als Orchestrator (OR-Verknuepfung Need-Driven + Phase-Aware)
6. Config-Records fuer Phase-Properties + Need-Driven-Properties
7. `PipelineFactory` anpassen — neue Policy-Hierarchie
8. Hysterese-Erweiterung fuer `entry_timing_bias` und `tactic`
9. Unit-Tests: Phase-Wechsel, ENTRY_CANDIDATE-Trigger, Boundary-Cases (exakt 10:00:00), Backtest-SimClock vs. Live-Wanduhr, Hysterese-Verhalten

### 5.2 Phase 2 — 3-Minuten-Bars im Backtest

**Vorsicht (ChatGPT-Review):** Decision-Bar-Wechsel veraendert Gate-Statistiken, Indikator-Glaettung, Hold-Duration-Bars, Regime-Hysterese. Das ist eine **Strategieparameterisierung**, nicht nur ein Scheduler-Fix. Alternative: LLM-Sampling von der Decision-Bar entkoppeln (z.B. Sampling auf 1-Min-Snapshots), waehrend Execution/Decision weiterhin auf dem bestehenden Timeframe bleibt.

Falls dennoch gewuenscht:

1. `BarInterval.THREE_MINUTES(180)` zum Enum hinzufuegen
2. Backtest-Replay-Logic anpassen: 1-Min-Bars zu 3-Min-Bars aggregieren
3. `DataPipelineService` Backtest-Pfad auf 3-Min-Decision-Bars umstellen
4. Backtest-Validierung: A/B-Vergleich 3-Min vs. 5-Min ueber mehrere Tage

### 5.3 Phase 3 — Volatilitaetsadaptive Uebersteuerung (04-llm-integration.md §6.1)

Optionale Erweiterung: Die Phase-Intervalle koennen dynamisch durch Volatilitaetsmessung uebersteuert werden (wie in §6.1 des bestehenden Konzepts beschrieben). Bei ATR > 120% Daily-ATR wird die Frequenz um eine Stufe erhoeht, bei ATR < 80% um eine Stufe reduziert. Dies ist eine orthogonale Erweiterung zur Phase-Logik.

### 5.4 Phase 4 — Zustandsbasierte Phase-Grenzen

Langfristige Evolution: Phase-Uebersetzungen nicht nur per Uhrzeit, sondern zusaetzlich per Marktzustand (Opening-Range gebildet, VolRatio normalisiert, etc.). Erfordert einen Zustandsdetektor, der die Opening-Range-Bildung erkennt.

---

## 6. Risiken und Mitigationen

| Risiko | Wahrscheinlichkeit | Mitigation |
|--------|---------------------|-----------|
| Erhoehte LLM-Kosten im Backtest | Mittel | +10-15 Calls/Tag → ~12-15 Min mehr Laufzeit. Akzeptabel. CachedAnalyst fuer Replay nutzen |
| LLM-Latenz im Live-Modus bei Opening | Niedrig | Claude API (nicht CLI) mit ~2-5s Latenz. Single-Flight-Regel verhindert Aufstau |
| Phase-Bestimmung im Backtest falsch | Niedrig | SimClock liefert korrekten Timestamp. Unit-Tests fuer Zeitzonen-Handling |
| LLM-Flipping bei hoher Frequenz | Mittel | Hysterese-Erweiterung fuer `entry_timing_bias` und `tactic` (analog `trail_mode`) |
| ENTRY_CANDIDATE False Positives | Niedrig | ENTRY_CANDIDATE feuert nur wenn Quant tatsaechlich ALLOW_ENTRY liefert und LLM-Analyse fehlt/abgelaufen |
| Overfitting auf Opening-Pattern | Niedrig | Dual-Key + Quant-Gate-Kaskade bleibt aktiv. LLM hat keine unilaterale Entscheidungsgewalt |
| Bar-Resolution-Wechsel (Phase 2) | Mittel | Separater A/B-Test. Alte Ergebnisse als Referenz. Kann auch ganz uebersprungen werden |
| Early Close / Trading Halts | Niedrig | Phase-Logik muss Exchange-Kalender beruecksichtigen (DST via ZoneId bereits gehandhabt, Kalenderlogik ergaenzen) |

---

## 7. Erwarteter Impact

### 7.1 Quantitativ (geschaetzt)

- **Frueherer erster Entry:** Erste LLM-Analyse bereits beim ersten RTH-Bar statt beim fuenften
- **Bedarfsgerechte Entries:** ENTRY_CANDIDATE-Trigger stellt sicher, dass LLM-Analyse genau dann vorliegt, wenn ein Entry-Signal da ist — nicht zufaellig nach Zeitplan
- **Reduzierte Entry-Slippage:** Geschaetzt 30-80 USD pro Tag weniger Slippage bei High-Beta-Aktien durch frueheren Einstieg
- **Bessere Opening-Range-Nutzung:** Gap-and-Go und Gap-and-Fade Setups werden ab dem ersten Bar erkannt

### 7.2 Qualitativ

- **Need-Driven statt zeitplanbasiert:** Das System optimiert auf "LLM ist frisch, wenn es zaehlt" statt auf eine starre Uhr
- **Alignment mit Trading-Realitaet:** Das System respektiert die empirisch belegte U-foermige Intraday-Volatilitaetskurve
- **Konsistenz Konzept/Implementierung:** Die volatilitaetsadaptive Cadence aus 04-llm-integration.md wird endlich realisiert
- **Bessere Nutzung des Pre-Market-Warmups:** Die 60-70 Warmup-Bars werden sofort genutzt statt nochmal 5 Bars zu warten

---

## 8. ChatGPT-Review — Zusammenfassung

**Reviewer:** ChatGPT (2026-03-02), Rolle: Quant-Trader und System-Architekt

### Uebernommene Kritikpunkte

| # | Punkt | Aenderung im Konzept |
|---|-------|---------------------|
| 1 | Phase-basiert ist Workaround, nicht Kernloesung. Need-Driven (Entry-Candidate) ist der primaere Mechanismus | Zwei-Saeulen-Modell: Need-Driven als primaer, Phase-Aware als sekundaer |
| 2 | Event-Schwellen als Prozentwerte widersprechen den eigenen Guardrails (04-llm-integration §10.2) | ATR-normalisierte Schwellen statt fixer Prozent |
| 3 | TTL/Freshness Inkonsistenz (1800s vs. 120s) muss geklaert werden | Klaerung der zwei verschiedenen Freshness-Konzepte in §3.4 |
| 4 | LLM-Flipping-Risiko bei hoher Frequenz | Hysterese-Erweiterung fuer `entry_timing_bias` und `tactic` |
| 5 | Decision-Bar-Wechsel ist ein Strategiewechsel, nicht nur Scheduler-Fix | Phase 2 als optionale Erweiterung mit A/B-Test-Pflicht markiert |
| 6 | Phase-by-Uhrzeit ist instrument-/tagabhaengig | Zustandsbasierte Phase-Grenzen als Phase 4 vorgesehen |
| 7 | Backtest-Oekonomie: CachedAnalyst nutzen | In Risikotabelle aufgenommen |

### Nicht uebernommene Punkte

| # | Punkt | Begruendung |
|---|-------|-------------|
| 1 | "Off-by-one Semantik (Bar-Open vs. Bar-Close)" | Im Code klar definiert: `shouldSample` wird nach Bar-Close aufgerufen. Kein Handlungsbedarf |
| 2 | "Premarket-Warmup verfaelscht VWAP/Volume" | Korrekt, aber durch LLM-System-Prompt adressiert ("Erster RTH-Bar, OR nicht etabliert"). Kein Code-Change noetig |
