# 04 -- LLM-Integration: Tactical Parameter Controller, Schema, Safety, Drift

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

Der aktive Provider wird **vor Handelsstart per Konfiguration** gewaehlt und steht fuer den gesamten Handelstag fest. Bei Ausfall greift der Circuit Breaker (Abschnitt 10) → Quant-Only-Modus, KEIN automatischer Provider-Wechsel.

**Backtest LLM Provider (Stakeholder-Entscheidung):** Konfigurierbar CACHED / CLAUDE / OPENAI.

---

## 3. LLM-Analyse-Schema (komplett)

### 3.1 Feldkategorien

| Kategorie | Felder | Nutzung |
|-----------|--------|---------|
| **Decision-Features** | `regime`, `regime_confidence`, `pattern_candidates` | Input fuer Rules Engine und Quant Validation |
| **Kontext-Features** | `opportunity_zones`, `entry_price_zone`, `target_price`, `hold_duration_bars`, `urgency_level` | Optionale Kontextsignale fuer Rules Engine |
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
| `opportunity_zones` | Object[]: `{price_min, price_max, type: ENTRY/EXIT, reasoning}` | Preis-Zonen mit erhoehter Chance | Kontext |
| `entry_price_zone` | Object: `{min, ideal, max}` | Konkreter Preisbereich fuer Entry. `ideal` = Limit-Preis, `max` = Obergrenze fuer Repricing | Kontext |
| `target_price` | Float oder null | Optionales Gewinnziel (LLM-Schaetzung, wird von Rules validiert) | Kontext |
| `hold_duration_bars` | Int | Erwartete Haltedauer in 3m-Bars (Referenz-Timeframe) | Kontext |
| `urgency_level` | Enum: LOW, MEDIUM, HIGH, CRITICAL | Zeitkritikalitaet der aktuellen Situation | Kontext |
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

### 3.3 Beziehung zwischen opportunity_zones und entry_price_zone

`entry_price_zone` ist die **Konkretisierung** einer ENTRY-typed `opportunity_zone` auf einen handelbaren Preisbereich:
- Die Rules Engine prueft, ob der aktuelle Preis in einer `opportunity_zone` liegt
- Die Execution nutzt `entry_price_zone` fuer den Order-Preis
- `entry_price_zone.ideal` = Limit-Preis
- `entry_price_zone.max` = harte Obergrenze fuer Repricing

### 3.4 Regime-Confidence-Schwellen

| Regime-Confidence | Quant-Score-Schwelle | Verhalten |
|-------------------|---------------------|-----------|
| < 0.5 | -- | Regime gilt als UNCERTAIN, kein Entry |
| 0.5 -- 0.7 | >= 0.70 | Erhoehte Quant-Huerde |
| > 0.7 | >= 0.50 | Standard-Huerde |

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
| Entry-Approaching | Preis betritt eine Opportunity-Zone (Edge-triggered, nicht Level-triggered) oder naehert sich `entry_price_zone` auf < 0.5x ATR(14). Debounce: 90s zwischen proaktiven Refreshes |
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

## 8. Prompt-Architektur: Dreistufiges Context Window

### 8.1 System Prompt (fest)

- Rollendefinition ("Du bist ein Intraday-Marktanalyst")
- Regeln (Output-Schema, Confidence-Kalibrierung, verbotene Felder)
- Instrument-Profil (Name, Sektor, Beta, durchschnittliche Tagesvolatilitaet)
- Verbotene Outputs: keine Orderdetails (Stueckzahlen, Stop-Level, Limit-Preise)

### 8.2 Kontext-Fenster (dynamisch)

- Komprimierte Kurshistorie (20d Daily + 5d Intraday)
- Heutiger Tagesplan und Session-Phase
- Bisherige Trades heute und aktuelle Position
- Risk-Budget als kategoriale Klasse (LOW/MED/HIGH), NICHT exakter Kontostand
- ATR-Decay-Ratio als Kontext fuer Tagesverlaufs-Anpassung

### 8.3 Aktueller Tick (pro Aufruf)

