# 04 -- LLM-Integration: Tactical Parameter Controller, Schema, Safety, Drift

> **Quellen:** Fachkonzept v1.5 (Kap. 6: LLM-Integration & Prompt-Architektur; Kap. 11: Anti-Halluzination), Master-Konzept v1.1 (LLM-Integration, Tactical Parameter Controller), Build-Spec v3.0 (Kap. 12: LLM-Schema, Prompt-Architektur, Anti-Halluzination, Circuit Breaker), Strategie-Sparring (Bounded Enums, TTL-Regeln, Drift-Guardrails), Stakeholder-Feedback (Provider-Entscheidung Claude/OpenAI, Vision-Modul-Priorisierung)

---

## 1. Architekturprinzip (Stakeholder-Entscheidung)

Das LLM ist KEIN reiner Analyst mehr -- es ist ein **Tactical Parameter Controller**. Es entscheidet nicht BUY/SELL, sondern liefert **gebundene, strukturierte Parameter** (bounded Enums), die deterministisch in Regeln uebersetzt werden.

> **Eiserne Regel:** Kein LLM-Output DARF direkt einen Trigger ausloesen. LLM-Features fliessen in die Rules Engine, die eigenstaendig und deterministisch entscheidet.

> **Kernaussage (Stakeholder-Entscheidung):** ODIN ist KEIN reines Algo-Trading-System. Das LLM MUSS unterstuetzende Entscheidungsgewalt haben -- aber gebunden an deterministische Leitplanken.

**Determinismus bleibt erhalten:** Gleicher Snapshot + gleiche LLM-Response = gleiche Parameter = gleiche Entscheidung. Reproduzierbarkeit in Simulation ueber CachedAnalyst.

---

## 2. Provider-Anbindung (Stakeholder-Entscheidung)

| Provider | SDK/Protokoll | Rolle |
|----------|--------------|-------|
| **Claude** | **Claude Agent SDK** (Java) | Primaer. Strukturierte Analyse via Tool-Use/Structured-Output |
| **OpenAI** | OpenAI API (REST, pay-per-use) | Alternativ. A/B-Tests, Evaluation. **Kein Runtime-Failover** |

Der aktive Provider wird **vor Handelsstart per Konfiguration** gewaehlt und steht fuer den gesamten Handelstag fest. Bei Ausfall greift der Circuit Breaker (Abschnitt 11) → Quant-Only-Modus, KEIN automatischer Provider-Wechsel.

**Backtest LLM Provider (Stakeholder-Entscheidung):** Konfigurierbar CACHED / CLAUDE / OPENAI.

---

## 3. LLM-Analyse-Schema (komplett)

### 3.1 Feldkategorien

| Kategorie | Felder | Nutzung |
|-----------|--------|---------|
| **Decision-Features** | `regime`, `regime_confidence`, `pattern_candidates`, `setup_type`, `tactic`, `urgency_level` | Input fuer Rules Engine und Quant Validation |
| **Kontext-Features** | `opportunity_zones`, `hold_duration_bars` | Optionale Kontextsignale fuer Rules Engine |
| **Tactical Controls (P0)** | `exit_bias`, `trail_mode`, `profit_protection_profile`, `target_policy` | Gebundene Enums fuer taktische Steuerung |
| **Tactical Controls (P1)** | `subregime`, `exhaustion_signal` | Regime-Feinsteuerung |
| **Tactical Controls (P2/P3)** | `entry_timing_bias`, `scale_out_profile` | Erweiterte Steuerung |
| **Logging-Only** | `reasoning`, `key_observations`, `risk_factors`, `market_context_signals`, `explanation_short` | Nur Telemetrie/Logging, NIE in Entscheidungslogik |

### 3.2 Vollstaendiges Feld-Schema

| Feld | Typ | Beschreibung | Kategorie |
|------|-----|-------------|-----------|
| `regime` | Enum: OPENING_VOLATILITY, TREND_UP, TREND_DOWN, RANGE_BOUND, HIGH_VOLATILITY, BREAKOUT_ATTEMPT, EXHAUSTION_RISK, RECOVERY, UNCERTAIN | Regime-Label | Decision |
| `regime_confidence` | Float 0.0--1.0 | Sicherheit der Regime-Einschaetzung | Decision |
| `pattern_candidates` | Object[]: `{pattern: Enum, confidence: Float, phase: String}` (max. 2) | Erkannte Pattern mit Confidence. Pattern-Enum: FLUSH_RECLAIM_RUN, COIL_BREAKOUT, OPENING_CONSOLIDATION, PULLBACK_REENTRY, NONE | Decision |
| `setup_type` | Enum: OPENING_CONSOLIDATION, FLUSH_RECLAIM, COIL_BREAKOUT, PULLBACK_REENTRY, RECOVERY_TRADE, NONE | Erkannter Setup-Typ fuer die aktuelle Marktsituation | Decision |
| `tactic` | Enum: WAIT_PULLBACK, BREAKOUT_FOLLOW, NO_TRADE | Taktische Empfehlung: auf Ruecksetzer warten, Breakout folgen, oder kein Trade | Decision |
| `urgency_level` | Enum: LOW, MEDIUM, HIGH | Zeitkritikalitaet der aktuellen Situation | Decision |
| `opportunity_zones` | Object[]: `{price_min, price_max, type: ENTRY/EXIT, reasoning}` | Preis-Zonen mit erhoehter Chance | Kontext |
| `hold_duration_bars` | Int | Erwartete Haltedauer in 3m-Bars (Referenz-Timeframe) | Kontext |
| `exit_bias` | Enum: HOLD, NEUTRAL, EXIT_SOON, EXIT_NOW | Aktiviert TACTICAL_EXIT-Check | Tactical P0 |
| `trail_mode` | Enum: WIDE, NORMAL, TIGHT | Multiplikator auf base trailingStopAtrFactor | Tactical P0 |
| `profit_protection_profile` | Enum: OFF, STANDARD, AGGRESSIVE | Waehlt R-basierte Stufentabelle | Tactical P0 |
| `target_policy` | Enum: KEEP, CAP_AT_R_MULTIPLE, TRAIL_ONLY | Target-Order-Steuerung | Tactical P0 |
| `subregime` | Enum pro Regime (siehe Abschnitt 7) | Feinere taktische Steuerung | Tactical P1 |
| `exhaustion_signal` | Enum: NONE, EARLY_WARNING, CONFIRMED | Exhaustion-Einschaetzung des LLM | Tactical P1 |
| `entry_timing_bias` | Enum: ALLOW_NOW, WAIT_PULLBACK, WAIT_CONFIRMATION, ALLOW_RE_ENTRY, SKIP | Verschaerft/lockert Entry-Guards | Tactical P2 |
| `scale_out_profile` | Enum: OFF, CONSERVATIVE, STANDARD, AGGRESSIVE | OMS-Steuerung fuer partielle Exits | Tactical P3 |
| `reasoning` | String (max 200 Chars) | Kurze Begruendung fuer Logging | Logging |
| `key_observations` | String[] (max 5) | Beobachtungen, die zur Analyse gefuehrt haben | Logging |
| `risk_factors` | String[] | Erkannte Risiken | Logging |
| `market_context_signals` | String[] (max 5) | Kontextuell relevante Beobachtungen | Logging |
| `explanation_short` | String (max 1--2 Saetze) | Fuer Dashboard-Anzeige | Logging |

### 3.3 Entry-Preise: Deterministische Bestimmung durch OMS

Das LLM liefert **keine** konkreten Entry-Preise, Limit-Preise oder Repricing-Obergrenzen. Entry-Preise werden deterministisch vom OMS bestimmt, basierend auf Struktur-Levels (Pre-Market High/Low, Prior Day Close/High/Low, erste 1m-Bar High/Low). Details siehe 07-oms-execution.md.

Die `opportunity_zones` des LLM dienen ausschliesslich als qualitative Orientierung -- die Rules Engine prueft, ob der aktuelle Preis in einer solchen Zone liegt. Die konkrete Order-Preisbildung ist vollstaendig OMS-Verantwortung.

### 3.4 Regime-Confidence-Schwellen

| Regime-Confidence | Quant-Gate-Anforderung | Verhalten |
|-------------------|------------------------|-----------|
| < 0.5 | -- | Regime gilt als UNCERTAIN, kein Entry |
| 0.5 -- 0.7 | Alle Gates + verschaerfte Schwellen | Erhoehte Quant-Huerde |
| > 0.7 | Alle Gates (Standard-Schwellen) | Standard-Huerde |

---

## 4. Tactical Decision-Features im Detail

### 4.1 P0 -- Kern (bounded Enums)