- Letzte N Bars (Detaildaten) in komprimierter Form
- Aktuelle Indikatoren (RSI, VWAP, ATR, EMA, ADX)
- Volumen-Anomalien
- Uhrzeit und verbleibende Handelszeit
- Monitor-Events seit letztem Call

### 8.4 Datenkompression fuer Context Window

| Zeitfenster | Granularitaet | Datenpunkte |
|-------------|--------------|-------------|
| Letzte 15 Minuten | 1-Minuten-Bars (OHLCV) | ca. 15 |
| Letzte 2 Stunden | 5-Minuten-Bars (OHLCV) | ca. 24 |
| Rest des Tages | 15-Minuten-Bars (OHLCV) | ca. 16 |
| Letzte 5 Handelstage | 30-Minuten-Bars | ca. 65 |
| Letzte 20 Handelstage | Daily Bars | 20 |

Zusaetzlich vorberechnete Aggregationen: VWAP, kumulatives Volumen-Profil, Intraday-High/Low, aktuelle RSI-Werte auf mehreren Timeframes, ATR-Decay-Ratio, Volume-Ratio.

### 8.5 Datensparsamkeit (Security)

Das LLM erhaelt KEINE:
- Account-Groesse oder persoenliche Daten
- Historische P&L-Werte
- Secrets (API Keys, Broker-Credentials)
- Exakte Kontoinformationen

Risk-Budget wird als kategoriale Klasse (LOW/MED/HIGH) mitgeteilt.

---

## 9. Anti-Halluzination: Fuenf Verteidigungsschichten (Normativ)

### Schicht 1: Schema-Validierung

Jede LLM-Antwort wird gegen das JSON-Schema validiert. Fehlende Felder, falsche Typen → Verwerfung (`LLM_SCHEMA_INVALID`). Ein Retry erlaubt, dann wird das letzte gueltige Assessment weiterverwendet.

### Schicht 2: Plausibilitaetspruefung (ATR-relativ)

| Halluzinationstyp | Erkennung | Aktion |
|-------------------|-----------|--------|
| `entry_price_zone` weicht > 1.5x ATR(14) vom aktuellen Kurs ab | ATR-relative Pruefung | Verwerfen |
| `target_price` unrealistisch (> 4x ATR(14) vom Entry) | ATR-relative Pruefung | Target auf 2x ATR begrenzen |
| `hold_duration_bars` > verbleibende Handelszeit | Vergleich mit MarketClock | Verwerfen oder kuerzen |
| `regime_confidence` = 1.0 bei offensichtlich unklarem Markt | Quant-Gegencheck | Confidence auf 0.7 deckeln |
| `opportunity_zones` ausserhalb der heutigen Range + 1x ATR | ATR-erweiterte Range | Verwerfen |

> **ATR-relative Schwellen:** Fixe Prozent-Schwellen (z.B. "1% vom Kurs") verwerfen in hochvolatilen Regimen systematisch valide Setups. Alle Plausibilitaetsschwellen werden daher in ATR-Einheiten ausgedrueckt.

### Schicht 3: Konsistenzpruefung

LLM-Output wird gegen sich selbst und den Marktkontext geprueft:
- `regime = TREND_DOWN` + `opportunity_zone` Typ ENTRY → Inkonsistent (ausser Aggressive Mode aktiv)
- `regime = UNCERTAIN` + `urgency = HIGH` → Inkonsistent (UNCERTAIN impliziert LOW/MEDIUM urgency)
- `risk_factors` enthalten "starker Abwaertstrend" + `regime = TREND_UP` → Warnung
- LLM labelt `regime = TREND_UP`, wenn 10m Confirmation eindeutig DOWN ist → Auto-Override auf CAUTION
- Confidence darf nicht immer konstant hoch sein → Drift-Alert

### Schicht 4: Quant-Gegenstimme

Die Quant-Validierung (Scoring-Modell + Hard-Veto-System) dient als unabhaengige Gegenkontrolle:
- `quant_score >= 0.50` → Trade grundsaetzlich erlaubt
- Hard-Veto-Checks (z.B. Spread > 0.5%) blockieren unabhaengig vom Gesamtscore
- Einzelne Checks haben ein Hard-Veto-Recht unabhaengig vom LLM-Output