| Feature | Typ | Interpretation durch Rules Engine |
|---------|-----|----------------------------------|
| `exit_bias` | Enum: HOLD, NEUTRAL, EXIT_SOON, EXIT_NOW | Aktiviert TACTICAL_EXIT-Check. HOLD/NEUTRAL = TACTICAL_EXIT deaktiviert. EXIT_SOON = Exit bei mind. 2 von 4 KPI-Signalen ODER 1 Signal + exhaustion >= EARLY_WARNING. EXIT_NOW = Exit bei mind. 1 Signal + (exhaustion == CONFIRMED ODER MFE >= 1R) |
| `trail_mode` | Enum: WIDE, NORMAL, TIGHT | Multiplikator auf base `trailingStopAtrFactor`: WIDE=1.5x, NORMAL=1.0x, TIGHT=0.75x. Clamp auf `[minTrail, maxTrail]` |
| `profit_protection_profile` | Enum: OFF, STANDARD, AGGRESSIVE | OFF = keine Profit-Protection. STANDARD = R-basierte Stufentabelle ab 1.5R. AGGRESSIVE = Stufentabelle ab 1.0R mit engeren Faktoren |
| `target_policy` | Enum: KEEP, CAP_AT_R_MULTIPLE, TRAIL_ONLY | KEEP = Standard-Targets. CAP_AT_R_MULTIPLE = Target deckeln bei aktuellem R-Multiple. TRAIL_ONLY = alle Targets stornieren, nur Trail |

### 4.2 P1 -- Regime-Feinsteuerung

| Feature | Typ | Interpretation |
|---------|-----|---------------|
| `subregime` | Enum pro Regime (siehe Abschnitt 7) | Beeinflusst Entry-Strenge und Exit-Defaults. KPI primaer, LLM als sekundaerer Hint |
| `exhaustion_signal` | Enum: NONE, EARLY_WARNING, CONFIRMED | Bei EARLY_WARNING: `exit_bias` mind. EXIT_SOON, `trail_mode` mind. TIGHT. Bei CONFIRMED: volle Exhaustion-Logik |

### 4.3 P2/P3 -- Erweiterte Steuerung

| Feature | Typ | Interpretation |
|---------|-----|---------------|
| `entry_timing_bias` | Enum: ALLOW_NOW, WAIT_PULLBACK, WAIT_CONFIRMATION, ALLOW_RE_ENTRY, SKIP | Verschaerft/lockert Entry-Guards. ALLOW_RE_ENTRY aktiviert Multi-Cycle-Day Recovery-Trade-Pfad |
| `scale_out_profile` | Enum: OFF, CONSERVATIVE, STANDARD, AGGRESSIVE | OMS waehlt Tranchen-Profil entsprechend |

---

## 5. Precedence-Kette und Konfliktaufloesung (Stakeholder-Entscheidung)

Die strikte Rangfolge aller Einflussquellen:

1. **Hard-Exits** (KillSwitch, ForcedClose, StopLoss, TrailingStop) -- IMMER zuerst, LLM-unabhaengig
2. **Profit-Protection** setzt Floor der Tightness
3. **LLM `trail_mode`** darf innerhalb der Grenzen variieren, aber nie gegen (2)
4. **LLM `exit_bias`** kann nur Soft-Exit-Schwellen veraendern, nicht Hard-Exits

**Konfliktaufloesung bei Trail-Widerspruch:**

```
effectiveTrailFactor = min(factor_from_llm, factor_from_profit_protection)
```

Es gilt IMMER der engere (konservativere) Wert. Das LLM kann den Trail nie weiter setzen als die Profit-Protection-Stufe vorgibt.

---

## 6. LLM Call-Cadence (Event-Driven + Periodisch)

### 6.1 Periodisches Intervall (Volatilitaets-abhaengig)

| Volatilitaet (ATR vs. Daily-ATR) | Periodisches LLM-Intervall |
|----------------------------------|---------------------------|
| Hoch (> 120% Daily-ATR) | Alle 3 Minuten |
| Normal (80--120%) | Alle 5 Minuten |
| Niedrig (< 80%) | Alle 15 Minuten |

### 6.2 Event-Trigger (zusaetzlich zum periodischen Intervall)

| Trigger | Bedingung |
|---------|-----------|
| Entry-Approaching | Preis betritt eine Opportunity-Zone (Edge-triggered, nicht Level-triggered) oder naehert sich einem OMS-Struktur-Level auf < 0.5x ATR(14). Debounce: 90s zwischen proaktiven Refreshes |
| Signifikante Ereignisse | VWAP-Durchbruch, Volumen-Spike > 2x, Stop-Loss-Annaeherung, Profit-Target erreicht |
| State-Transition | Jeder Zustandswechsel der Pipeline-FSM |
| Crash/Spike Event | CRASH_DOWN oder SPIKE_UP erkannt |
| Exhaustion Climax | EXHAUSTION_CLIMAX auf 1m |
| Bestaetiger Regimewechsel | 10m Confirmation liefert neues Regime |

### 6.3 Freshness-Garantie bei Low-Vol

Das periodische 15-Min-Intervall allein wuerde die Entry-Freshness-Anforderung (< 120s) verletzen. Die Event-Trigger (insbesondere Entry-Approaching) sorgen dafuer, dass bei einer tatsaechlichen Entry-Evaluation IMMER ein frischer LLM-Output vorliegt. Kein Entry ohne frischen Kontext -- auch nicht bei 15-Min-Grundintervall.

### 6.4 Single-Flight-Regel

Nie mehr als ein laufender LLM-Call gleichzeitig pro Pipeline. Weitere Trigger werden koalesziert.

---

## 7. Subregimes

Das 5-stufige Basis-Regime-Modell (TREND_UP, TREND_DOWN, RANGE_BOUND, HIGH_VOLATILITY, UNCERTAIN) wird um Subregimes erweitert (5 Regimes x 4 Subregimes = 20 Zustaende):

| Regime | Subregimes |
|--------|-----------|
| TREND_UP | EARLY, MATURE, LATE, EXHAUSTION |
| TREND_DOWN | RELIEF_RALLY, PULLBACK, ACCELERATION, CAPITULATION |
| RANGE_BOUND | MEAN_REVERT, RANGE_EXPANSION, BREAKOUT_ATTEMPT, CHOP |
| HIGH_VOLATILITY | NEWS_SPIKE, MOMENTUM_EXPANSION, AFTERSHOCK_CHOP, LIQUID_VOL |
| UNCERTAIN | TRANSITION, MIXED_SIGNALS, LOW_LIQUIDITY, DATA_QUALITY |

**Bestimmung:** KPI-Engine primaer, LLM als sekundaerer Hint. Der Resolver waehlt bei Widerspruch zwischen KPI und LLM das konservativere Subregime.

**Wirkung auf Strategie:**

| Subregime-Beispiel | Wirkung |
|--------------------|---------|
| TREND_UP/EARLY | Standard-Entry-Strenge, breiter Trail |
| TREND_UP/LATE | Erhoehte Entry-Strenge, Trail beginnt enger |
| TREND_UP/EXHAUSTION | Entries blockiert, `exit_bias` mind. EXIT_SOON, `trail_mode` mind. TIGHT |
| HIGH_VOLATILITY/NEWS_SPIKE | Entries extrem selektiv, doppelter Spread-Schutz |

---

## 8. Pre-Market Instrument Profiling

In der **WARMUP-Phase** (vor dem ersten Trading) analysiert das LLM die historischen Intraday-Verlaeufe des uebergebenen Instruments (letzte N Handelstage). Ziel ist ein strukturiertes Instrument-Profil, das als **statischer Kontext** in alle Intraday-LLM-Calls des Tages einfliesst.

### 8.1 Auftrag an das LLM

Das LLM erhaelt die historischen Intraday-Daten und identifiziert typische Muster:

| Dimension | Fragestellung |
|-----------|--------------|
| **Gap-Verhalten** | Gap-and-Go vs. Gap-and-Fade -- wie reagiert das Instrument typischerweise auf Overnight-Gaps? |
| **Recovery-Tendenz** | V-Recovery vs. Continuation nach Flush -- neigt das Instrument zu schnellen Erholungen oder setzt es Abwaertsbewegungen fort? |
| **Reversal-Zeitfenster** | In welchen Zeitfenstern treten typische Intraday-Reversals auf? (z.B. 10:00--10:30 ET, Mittagstief) |
| **Volatilitaetsprofil** | Wie entwickelt sich die Volatilitaet ueber den Tag? Fruehe Expansion → Mittagskontraktion → Power-Hour-Expansion? |
| **Trendkontinuitaet** | Neigt das Instrument zu Trendtagen oder Mean-Reversion? |