### Schicht 5: Historische Kalibrierung

Nach einer Einlaufphase von mindestens **100 Handelstagen** wird die Kalibrierung der LLM-Confidence geprueft:
- Ist `regime_confidence = 0.8` tatsaechlich in 80% der Faelle korrekt?
- **Bootstrapping-Methode:** Aus den gesammelten Daten werden per Resampling Konfidenzintervalle fuer die Kalibrierungskurve berechnet
- Systematische Ueber-/Unterschaetzung fuehrt zur Anpassung der Schwellenwerte in der Rules Engine
- Kalibrierungskurve wird nach 100 Handelstagen erstmals berechnet und danach taeglich rollierend aktualisiert

---

## 10. Circuit Breaker und Quant-Only-Modus (Normativ)

### 10.1 Eskalationsstufen

| Stufe | Schwelle | Reaktion | Recovery |
|-------|----------|----------|----------|
| **Per-Call-Timeout** | LLM-Response > 10s | Call abbrechen. Fuer Entries: kein Entry (frischer Kontext noetig). Fuer Management: letzten gueltigen Kontext verwenden (max. 15 Min alt) | Naechster regulaerer Call |
| **Quant-Only** | 3 Failures in Folge (Timeouts oder HTTP 5xx) | Wechsel in Quant-Only-Modus: Keine neuen Entries, bestehende Positionen mit Trailing-Stops managen | Erster erfolgreicher LLM-Call hebt Quant-Only auf |
| **Trade-Halt** | > 30 Min ohne erfolgreichen LLM-Call | Keine neuen Trades mehr, bestehende Positionen bis EOD managen oder bei naechstem Exit-Signal schliessen | Manueller Reset durch Operator |
| **Halluzinations-Kaskade** | 3+ verworfene LLM-Outputs in Folge | Quant-Only-Modus fuer Rest des Tages | Naechster Handelstag |

### 10.2 Worst Case: LLM komplett weg

System schaltet auf Quant-Only:
- Keine neuen Trades (Entry blockiert)
- Bestehende Positionen werden deterministisch gemanagt (Trailing-Stops, Profit-Protection-Stufen)
- EOD wird geschlossen
- Graceful Degradation -- zeitkritische Aktionen sind grundsaetzlich LLM-unabhaengig

---

## 11. TTL/Freshness-Regeln (Normativ)

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

## 12. Anti-LLM-Drift Guardrails (Normativ)

Drei Mechanismen verhindern, dass das LLM systematisch zu aggressive oder zu passive Parameter liefert:

| Guardrail | Regel | Begruendung |
|-----------|-------|-------------|
| **R-abhaengige Mindestweiten** | Vor 1.5R ist Trail verboten. 1.5--2R: mindestens 2.5 als Factor | Verhindert vorzeitiges Ausstoppen in fruehen Trade-Phasen |
| **Mode-Hysterese** | `trail_mode` darf nur wechseln, wenn 2 consecutive Bars den gleichen Mode liefern | Verhindert Flipping zwischen WIDE und TIGHT |
| **Monitoring und Auto-Korrektur** | Wenn `pct_tight > 80%` UND `median_R_captured < 0.35R`: LLM-Influence zurueck auf NORMAL | Erkennt systematische Drift und korrigiert automatisch |

---

## 13. News Security und Input Validation (Normativ)

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

## 14. Model Risk Management (Normativ)

### 14.1 Versionierung

| Artefakt | Schema | Geloggt in |
|----------|--------|-----------|
| `prompt_version` | Semantic Versioning (Major = Schema-Aenderung, Minor = Regel-Aenderung, Patch = Wording) | ConfigSnapshot pro Run |
| `schema_version` | Semantic Versioning | ConfigSnapshot pro Run |
| `strategy_version` | Semantic Versioning | ConfigSnapshot pro Run |
| `cost_model_version` | Semantic Versioning | ConfigSnapshot pro Run |

### 14.2 Deterministische LLM-Settings