### 8.2 Output-Schema (Instrument-Profil)

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `gap_behavior` | Enum: GAP_AND_GO, GAP_AND_FADE, MIXED | Typisches Verhalten bei Overnight-Gaps |
| `recovery_tendency` | Enum: HIGH, MODERATE, LOW | Neigung zu V-Recovery nach Flush |
| `typical_reversal_windows` | String[] (max 3) | Zeitfenster mit erhoehter Reversal-Wahrscheinlichkeit (ET) |
| `volatility_profile` | Enum: FRONT_LOADED, EVEN, U_SHAPED, BACK_LOADED | Verteilung der Intraday-Volatilitaet |
| `trend_persistence` | Enum: HIGH, MODERATE, LOW | Neigung zu Trendtagen vs. Mean-Reversion |
| `reasoning` | String (max 200 Chars) | Kurze Begruendung fuer Logging |

### 8.3 Deterministische Validierung durch QuantEngine

Die LLM-Einschaetzungen muessen durch historische Statistik gestuetzt sein. Die QuantEngine berechnet parallel auf Basis derselben historischen Daten quantitative Metriken (z.B. Gap-Fill-Rate, Recovery-Quote nach Flushes, ATR-Verteilung ueber Tageszeiten).

**Bei Widerspruch gilt die quantitative Evidenz.** Beispiel: Wenn das LLM `recovery_tendency = HIGH` einschaetzt, aber die historische V-Recovery-Rate unter 30% liegt, wird das Profil auf `recovery_tendency = LOW` korrigiert.

### 8.4 Verwendung im Tagesverlauf

Das Instrument-Profil wird einmalig in der WARMUP-Phase erstellt und danach nicht mehr aktualisiert. Es fliesst als statischer Kontextblock in jeden Intraday-LLM-Call ein (Bestandteil des System Prompts, siehe Abschnitt 9.1). Die Rules Engine kann Profil-Felder als zusaetzliche Gates verwenden (z.B. `recovery_tendency = LOW` → Setup B (Flush-Reclaim) mit verschaerften Schwellen).

---

## 9. Prompt-Architektur: Dreistufiges Context Window

### 9.1 System Prompt (fest)

- Rollendefinition ("Du bist ein Intraday-Marktanalyst")
- Regeln (Output-Schema, Confidence-Kalibrierung, verbotene Felder)
- Instrument-Profil (Name, Sektor, Beta, durchschnittliche Tagesvolatilitaet)
- **Instrument-Profil aus Pre-Market-Analyse** (Abschnitt 8): Gap-Verhalten, Recovery-Tendenz, Volatilitaetsprofil -- als statischer Kontextblock
- **Safety-Contract:** LLM gibt keine numerischen Preise oder Orderdetails aus -- weder im Freitext noch in strukturierten Feldern. Verboten: Stueckzahlen, Stop-Level, Limit-Preise, Entry-Preise, Target-Preise

### 9.2 Kontext-Fenster (dynamisch)

- Komprimierte Kurshistorie (20d Daily + 5d Intraday)
- Heutiger Tagesplan und Session-Phase
- Bisherige Trades heute und aktuelle Position
- Risk-Budget als kategoriale Klasse (LOW/MED/HIGH), NICHT exakter Kontostand
- ATR-Decay-Ratio als Kontext fuer Tagesverlaufs-Anpassung

### 9.3 Aktueller Tick (pro Aufruf)

- Letzte N Bars (Detaildaten) in komprimierter Form
- Aktuelle Indikatoren (RSI, VWAP, ATR, EMA, ADX)
- Volumen-Anomalien
- Uhrzeit und verbleibende Handelszeit
- Monitor-Events seit letztem Call

### 9.4 Datenkompression fuer Context Window

| Zeitfenster | Granularitaet | Datenpunkte |
|-------------|--------------|-------------|
| Letzte 15 Minuten | 1-Minuten-Bars (OHLCV) | ca. 15 |
| Letzte 2 Stunden | 5-Minuten-Bars (OHLCV) | ca. 24 |
| Rest des Tages | 15-Minuten-Bars (OHLCV) | ca. 16 |
| Letzte 5 Handelstage | 30-Minuten-Bars | ca. 65 |
| Letzte 20 Handelstage | Daily Bars | 20 |

Zusaetzlich vorberechnete Aggregationen: VWAP, kumulatives Volumen-Profil, Intraday-High/Low, aktuelle RSI-Werte auf mehreren Timeframes, ATR-Decay-Ratio, Volume-Ratio.

### 9.5 Datensparsamkeit (Security)

Das LLM erhaelt KEINE:
- Account-Groesse oder persoenliche Daten
- Historische P&L-Werte
- Secrets (API Keys, Broker-Credentials)
- Exakte Kontoinformationen

Risk-Budget wird als kategoriale Klasse (LOW/MED/HIGH) mitgeteilt.

---

## 10. Anti-Halluzination: Fuenf Verteidigungsschichten (Normativ)

### Schicht 1: Schema-Validierung

Jede LLM-Antwort wird gegen das JSON-Schema validiert. Fehlende Felder, falsche Typen → Verwerfung (`LLM_SCHEMA_INVALID`). Ein Retry erlaubt, dann wird das letzte gueltige Assessment weiterverwendet.

### Schicht 2: Plausibilitaetspruefung (ATR-relativ)

| Halluzinationstyp | Erkennung | Aktion |
|-------------------|-----------|--------|
| `hold_duration_bars` > verbleibende Handelszeit | Vergleich mit MarketClock | Verwerfen oder kuerzen |
| `regime_confidence` = 1.0 bei offensichtlich unklarem Markt | Quant-Gegencheck | Confidence auf 0.7 deckeln |
| `opportunity_zones` ausserhalb der heutigen Range + 1x ATR | ATR-erweiterte Range | Verwerfen |
| `tactic` = BREAKOUT_FOLLOW bei RANGE_BOUND ohne Volumen-Signal | Quant-Gegencheck | Auf WAIT_PULLBACK korrigieren |
| `urgency_level` = HIGH bei UNCERTAIN-Regime | Konsistenzcheck | Auf LOW korrigieren |

> **ATR-relative Schwellen:** Fixe Prozent-Schwellen (z.B. "1% vom Kurs") verwerfen in hochvolatilen Regimen systematisch valide Setups. Alle Plausibilitaetsschwellen werden daher in ATR-Einheiten ausgedrueckt.

### Schicht 3: Konsistenzpruefung

LLM-Output wird gegen sich selbst und den Marktkontext geprueft:
- `regime = TREND_DOWN` + `opportunity_zone` Typ ENTRY → Inkonsistent (ausser Aggressive Mode aktiv)
- `regime = UNCERTAIN` + `urgency_level = HIGH` → Inkonsistent (UNCERTAIN impliziert LOW urgency)
- `tactic = BREAKOUT_FOLLOW` + `regime = RANGE_BOUND` ohne Volumen-Signal → Inkonsistent
- `risk_factors` enthalten "starker Abwaertstrend" + `regime = TREND_UP` → Warnung
- LLM labelt `regime = TREND_UP`, wenn 10m Confirmation eindeutig DOWN ist → Auto-Override auf CAUTION
- Confidence darf nicht immer konstant hoch sein → Drift-Alert

### Schicht 4: Quant-Gegenstimme

Die Quant-Validierung (Gate-Kaskade + Hard-Veto-System) dient als unabhaengige Gegenkontrolle:
- Jeder Indikator hat eine harte Ja/Nein-Schwelle (Gate). **Alle** Gates muessen bestanden sein fuer Entry-Freigabe
- Hard-Veto-Gates (z.B. Spread > 0.5%) blockieren unabhaengig von allen anderen Gates
- Kein gewichteter Score, keine Normalisierung -- reine Gate-Kaskade (Details siehe 03-strategy-logic.md)

### Schicht 5: Historische Kalibrierung

Nach einer Einlaufphase von mindestens **100 Handelstagen** wird die Kalibrierung der LLM-Confidence geprueft:
- Ist `regime_confidence = 0.8` tatsaechlich in 80% der Faelle korrekt?
- **Bootstrapping-Methode:** Aus den gesammelten Daten werden per Resampling Konfidenzintervalle fuer die Kalibrierungskurve berechnet
- Systematische Ueber-/Unterschaetzung fuehrt zur Anpassung der Schwellenwerte in der Rules Engine
- Kalibrierungskurve wird nach 100 Handelstagen erstmals berechnet und danach taeglich rollierend aktualisiert

---

## 11. Circuit Breaker und Quant-Only-Modus (Normativ)

### 11.1 Eskalationsstufen

| Stufe | Schwelle | Reaktion | Recovery |
|-------|----------|----------|----------|
| **Per-Call-Timeout** | LLM-Response > 10s | Call abbrechen. Fuer Entries: kein Entry (frischer Kontext noetig). Fuer Management: letzten gueltigen Kontext verwenden (max. 15 Min alt) | Naechster regulaerer Call |
| **Quant-Only** | 3 Failures in Folge (Timeouts oder HTTP 5xx) | Wechsel in Quant-Only-Modus: Keine neuen Entries, bestehende Positionen mit Trailing-Stops managen | Erster erfolgreicher LLM-Call hebt Quant-Only auf |
| **Trade-Halt** | > 30 Min ohne erfolgreichen LLM-Call | Keine neuen Trades mehr, bestehende Positionen bis EOD managen oder bei naechstem Exit-Signal schliessen | Manueller Reset durch Operator |
| **Halluzinations-Kaskade** | 3+ verworfene LLM-Outputs in Folge | Quant-Only-Modus fuer Rest des Tages | Naechster Handelstag |