| Setting | Wert | Begruendung |
|---------|------|-------------|
| temperature | 0 | Maximale Reproduzierbarkeit |
| top_p | 1.0 | Kein Nucleus-Sampling |
| seed | Fester Wert (konfigurierbar) | Reproduzierbare Ergebnisse bei gleichem Input |
| max_tokens | 2000 | Begrenzt Output-Laenge |

### 14.3 Regression-Gates

Vor Deployment einer neuen Version MUSS:
- Challenger-Suite (20 Szenarien) durchlaufen
- Backtest ueber repraesentative Perioden laufen
- LLM Schema-Compliance 100% (sonst reject)
- Drift Checks: Confidence-Verteilungen, Agreement-Rate
- Latenz p95 < 5 Sekunden

### 14.4 Evaluation-Suites

- **Regressionstests:** 50+ historische Szenarien mit bekanntem Expected-Output. Alle muessen bestehen
- **Halluzinations-Tests:** 10+ Szenarien mit absichtlich widerspruechlichen Daten. LLM muss UNCERTAIN melden
- **Schema-Compliance:** 100% der Outputs muessen valides JSON nach Schema sein
- **Performance-Benchmark:** Latenz p95 < 5 Sekunden

### 14.5 Safety-Contract Tests

Automatisierte Tests stellen sicher, dass LLM:
- Keine Orderdetails ausgibt (Stueckzahlen, Preise, Stop-Level)
- Keine verbotenen Felder ausgibt
- Bei unklaren Situationen UNCERTAIN/CAUTION waehlt (nicht overconfident)

### 14.6 Change Management

1. Aenderung am Prompt/Schema wird als PR erstellt
2. Paper-Trading-Test (min. 5 Handelstage)
3. Review durch zweite Person (oder automatisierte Evaluation-Suite)
4. Erst nach Approval: Rollout via Konfigurationsschalter (kein Deployment)
5. Bei Anomalien: Sofortiger Rollback auf vorherige Version

### 14.7 Monitoring in Produktion

- LLM Timeout Rate
- Schema Invalid Rate
- Response Latency p95
- Anteil LLM-Vetos
- Performance-Delta Hybrid vs Quant-only (Shadow Mode moeglich)
- Verteilung von `regime_confidence` ueber Zeit (wenn immer 0.9 → Alert)
- Agreement Rate Quant vs LLM
- Impact-Analyse: wie oft LLM-Veto Trades verhindert, wie oft es Exits beschleunigt

---

## 15. LLM-Response-Caching fuer deterministischen Replay (Normativ)

Alle LLM-Responses werden gecacht:

- **Cache-Key:** Hash aus (System-Prompt-Version + Schema-Version + Kontext-Fenster + aktuellem Tick-Payload)
- **Speicherung:** Jeder LLM-Call wird mit Input + Output + Model-ID + Timestamp persistiert
- **Replay-Modus:** Im Backtesting MUSS der CachedAnalyst verwendet werden fuer deterministische Reproduzierbarkeit
- **Cache-Invalidierung:** Bei Prompt-Version-Wechsel (Major/Minor) wird der Cache fuer die neue Version neu aufgebaut
- **Einschraenkung:** temp=0 + seed garantieren nicht 100% Reproduzierbarkeit ueber Model-Updates hinweg. Der Response-Cache ist die einzige zuverlaessige Methode fuer exakte Replays

---

## 16. Optionales Vision-Modul (Ausblick)

Ergaenzend zur zahlenbasierten Analyse ist ein Vision-Modul denkbar, das Chart-Bilder analysiert:
- **Einsatz:** Vision-Modell erkennt ausschliesslich Pattern-Labels (Triangle, Pennant, Flag, Double-Bottom). Keine Kurslevels, keine Handlungsempfehlungen
- **Order-Trigger bleibt zahlenbasiert:** Breakout-Level, ATR, Stops werden weiterhin aus numerischen Daten berechnet
- **Priorisierung:** Erst nach erfolgreicher Validierung des zahlenbasierten Kerns. Vision-Modul ist kein MVP-Bestandteil