### 11.2 Worst Case: LLM komplett weg

System schaltet auf Quant-Only:
- Keine neuen Trades (Entry blockiert)
- Bestehende Positionen werden deterministisch gemanagt (Trailing-Stops, Profit-Protection-Stufen)
- EOD wird geschlossen
- Graceful Degradation -- zeitkritische Aktionen sind grundsaetzlich LLM-unabhaengig

---

## 12. TTL/Freshness-Regeln (Normativ)

| Aktion | TTL-Anforderung | Begruendung |
|--------|-----------------|-------------|
| Neuer Entry | Frisch (< 120s seit LLM-Response-Timestamp) | Regime-Kontext ist Voraussetzung fuer Entry |
| Re-Entry (Multi-Cycle) | Frisch (< 120s) | Wie Entry |
| Position-Management | Wuenschenswert (< 900s / 15 Min) | Letzter gueltiger Kontext reicht fuer Trail-Anpassung |
| Regime-Wechsel-Pruefung | Erforderlich | LLM liefert Regime-Diagnose |
| Stop-Loss-Execution | Keine (rein Broker-seitig) | GTC-Stops reagieren ohne System |
| Forced Close | Keine (zeitbasiert) | Deterministisch |
| Kill-Switch | Keine (System-Level) | Notfall |
| Trailing-Stop-Anpassung | Keine (rein Quant/Rules) | ATR-basiert, deterministisch |
| Scaling-Out (Tranchen) | Keine (rein Rules) | R-basierte Trigger |

**TTL-Steuerung nach Volatilitaetsregime:** Bei HIGH_VOLATILITY wird die TTL kuerzer gesetzt. Die genauen Werte sind konfigurierbar (`odin.brain.llm.entry-ttl-sec`, `odin.brain.llm.mgmt-ttl-sec`).

---

## 13. Anti-LLM-Drift Guardrails (Normativ)

Drei Mechanismen verhindern, dass das LLM systematisch zu aggressive oder zu passive Parameter liefert:

| Guardrail | Regel | Begruendung |
|-----------|-------|-------------|
| **R-abhaengige Mindestweiten** | Vor 1.5R ist Trail verboten. 1.5--2R: mindestens 2.5 als Factor | Verhindert vorzeitiges Ausstoppen in fruehen Trade-Phasen |
| **Mode-Hysterese** | `trail_mode` darf nur wechseln, wenn 2 consecutive Bars den gleichen Mode liefern | Verhindert Flipping zwischen WIDE und TIGHT |
| **Monitoring und Auto-Korrektur** | Wenn `pct_tight > 80%` UND `median_R_captured < 0.35R`: LLM-Influence zurueck auf NORMAL | Erkennt systematische Drift und korrigiert automatisch |

---

## 14. News Security und Input Validation (Normativ)

Externe Textdaten (Nachrichtenartikel, Social-Media-Posts, Analystenkommentare) sind potentielle Angriffsvektoren fuer Prompt Injection. Alle externen Texte werden als **untrusted input** behandelt:

| Massnahme | Detail |
|-----------|--------|
| Whitelist-Felder | Nur strukturierte Felder ans LLM: Headline (max 200 Chars), Source, Timestamp, Sentiment-Score |
| Keine Rohtexte | Artikel-Body wird NIEMALS in den LLM-Prompt eingespeist |
| Character-Sanitization | Nur ASCII + Standard-Unicode. Control Characters, Zero-Width-Zeichen, RTL-Override werden entfernt |
| Length Limits | Jedes String-Feld hat striktes Max-Length mit Truncation |
| Prompt-Injection-Schutz | Externe Texte in dediziertem `external_context` JSON-Block, nicht als System/User-Prompt |
| Anomalie-Detection | LLM-Output nach News-Einspeisung signifikant abweichend → verwerfen und ohne News wiederholen |

---

## 15. Model Risk Management (Normativ)

### 15.1 Versionierung

| Artefakt | Schema | Geloggt in |
|----------|--------|-----------|
| `prompt_version` | Semantic Versioning (Major = Schema-Aenderung, Minor = Regel-Aenderung, Patch = Wording) | ConfigSnapshot pro Run |
| `schema_version` | Semantic Versioning | ConfigSnapshot pro Run |
| `strategy_version` | Semantic Versioning | ConfigSnapshot pro Run |
| `cost_model_version` | Semantic Versioning | ConfigSnapshot pro Run |

### 15.2 Deterministische LLM-Settings

| Setting | Wert | Begruendung |
|---------|------|-------------|
| temperature | 0 | Maximale Reproduzierbarkeit |
| top_p | 1.0 | Kein Nucleus-Sampling |
| seed | Fester Wert (konfigurierbar) | Reproduzierbare Ergebnisse bei gleichem Input |
| max_tokens | 2000 | Begrenzt Output-Laenge |

### 15.3 Regression-Gates

Vor Deployment einer neuen Version MUSS:
- Challenger-Suite (20 Szenarien) durchlaufen
- Backtest ueber repraesentative Perioden laufen
- LLM Schema-Compliance 100% (sonst reject)
- Drift Checks: Confidence-Verteilungen, Agreement-Rate
- Latenz p95 < 5 Sekunden

### 15.4 Evaluation-Suites

- **Regressionstests:** 50+ historische Szenarien mit bekanntem Expected-Output. Alle muessen bestehen
- **Halluzinations-Tests:** 10+ Szenarien mit absichtlich widerspruechlichen Daten. LLM muss UNCERTAIN melden
- **Schema-Compliance:** 100% der Outputs muessen valides JSON nach Schema sein
- **Performance-Benchmark:** Latenz p95 < 5 Sekunden

### 15.5 Safety-Contract Tests

Automatisierte Tests stellen sicher, dass LLM:
- Keine numerischen Preise oder Orderdetails ausgibt -- weder im Freitext noch in strukturierten Feldern (Stueckzahlen, Entry-Preise, Target-Preise, Stop-Level, Limit-Preise)
- Keine verbotenen Felder ausgibt
- Bei unklaren Situationen UNCERTAIN/CAUTION waehlt (nicht overconfident)

### 15.6 Change Management

1. Aenderung am Prompt/Schema wird als PR erstellt
2. Paper-Trading-Test (min. 5 Handelstage)
3. Review durch zweite Person (oder automatisierte Evaluation-Suite)
4. Erst nach Approval: Rollout via Konfigurationsschalter (kein Deployment)
5. Bei Anomalien: Sofortiger Rollback auf vorherige Version

### 15.7 Monitoring in Produktion

- LLM Timeout Rate
- Schema Invalid Rate
- Response Latency p95
- Anteil LLM-Vetos
- Performance-Delta Hybrid vs Quant-only (Shadow Mode moeglich)
- Verteilung von `regime_confidence` ueber Zeit (wenn immer 0.9 → Alert)
- Agreement Rate Quant vs LLM
- Impact-Analyse: wie oft LLM-Veto Trades verhindert, wie oft es Exits beschleunigt

---

## 16. LLM-Response-Caching fuer deterministischen Replay (Normativ)

Alle LLM-Responses werden gecacht:

- **Cache-Key:** Hash aus (System-Prompt-Version + Schema-Version + Kontext-Fenster + aktuellem Tick-Payload)
- **Speicherung:** Jeder LLM-Call wird mit Input + Output + Model-ID + Timestamp persistiert
- **Replay-Modus:** Im Backtesting MUSS der CachedAnalyst verwendet werden fuer deterministische Reproduzierbarkeit
- **Cache-Invalidierung:** Bei Prompt-Version-Wechsel (Major/Minor) wird der Cache fuer die neue Version neu aufgebaut
- **Einschraenkung:** temp=0 + seed garantieren nicht 100% Reproduzierbarkeit ueber Model-Updates hinweg. Der Response-Cache ist die einzige zuverlaessige Methode fuer exakte Replays

---

## 17. Optionales Vision-Modul (Ausblick)

Ergaenzend zur zahlenbasierten Analyse ist ein Vision-Modul denkbar, das Chart-Bilder analysiert:
- **Einsatz:** Vision-Modell erkennt ausschliesslich Pattern-Labels (Triangle, Pennant, Flag, Double-Bottom). Keine Kurslevels, keine Handlungsempfehlungen
- **Order-Trigger bleibt zahlenbasiert:** Breakout-Level, ATR, Stops werden weiterhin aus numerischen Daten berechnet
- **Priorisierung:** Erst nach erfolgreicher Validierung des zahlenbasierten Kerns. Vision-Modul ist kein MVP-Bestandteil
