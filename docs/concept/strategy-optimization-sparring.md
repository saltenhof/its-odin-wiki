# Strategieoptimierung & LLM-Integration — Sparring-Ergebnisse

Ergebnisse des ChatGPT-Sparrings zu generischer Algo-Optimierung und LLM-Integration als Tactical Parameter Controller — Februar 2026

---

## Inhaltsverzeichnis

1. [Executive Summary](#1-executive-summary)
2. [Ausgangslage](#2-ausgangslage)
3. [Teil 1: Algo-Tuning — Generische Profit-Protection](#3-teil-1-algo-tuning-generische-profit-protection)
4. [Teil 2: LLM als Tactical Parameter Controller](#4-teil-2-llm-als-tactical-parameter-controller)
5. [Teil 3: TACTICAL_EXIT und Exhaustion-Detection](#5-teil-3-tactical_exit-und-exhaustion-detection)
6. [Teil 4: Implementierungs-Roadmap](#6-teil-4-implementierungs-roadmap)
7. [Teil 5: Multi-Cycle-Day-Strategie](#7-teil-5-multi-cycle-day-strategie)
8. [Appendix: User-Entscheidungen](#8-appendix-user-entscheidungen)
9. [Relation zum Fachkonzept v1.5](#9-relation-zum-fachkonzept-v15)

---

## 1. Executive Summary

Dieses Dokument fasst die Ergebnisse eines strukturierten Sparrings zusammen, das drei fundamentale Weiterentwicklungen fuer ODIN erarbeitet hat:

1. **R-basierte Profit-Protection:** Ein mehrstufiges, deterministisches Trail-System, das Gewinne proportional zum erreichten R-Multiple absichert. Ersetzt den bisherigen einstufigen Trailing-Stop durch eine dynamische Stufentabelle mit Regime-Anpassung.

2. **LLM als Tactical Parameter Controller:** Evolution der LLM-Rolle vom reinen Analyst zum Parameter-Lieferanten. Das LLM entscheidet nicht mehr BUY/SELL, sondern liefert gebundene, strukturierte Parameter (exit_bias, trail_mode, etc.), die deterministisch in Regeln uebersetzt werden. Dies bewahrt die Reproduzierbarkeit bei gleichzeitig adaptiver Intelligenz.

3. **Multi-Cycle-Day-Strategie:** Paradigmenwechsel vom Single-Trade-Day (ein Entry/Exit-Zyklus) zu mehreren Zyklen pro Tag auf demselben Instrument. Das LLM erkennt Plateau/Exhaustion fuer den proaktiven Exit und identifiziert Recovery-Signale fuer den Re-Entry — Entscheidungen, die ein statischer Algorithmus nicht treffen kann.

Alle drei Konzepte sind komplementaer: Die Profit-Protection bildet den deterministischen Floor, den das LLM nicht unterschreiten kann. Das LLM kann innerhalb dieses Floors taktisch agieren — aber niemals gegen die Sicherheitsmechanismen. Multi-Cycle-Day baut auf TACTICAL_EXIT und Exhaustion-Detection auf und ist die logische Konsequenz der erweiterten LLM-Integration.

> **Kernaussage:** ODIN ist kein reines Algo-Trading-System. Das LLM muss unterstuetzende Entscheidungsgewalt haben — aber gebunden an deterministische Leitplanken.

---

## 2. Ausgangslage

### Anlass: IREN-Case

Die Analyse eines konkreten Trades (IREN) zeigte typische Schwaechen des bisherigen Exit-Managements:

- **Einstufiger Trailing-Stop:** Gleicher ATR-Faktor in allen Marktphasen und Profit-Leveln. Kein Tightening bei hohen Gewinnen, kein Lockern in fruehen Trend-Phasen.
- **Fehlende Profit-Protection:** Bei schnellen Reversals nach starken Moves ging ein signifikanter Teil des unrealisierten Gewinns verloren.
- **LLM ohne Einfluss auf Trail-Parameter:** Die LLM-Analyse erkannte zwar Exhaustion-Signale, konnte aber keine Trail-Anpassung ausloesen.

Diese Beobachtungen fuehrten zu zwei paralellen Arbeitsstroemen: generisches Algo-Tuning (unabhaengig vom LLM) und eine erweiterte LLM-Integration.

---

## 3. Teil 1: Algo-Tuning — Generische Profit-Protection

### 3.1 R-basierte Profit-Protection Stufentabelle

Der Kern des neuen Algo-Tunings ist eine R-basierte Stufentabelle, die den Trailing-Stop progressiv enger zieht, je weiter der Trade im Gewinn liegt. R bezeichnet das Vielfache des initialen Risikos (Distanz Entry zu Initial Stop).

| R-Schwelle | Stop-Typ | Trail-ATR-Factor | MFE-Lock % |
|------------|----------|-----------------|------------|
| < 1.0R | Nur Initial Stop (`stop0`) | -- | -- |
| >= 1.0R | Break-even: stop >= entry + 0.10R (Puffer fuer Fees) | -- | -- |
| >= 1.5R | Trail aktiv (wide) | 2.75 | -- |
| >= 2.0R | Trail enger + MFE-Lock | 2.00 | 35% |
| >= 3.0R | Aggressiver Trail + Lock | 1.25 | 60% |
| >= 4.0R | Final Protect | 1.00 | 75% |

**Berechnung des effektiven Stops:**

```
effectiveStop = max(prevStop, stop0, trailCandidate, mfeLockStop)
```

Der effektive Stop kann nur steigen, nie fallen (Highwater-Mark-Prinzip). `mfeLockStop` sichert einen Mindestprozentsatz des bisher erreichten Maximum Favorable Excursion (MFE) ab.

### 3.2 Regime-spezifische Trail-Weite

Die Trail-Weite (ATR-Faktor `k`) wird zusaetzlich an das aktuelle Marktregime angepasst:

| Regime | Trail-Faktor `k` | Begruendung |
|--------|------------------|-------------|
| TREND_UP stark (ADX hoch) | 2.0 - 3.0 | Breiter Trail laesst dem Trend Luft |
| TREND_UP schwach / Chop | 1.2 - 2.0 | Engerer Trail, da Trendkontinuation unsicher |
| HIGH_VOLATILITY | Initial 3.0, dann Umschaltung per Profit-Protection-Stufen | Anfangs breit gegen Noise, dann progressives Tightening |

### 3.3 ADX als Entry-Filter

Die EntryRules pruefen TREND_UP-Entries aktuell nicht auf ADX-Staerke. Ein expliziter ADX-Filter wuerde schwache, unzuverlaessige Trends herausfiltern:

- **ADX > Schwellwert** (z.B. 20-25): Trend ist statistisch signifikant, Entry erlaubt
- **ADX < Schwellwert:** Trend ist schwach oder nicht vorhanden, Entry wird verweigert

Dies reduziert die Anzahl der Trades in Seitwaertsphasen, die faelschlicherweise als Trend identifiziert werden.

### 3.4 Sensitivitaetsanalyse: Plateau-Test

Fuer die kritischen Parameter `stopLossAtrFactor`, `trailingStopAtrFactor` und `vwapOffsetAtrFactor` soll ein systematischer Plateau-Test durchgefuehrt werden:

- **Methode:** Jeden Parameter um +/- 10% variieren und die Performance-Aenderung messen.
- **Plateau = robust:** Wenn die Performance-Kurve flach ist, ist der Parameter-Wert stabil und nicht ueberoptimiert.
- **Nadel = Overfit:** Wenn die Performance bei minimaler Aenderung stark einbricht, ist der Wert fragil und wahrscheinlich an historische Daten ueberangepasst.

---

## 4. Teil 2: LLM als Tactical Parameter Controller

### 4.1 Kernkonzept

Das LLM wird vom reinen "Analyst" zum **Tactical Parameter Controller** weiterentwickelt. Es entscheidet nicht "BUY/SELL", sondern liefert **gebundene, strukturierte Parameter**, die deterministisch in Regeln uebersetzt werden.

> **Determinismus bleibt erhalten:** Gleicher Snapshot + gleiche LLM-Response = gleiche Parameter = gleiche Entscheidung. Reproduzierbarkeit in Simulation ueber CachedAnalyst.

### 4.2 Decision-Features P0 (Kern)

| Feature | Typ | Interpretation durch Rules Engine |
|---------|-----|----------------------------------|
| `exit_bias` | Enum: `HOLD`, `NEUTRAL`, `EXIT_SOON`, `EXIT_NOW` | Aktiviert TACTICAL_EXIT-Check. `EXIT_SOON` = Exit bei mind. 2 von 4 KPI-Signalen. `EXIT_NOW` = Exit bei mind. 1 Signal + Exhaustion |
| `trail_mode` | Enum: `WIDE`, `NORMAL`, `TIGHT` | Multiplikator auf base `trailingStopAtrFactor`: WIDE=1.5x, NORMAL=1.0x, TIGHT=0.75x. Clamp auf `[minTrail, maxTrail]` |
| `profit_protection_profile` | Enum: `OFF`, `STANDARD`, `AGGRESSIVE` | Waehlt R-basierte Stufentabelle. STANDARD = ab 2R tighten. AGGRESSIVE = ab 1R tighten |
| `target_policy` | Enum: `KEEP`, `CAP_AT_R_MULTIPLE`, `TRAIL_ONLY` | KEEP = Status quo. CAP = Target deckeln. TRAIL_ONLY = Target ignorieren, Trail dominiert |

### 4.3 Decision-Features P1 (Regime-Feinsteuerung)

| Feature | Typ | Interpretation |
|---------|-----|---------------|
| `subregime` | Enum pro Regime (z.B. TREND_UP: `EARLY`, `MATURE`, `LATE`, `EXHAUSTION`) | Beeinflusst Entry-Strenge und Exit-Defaults |
| `exhaustion_signal` | Enum: `NONE`, `EARLY_WARNING`, `CONFIRMED` | Schaltet Protect-Policy: `exit_bias` mind. `EXIT_SOON`, `trail_mode` mind. `TIGHT` |

### 4.4 Decision-Features P2/P3 (spaeter)

| Feature | Typ | Interpretation |
|---------|-----|---------------|
| `entry_timing_bias` | Enum: `ALLOW_NOW`, `WAIT_PULLBACK`, `WAIT_CONFIRMATION`, `SKIP` | Verschaerft/lockert Entry-Guards |
| `scale_out_profile` | Enum: `OFF`, `CONSERVATIVE`, `STANDARD`, `AGGRESSIVE` | OMS-Steuerung fuer partielle Exits |

### 4.5 Precedence-Kette und Konfliktaufloesung

Die Precedence-Kette definiert die strikte Rangfolge aller Einflussquellen:

1. **Hard-Exits** (KillSwitch, ForcedClose, StopLoss, TrailingStop) — IMMER zuerst, LLM-unabhaengig
2. **Profit-Protection** setzt Floor der Tightness
3. **LLM `trail_mode`** darf innerhalb der Grenzen variieren, aber nie gegen (2)
4. **LLM `exit_bias`** kann nur Soft-Exit-Schwellen veraendern, nicht Hard-Exits

**Konfliktaufloesung bei Trail-Widerspruch:**

```
effectiveTrailFactor = min(factor_from_llm, factor_from_profit_protection)
```

Es gilt immer der engere (konservativere) Wert. Das LLM kann den Trail nie weiter setzen als die Profit-Protection-Stufe vorgibt.

### 4.6 Anti-LLM-Drift Guardrails

Drei Mechanismen verhindern, dass das LLM systematisch zu aggressive oder zu passive Parameter liefert:

| Guardrail | Regel | Begruendung |
|-----------|-------|-------------|
| R-abhaengige Mindestweiten | Vor 1.5R ist Trail verboten. 1.5-2R: mindestens 2.5 als Factor | Verhindert vorzeitiges Ausstoppen in fruehen Trade-Phasen |
| Mode-Hysterese | `trail_mode` darf nur wechseln, wenn 2 consecutive Bars den gleichen Mode liefern | Verhindert Flipping zwischen WIDE und TIGHT |
| Monitoring | Wenn `pct_tight > 80%` UND `median_R_captured < 0.35R`: LLM-Influence zurueck auf NORMAL | Erkennt systematische Drift und korrigiert automatisch |

### 4.7 Subregimes

Das bisherige 5-stufige Regime-Modell (TREND_UP, TREND_DOWN, RANGE_BOUND, HIGH_VOLATILITY, UNCERTAIN) wird um Subregimes erweitert, die eine feinere taktische Steuerung ermoeglichen:

| Regime | Subregimes |
|--------|-----------|
| TREND_UP | `EARLY`, `MATURE`, `LATE`, `EXHAUSTION` |
| TREND_DOWN | `RELIEF_RALLY`, `PULLBACK`, `ACCELERATION`, `CAPITULATION` |
| RANGE_BOUND | `MEAN_REVERT`, `RANGE_EXPANSION`, `BREAKOUT_ATTEMPT`, `CHOP` |
| HIGH_VOLATILITY | `NEWS_SPIKE`, `MOMENTUM_EXPANSION`, `AFTERSHOCK_CHOP`, `LIQUID_VOL` |
| UNCERTAIN | `TRANSITION`, `MIXED_SIGNALS`, `LOW_LIQUIDITY`, `DATA_QUALITY` |

**Bestimmung:** KPI-Engine primaer, LLM als sekundaerer Hint. Der Resolver waehlt bei Widerspruch zwischen KPI und LLM das konservativere Subregime.

---

## 5. Teil 3: TACTICAL_EXIT und Exhaustion-Detection

### 5.1 TACTICAL_EXIT — Neue Exit-Prioritaet

TACTICAL_EXIT wird als neue Exit-Prioritaet eingefuegt, zwischen REGIME_CHANGE (Prio 5) und TARGET (Prio 6). Es ist der primaere Mechanismus, ueber den das LLM Einfluss auf Exit-Timing nehmen kann — gebunden an KPI-Bestaetigung.

**KPI-Signale** (alle aus IndicatorResult/MarketSnapshot):

| Signal | Bedingung |
|--------|-----------|
| `rsi_reversal` | RSI war > 70, kreuzt jetzt unter 60 |
| `ema_bear_cross` | EMA(9) < EMA(21) |
| `vwap_loss` | Close < VWAP bei hoher volumeRatio |
| `structure_break` | Lower-Low / Close unter Vorbar-Low |

**Logik nach `exit_bias`:**

| exit_bias | Verhalten |
|-----------|-----------|
| `NEUTRAL` | TACTICAL_EXIT deaktiviert |
| `HOLD` | TACTICAL_EXIT deaktiviert |
| `EXIT_SOON` | Exit wenn mind. 2 von 4 Signalen true ODER 1 Signal + exhaustion >= `EARLY_WARNING` |
| `EXIT_NOW` | Exit wenn mind. 1 Signal true UND (exhaustion == `CONFIRMED` ODER MFE >= 1R) |

> **Eiserne Regel:** Das LLM allein erzwingt keinen Exit. Ohne mindestens ein bestaetigendes KPI-Signal bleibt die Position offen, unabhaengig vom `exit_bias`.

### 5.2 Exhaustion-Detection — KPI-Kriterien

Die Exhaustion-Detection basiert auf drei Saeulen, die unabhaengig voneinander ausgewertet werden. Alle Inputs stammen aus der KPI-Engine:

#### Saeule A: Extension (mind. 1 true)

| Kriterium | Schwellwert |
|-----------|-------------|
| RSI(14, 5m) | >= 78 |
| Bollinger-Extension | Close >= Bollinger Upper UND (Close - BollingerUpper) / ATR >= 0.25 |
| VWAP-Extension | (High - VWAP) / ATR >= 2.5 |

#### Saeule B: Climax (mind. 1 true)

| Kriterium | Schwellwert |
|-----------|-------------|
| Volumen-Climax | volumeRatio >= 2.5 |
| Range-Climax | Bar-Range / ATR >= 1.8 |

#### Saeule C: Rejection / Momentum Loss (mind. 1 true)

| Kriterium | Schwellwert |
|-----------|-------------|
| BB Re-Entry | prev_close > BB_upper UND close < BB_upper |
| VWAP Snapback | high > VWAP + 2.5*ATR UND close <= VWAP + 0.5*ATR |
| Fast RSI Drop | RSI mind. 8 Punkte unter RSI vor 2 Bars |
| Trend Stall | ema9 - ema21 schrumpft 2 Bars in Folge |

**Gesamtkriterium:**

```
exhaustion_kpi_confirmed = (A AND B) AND (C)
```

Alle drei Saeulen muessen mindestens ein true-Kriterium liefern. Dies stellt sicher, dass Exhaustion nur bei Konvergenz mehrerer unabhaengiger Signale erkannt wird — kein einzelner Indikator reicht aus.

---

## 6. Teil 4: Implementierungs-Roadmap

### Uebersicht

| Phase | Scope | Status |
|-------|-------|--------|
| **P0** | R-basierte Profit-Protection + Trail erst ab 1.5R + Regime-spezifische Trail-Weite | Teilweise implementiert |
| **P1** | LLM-Schema erweitern (exit_bias, trail_mode etc.) + TacticalPolicy-Resolver + TACTICAL_EXIT | Offen |
| **P2** | exhaustion_signal + Subregime-Resolver + Anti-LLM-Drift Monitoring | Offen |
| **P3** | entry_timing_bias + scale_out_profile | Offen |
| **P4** | Multi-Cycle-Day: FLAT_INTRADAY State, Cycle-Counter, ALLOW_RE_ENTRY, Zyklen-Risk-Budget | Offen |

### P0: Deterministische Profit-Protection (sofort)

- R-basierte Stufentabelle implementieren
- Highwater-Mark Trail (bereits implementiert, Commit `75a1042`)
- ATR einfrieren mit Faktor 2.2 (bereits implementiert, Commit `75a1042`)
- Trail erst ab 1.5R aktivieren
- Regime-spezifische Trail-Weite

### P1: LLM Decision-Features (naechste Phase)

- LLM-Response-Schema um P0-Features erweitern (`exit_bias`, `trail_mode`, `profit_protection_profile`, `target_policy`)
- TacticalPolicy-Resolver: Uebersetzt LLM-Features in konkrete Parameter
- TACTICAL_EXIT als neue Exit-Prioritaet in der Rules Engine
- Precedence-Kette und Konfliktaufloesung

### P2: Exhaustion und Subregimes (2-3 Wochen)

- `exhaustion_signal` Feature
- KPI-basierte Exhaustion-Detection (Drei-Saeulen-Modell)
- Subregime-Resolver (KPI primaer, LLM sekundaer)
- Anti-LLM-Drift Monitoring

### P3: Erweiterte Entry-Steuerung (spaeter)

- `entry_timing_bias` Feature
- `scale_out_profile` Feature (OMS-Integration)

---

## 7. Teil 5: Multi-Cycle-Day-Strategie

### 7.1 Paradigmenwechsel: Von Single-Trade-Day zu Multi-Cycle-Day

Das bisherige ODIN-Modell geht implizit von einem **Single-Trade-Day** aus: Pro Pipeline und Instrument wird ein Entry/Exit-Zyklus durchgefuehrt — die Pipeline kauft, verwaltet die Position (Trailing-Stop, Scaling-Out), verkauft, und wechselt dann in den Endzustand. Ein Re-Entry nach komplettem Exit ist nicht vorgesehen.

Dieses Modell verschenkt systematisch Ertragspotenzial. Intraday-Kurse bewegen sich nicht linear von A nach B, sondern durchlaufen multiple Impulse, Korrekturen und Erholungen. Ein Instrument, das morgens stark steigt, mittags korrigiert und nachmittags wieder anzieht, bietet *zwei* handelbare Zyklen — das Single-Trade-Day-Modell kann nur den ersten nutzen.

**Multi-Cycle-Day** bedeutet: Eine Pipeline kann pro Tag **mehrere vollstaendige Entry/Exit-Zyklen** auf demselben Instrument durchfuehren, sofern die Marktsituation das rechtfertigt. Dabei gelten weiterhin alle bestehenden Regeln (Risk-Limits, EOD-Flat, Kill-Switch).

> **Kernthese:** Dies ist DER zentrale Use-Case fuer die LLM-Integration. Ein statischer Algorithmus kann nicht zuverlaessig entscheiden, ob ein Plateau temporaer oder terminal ist, ob ein Drawdown V-foermig recovert oder in einen Abwaertstrend uebergeht. Das LLM kann kontextuelle Muster erkennen (Exhaustion, Recovery-Signale, Marktstruktur), die deterministisch nicht abbildbar sind.

### 7.2 Zyklen-Typen

Multi-Cycle-Day unterscheidet drei Zyklen-Typen, die sich in Timing, Positionsgroesse und Haltedauer unterscheiden:

#### Zyklus-Typ 1: Trend-Riding (primaerer Zyklus)

Der klassische ODIN-Trade: Momentum-basierter Entry, Gewinne absichern, Exit bei Exhaustion oder Trailing-Stop.

| Eigenschaft | Auspraegung |
|-------------|-------------|
| Timing | Fruehe RTH oder Pre-Market-Breakout |
| Entry-Signal | Standard-Entry-Regeln (Regime, KPI-Bestaetigung, LLM-Kontext) |
| Positionsgroesse | Volles Tagesbudget fuer dieses Instrument (Standard-Sizing) |
| Exit-Trigger | Exhaustion-Detection, TACTICAL_EXIT, Trailing-Stop, Target |
| Besonderheit | Exit bei Plateau/Exhaustion statt Warten auf Trailing-Stop |

Der entscheidende Unterschied zum bisherigen Modell: Der Exit erfolgt **proaktiv beim Plateau**, nicht erst beim Trailing-Stop-Trigger. Wenn das LLM Exhaustion-Signale erkennt und die KPI-Engine das bestaetigt (TACTICAL_EXIT mit `exit_bias = EXIT_NOW`), wird die Position geschlossen — auch wenn der Trailing-Stop noch nicht ausgeloest wurde. Dies realisiert Gewinne naeher am Hoch und schafft die Voraussetzung fuer einen moeglichen Re-Entry.

#### Zyklus-Typ 2: Recovery-Trade

Re-Entry nach einem signifikanten Drawdown, wenn das LLM Recovery-Signale identifiziert.

| Eigenschaft | Auspraegung |
|-------------|-------------|
| Timing | Nach Abschluss einer Korrektur (typisch: 30-90 Min nach Zyklus-1-Exit) |
| Entry-Signal | LLM-`entry_timing_bias = ALLOW_RE_ENTRY` + KPI-Bestaetigung (V-Recovery, Volumen-Shift) |
| Positionsgroesse | **Reduziert** (50-75% des Standard-Sizings), da Tagesbudget teilweise aufgebraucht |
| Exit-Trigger | Trailing-Stop, EOD-Flat, Target |
| Besonderheit | Konservativere Entry-Schwellen, engerer initialer Stop |

Der Recovery-Trade ist das primaere Multi-Cycle-Szenario. Er setzt voraus, dass:

1. Zyklus 1 profitabel abgeschlossen wurde (realisierter Gewinn als Puffer)
2. Ein messbarer Drawdown stattgefunden hat (mindestens 2-3% vom Zyklus-1-Hoch)
3. Das LLM eine Erholung identifiziert, die nicht nur ein Dead-Cat-Bounce ist
4. Die KPI-Engine die Recovery-These stuetzt (Volumen-Shift, Support-Test, EMA-Rekreuzung)

#### Zyklus-Typ 3: Scalp/Quick-Trade (optional, spaetere Phase)

Kurzer opportunistischer Trade bei klaren kurzfristigen Signalen. Kleine Position, enge Stops, schneller Exit. Dieser Zyklus-Typ ist explizit als P4-Feature geplant und wird in der initialen Multi-Cycle-Implementierung **nicht** unterstuetzt.

### 7.3 LLM-Rolle bei Multi-Cycle-Entscheidungen

Multi-Cycle-Day erfordert vier Arten von LLM-Entscheidungen, die ein statischer Algorithmus nicht abbilden kann:

#### Plateau-Erkennung (Exit Zyklus 1)

Das LLM analysiert den Kontext und erkennt, wann ein Kurs seinen Zenit erreicht hat:

- **Widerstandsniveaus:** Historische Highs, runde Zahlen, institutionelle Level
- **Erschoepfungsmuster:** Abnehmende Bar-Ranges bei steigendem Volumen, Doji-Formationen, Wick-Rejections
- **Marktstruktur:** Nachlassender Orderflow, Bid-Staerke nimmt ab, Spread weitet sich

Die Plateau-Erkennung wird ueber die bestehende Exhaustion-Detection (Kap. 5.2) und `exit_bias = EXIT_NOW` operationalisiert. Multi-Cycle ergaenzt hier keinen neuen Mechanismus, sondern nutzt den Mechanismus *konsequenter* — Exit auf dem Plateau statt Warten auf den Trailing-Stop.

#### Drawdown-Timing (Warten)

Nach dem Exit aus Zyklus 1 muss das LLM bewerten, ob die laufende Korrektur abgeschlossen ist:

- **"Abverkauf noch nicht durch":** Hohe Verkaufsvolumina, keine Unterstuetzung an Support-Leveln, staerker werdende Abwaertsdynamik
- **"Boden gefunden":** Volumen-Climax (Selling Exhaustion), Preis haelt Support, erste Kaeufer-Signale

Diese Bewertung verhindert den klassischen Fehler eines voreiligen Re-Entry in einen laufenden Abverkauf.

#### Recovery-Signal (Entry Zyklus 2)

Das LLM identifiziert den Beginn einer nachhaltigen Erholung:

- **V-foermige Recovery:** Scharfe Umkehr mit starkem Kaufvolumen
- **Konstruktive Konsolidierung:** Boden-Bildung mit abnehmender Volatilitaet und steigendem Bid
- **Kontext-Wechsel:** Neuer Katalysator oder veraenderter Sektorflow

#### Multi-Cycle-Entscheidungsfeld: `entry_timing_bias`

Das bestehende Decision-Feature `entry_timing_bias` (P2/P3, Kap. 4.4) wird um den Wert `ALLOW_RE_ENTRY` erweitert:

| Wert | Bedeutung |
|------|-----------|
| `ALLOW_NOW` | Standard-Entry, erster Zyklus |
| `WAIT_PULLBACK` | Warten auf Ruecksetzer |
| `WAIT_CONFIRMATION` | Warten auf Bestaetigung |
| `ALLOW_RE_ENTRY` | **Neu:** Re-Entry nach komplettem Exit erlaubt. LLM hat Recovery-Signale identifiziert |
| `SKIP` | Kein Entry |

`ALLOW_RE_ENTRY` aktiviert den Recovery-Trade-Pfad mit reduzierten Positionsgroessen und erhoehten Entry-Schwellen.

### 7.4 Referenzbeispiel: IREN am 18.02.2026

Das folgende Szenario zeigt einen konkreten Multi-Cycle-Day anhand von IREN (Iris Energy Limited) am 18. Februar 2026.

#### Zyklus 1: Trend-Riding (14:50 - ~16:30 UTC)

| Zeitpunkt (UTC) | Kurs | Aktion | Begruendung |
|-----------------|------|--------|-------------|
| ~14:50 | 42.00 | **Entry** | Momentum-Signal: EMA(9) > EMA(21), RSI steigend, starkes Volumen. LLM: Regime TREND_UP/EARLY |
| 15:00 - 16:00 | 42.00 → 43.50 | Position laeuft | Trail-Stops werden nachgezogen (R-basierte Profit-Protection). Subregime wechselt von EARLY zu MATURE |
| ~16:15 | 43.50 - 43.73 | Plateau-Erkennung | LLM erkennt Resistance-Zone: Bar-Ranges nehmen ab, Volumen bleibt hoch aber Kurs stagniert. Wick-Rejections an 43.70+. `exhaustion_signal = CONFIRMED`, `exit_bias = EXIT_NOW` |
| ~16:30 | ~43.50 | **Exit (80-100%)** | TACTICAL_EXIT ausgeloest: `exit_bias = EXIT_NOW` + KPI bestaetigt (RSI-Reversal, Structure-Break). Realisierter Gewinn: ~1.50 USD/Aktie (~3.6%) |

#### Pause: Drawdown abwarten (16:30 - 20:30 UTC)

| Zeitpunkt (UTC) | Kurs | Pipeline-State | LLM-Bewertung |
|-----------------|------|---------------|---------------|
| 16:30 - 17:30 | 43.50 → 42.00 | FLAT_INTRADAY | "Korrektur laeuft, kein Boden in Sicht. Hohes Verkaufsvolumen" |
| 17:30 - 19:00 | 42.00 → 41.15 | FLAT_INTRADAY | "Abverkauf noch aktiv, -6% vom Tageshoch. Kein Recovery-Signal" |
| 19:00 - 20:00 | 41.15 - 41.50 | FLAT_INTRADAY | "Erste Stabilisierung. Volumen-Climax (Selling Exhaustion). Abwarten" |
| ~20:30 | 41.50 | FLAT_INTRADAY → SEEKING_ENTRY | "V-foermige Erholung beginnt. Kaufvolumen nimmt zu, Support bei 41.00 haelt. Re-Entry sinnvoll." `entry_timing_bias = ALLOW_RE_ENTRY` |

#### Zyklus 2: Recovery-Trade (20:30 - Close)

| Zeitpunkt (UTC) | Kurs | Aktion | Begruendung |
|-----------------|------|--------|-------------|
| ~20:30 | 41.50 - 42.00 | **Re-Entry** | LLM: `ALLOW_RE_ENTRY` + KPI: EMA-Rekreuzung, Volumen-Shift, RSI von Ueberverkauft steigend. Positionsgroesse: 60% des Standards (Tagesbudget teilweise aufgebraucht) |
| 20:30 - 21:00 | 42.00 → 42.06 | Position laeuft | Konservativerer Trail (enger als Zyklus 1). LLM: Subregime TREND_UP/EARLY |
| ~21:00 (Close) | 42.06 | **Exit** | EOD-Flat (FORCED_CLOSE). Realisierter Gewinn: ~0.06 USD/Aktie (minimal, aber risikokontrolliert) |

#### Tagesbilanz

| Metrik | Zyklus 1 | Zyklus 2 | Gesamt |
|--------|----------|----------|--------|
| Entry-Preis | 42.00 | 41.50-42.00 | -- |
| Exit-Preis | ~43.50 | ~42.06 | -- |
| Gewinn/Aktie | ~1.50 USD | ~0.06 USD | ~1.56 USD |
| Positionsgroesse | 100% | 60% | -- |
| Haltedauer | ~100 Min | ~30 Min | ~130 Min |

> **Vergleich mit Single-Trade-Day:** Im Single-Trade-Modell haette ODIN entweder (a) auf dem Plateau gehalten und den Drawdown auf 41.15 mitgenommen (Trailing-Stop bei ~42.80 ausgeloest, Gewinn ~0.80 USD/Aktie) oder (b) nach dem Exit auf dem Plateau den Tag beendet (Gewinn ~1.50 USD/Aktie, kein Zyklus 2). Multi-Cycle-Day ermoeglicht das Beste aus beiden Welten: Gewinnrealisierung auf dem Plateau *und* Partizipation an der Recovery.

### 7.5 Architektur-Implikationen

Multi-Cycle-Day erfordert gezielte Erweiterungen in mehreren Modulen. Die folgenden Aenderungen sind so konzipiert, dass sie das bestehende System erweitern, ohne bestehende Funktionalitaet zu brechen.

#### PipelineStateMachine: Neuer Zustand FLAT_INTRADAY

Die bisherige FSM kennt keinen Zustand fuer "Position komplett geschlossen, aber Handelstag nicht vorbei". Der Uebergang `POSITIONED → OBSERVING` (nach Exit) genuegt konzeptionell, aber semantisch fehlt die Unterscheidung zwischen:

- **OBSERVING (erstmalig):** Pipeline hat noch nie gehandelt, sucht erstes Setup
- **FLAT_INTRADAY (nach Exit):** Pipeline hat einen Zyklus abgeschlossen, Position ist flat, Re-Entry moeglich

Neuer Zustand:

| State | Beschreibung | Decision Loop | Re-Entry moeglich |
|-------|-------------|---------------|-------------------|
| FLAT_INTRADAY | Position komplett geschlossen, Tag laeuft weiter | Ja (LLM-Analyse, Regime-Monitoring) | Ja, bei `ALLOW_RE_ENTRY` |

State-Uebergaenge:

```
POSITIONED → FLAT_INTRADAY     (komplett verkauft, Handelstag laeuft)
FLAT_INTRADAY → SEEKING_ENTRY  (LLM: ALLOW_RE_ENTRY + KPI-Bestaetigung)
FLAT_INTRADAY → FORCED_CLOSE   (Handelszeit endet)
FLAT_INTRADAY → DAY_STOPPED    (Hard-Stop erreicht oder Max-Cycles)
```

> **Abgrenzung zu OBSERVING:** FLAT_INTRADAY traegt den Kontext des abgeschlossenen Zyklus (realisierter P&L, verbrauchtes Risk-Budget, Cycle-Counter). OBSERVING ist der initiale Zustand ohne diesen Kontext. Die Trennung ist notwendig, damit die Rules Engine bei Re-Entry-Pruefungen auf die Tageshistorie zugreifen kann.

#### OMS: Re-Entry nach komplettem Exit

Das Order Management System muss Re-Entry nach komplettem Exit unterstuetzen:

- **Cycle-Counter:** Pro Pipeline wird ein `cycleNumber` gefuehrt (Zyklus 1, 2, etc.)
- **Unabhaengige Trade-Verwaltung:** Jeder Zyklus wird als eigenstaendiger Trade mit eigenen Stops, Targets und P&L verwaltet
- **Positionsgroessen-Berechnung:** Das Sizing fuer Zyklus 2+ beruecksichtigt das verbrauchte Tagesbudget

#### Risk-Budget und Positionsgroesse

Die Risk-Limits muessen Zyklen-uebergreifend angewandt werden:

| Risk-Regel | Single-Trade-Day | Multi-Cycle-Day |
|------------|-------------------|-----------------|
| Max. Tagesverlust | Ueber alle Pipelines | Ueber alle Pipelines UND alle Zyklen |
| Positionsgroesse | Standard-Sizing | Zyklus 1: Standard. Zyklus 2+: Reduziert (konfigurierbar, Default 60%) |
| Risk-Budget pro Zyklus | Gesamtes Tagesbudget | Zyklus 1: Tagesbudget. Zyklus 2+: Restbudget nach realisiertem P&L |
| Max. Zyklen pro Tag | 1 | Konfigurierbar (Default: 3), Hard-Cap als Guardrail |

#### EventLog: Zyklen-Trennung im Audit-Trail

Alle EventRecords muessen den Zyklus identifizieren koennen:

- **`cycleNumber`** als Feld in relevanten Events (Entry, Exit, Stop-Adjustment, LLM-Response)
- **Cycle-Summary** als aggregierter Event am Ende jedes Zyklus
- **Tages-Summary** aggregiert ueber alle Zyklen

#### `entry_timing_bias`: Erweiterung

Das bestehende Decision-Feature `entry_timing_bias` (Kap. 4.4, P2/P3) wird um `ALLOW_RE_ENTRY` ergaenzt. Dieser Wert hat spezifische Guards:

- Nur verfuegbar wenn Pipeline-State == FLAT_INTRADAY
- Erfordert mindestens einen abgeschlossenen, profitablen Zyklus
- `cycleNumber < maxCyclesPerDay`
- Verbleibendes Risk-Budget >= minimaler Entry-Size

### 7.6 Multi-Cycle in der Implementierungs-Roadmap

Multi-Cycle-Day baut auf den bestehenden Phasen (P0-P3) auf und wird als **P4** eingeplant:

| Phase | Scope | Abhaengigkeit |
|-------|-------|---------------|
| **P4a** | FLAT_INTRADAY State + Cycle-Counter + EventLog-Erweiterung | P0 (Profit-Protection), P1 (TACTICAL_EXIT fuer proaktiven Plateau-Exit) |
| **P4b** | `ALLOW_RE_ENTRY` in `entry_timing_bias` + Recovery-Trade Entry-Guards | P2 (`entry_timing_bias` Feature), P4a |
| **P4c** | Risk-Budget Zyklen-uebergreifend + Reduced Sizing | P4b |
| **P4d** | Scalp/Quick-Trade (Zyklus-Typ 3) | P4c, nach ausreichender Backtest-Validierung |

> **Abhaengigkeitskette:** Multi-Cycle-Day ist die logische Konsequenz der in P1/P2 eingefuehrten LLM-Features. Ohne proaktive Plateau-Erkennung (TACTICAL_EXIT) und Exhaustion-Detection gibt es keinen sinnvollen Exit-Trigger fuer Zyklus 1. Ohne `entry_timing_bias` gibt es keinen Mechanismus fuer den Re-Entry. Die Phasen bauen zwingend aufeinander auf.

### 7.7 Guardrails und Sicherheitsmechanismen

Multi-Cycle-Day erweitert die Angriffsflaeche des Systems. Folgende Guardrails verhindern unkontrolliertes Re-Entry:

| Guardrail | Regel | Begruendung |
|-----------|-------|-------------|
| Max-Cycles-Per-Day | Konfigurierbar (Default: 3), Hard-Cap | Verhindert Overtrading bei volatilen Tagen |
| Cooling-Off nach Exit | Minimum 15 Minuten zwischen Exit Zyklus N und Entry Zyklus N+1 | Verhindert Impuls-Re-Entry direkt nach Exit |
| Profit-Gate | Re-Entry nur wenn Zyklus N profitabel (realisierter P&L > 0) | Kein "Nachlegen" nach Verlust-Trades |
| Budget-Gate | Verbleibendes Risk-Budget muss minimale Positionsgroesse erlauben | Verhindert Micro-Entries ohne sinnvolles R/R |
| LLM-Pflicht | Re-Entry nur mit frischem LLM-Assessment (`ALLOW_RE_ENTRY`) | Kein algorithmischer Re-Entry ohne LLM-Kontext |
| Zeitfenster | Kein Re-Entry in den letzten 30 Minuten vor FORCED_CLOSE | Zu wenig Zeit fuer sinnvollen Trade |

---

## 8. Appendix: User-Entscheidungen

Die folgenden Entscheidungen wurden im Rahmen des Sparrings vom User getroffen und sind bindend:

| Vorschlag | Entscheidung | Begruendung |
|-----------|-------------|-------------|
| P0 Highwater-Mark Trail | **Angenommen** | Bereits implementiert (Commit `75a1042`) |
| P1 ATR einfrieren, Faktor 2.2 | **Angenommen** | Bereits implementiert (Commit `75a1042`) |
| P3 Extension-Filter (kein Entry >3% vom Low) | **Abgelehnt** | Zu statisch. V-Recovery ist normal bei High-Beta-Aktien. Ein fixer Prozent-Schwellwert wuerde valide Entries in volatilen Phasen verhindern |
| P6 Time-of-Day Trail | **Abgelehnt** | Trail ist ein Fallnetz, kein Timing-Tool. Zeitbasierte Trail-Anpassung fuehrt zu Overfitting an historische Intraday-Muster |

> **Leitentscheidung:** "ODIN ist kein reines Algo-Trading. Das LLM muss unterstuetzende Entscheidungsgewalt haben." — Dies begruendet die gesamte Architektur des Tactical Parameter Controllers und die Abkehr von einem rein deterministischen System.

---

## 9. Relation zum Fachkonzept v1.5

Dieser Abschnitt dokumentiert **alle identifizierten Konflikte (K1–K19)** zwischen dem **Fachkonzept v1.5 (FK)** und diesem **Strategie-Dokument**. Die **Status-Spalte** ist normativ und definiert, ob und wie FK v1.5 durch dieses Dokument uebersteuert wird.

| ID | Thema | Fachkonzept v1.5 | Strategie-Dokument | Status | Anmerkung |
|----|-------|-------------------|---------------------|--------|-----------|
| K1 | LLM-Einfluss auf Stop-/Trail-Parameter | LLM hat **keinen Einfluss** auf Stop-Level; Stop-Logik ist Rules/ATR-dominiert (Kap. 2, 7). | LLM wird als **Tactical Parameter Controller** konzipiert und liefert gebundene Parameter (z.B. `trail_mode`) (Kap. 4). | Ersetzt FK | **Paradigmenwechsel (bindende Leitentscheidung):** LLM darf Parameter liefern, aber **nur gebunden (Enums/Profiles)**; Rules/Resolver uebersetzen deterministisch und clampen. LLM kann keine numerischen Stop-Level vorschlagen oder direkt setzen. |
| K2 | LLM-Criticality: Trailing LLM-unabhaengig vs. `trail_mode` | Trailing-Stop-Anpassung ist als **LLM-unabhaengig** klassifiziert (FK Kap. 9, LLM-Criticality). | `trail_mode` wirkt als Multiplikator und wird in Konfliktaufloesung/Precedence einbezogen (Kap. 4.5). | Ersetzt FK | FK-Criticality muss aktualisiert werden: **Hard-Exit bleibt LLM-unabhaengig**, aber Trail-**Weite** kann (gebunden) LLM-beeinflusst sein. Bei LLM-Ausfall: Fallback `trail_mode=NORMAL`. |
| K3 | Trailing-Stop-Definition | Einstufig: **1x ATR vom Intraday-High** (FK Kap. 7, Exit-Regeln). | Mehrstufig: **R-basierte Profit-Protection** + Regime-Anpassung + MFE-Lock (Kap. 3.1). | Ersetzt FK | FK-Exit-Regel "Trailing-Stop" wird durch **Profit-Protection v2** ersetzt (P0, teilweise implementiert). |
| K4 | Trailing-Aktivierungsschwelle | Trailing als generischer Exit-Mechanismus ohne R-Schwelle (FK Kap. 7). | **Trail verboten vor 1.5R**, danach aktiv (wide → tighter) (Kap. 3.1, 4.6). | Ersetzt FK | Teil von P0: verhindert zu fruehes Ausstoppen in der Fruehphase eines Trades. |
| K5 | Break-even nach 1R | Nach 1R: Stop auf **Break-even (Entry)** (FK Kap. 7, Scaling-Out). | Nach 1R: Stop >= **Entry + 0.10R** (Fees/Slippage-Puffer) (Kap. 3.1). | Ersetzt FK | FK-Begriff "Break-even" wird als **Entry + fee_buffer_R** praezisiert (Default 0.10R, konfigurierbar). |
| K6 | Stop-Resolver / Monotonie | Keine formale Definition, wie Stop-Kandidaten zusammengefuehrt werden (FK Kap. 7). | Formal: `effectiveStop = max(prevStop, stop0, trailCandidate, mfeLockStop)`; Stop **kann nur steigen** (Kap. 3.1). | Ersetzt FK | Normative Resolver-Regel: Highwater-Mark-Invariante. Audit-Log der Kandidaten empfohlen. |
| K7 | Runner-Trailing: Struktur (EMA/HL) vs. Profit-Protection | Runner-Trail unter EMA(9) oder letztem Higher-Low; Tightening bei RSI > 80 (FK Kap. 7). | Runner-Mechanik nicht explizit adressiert; Fokus auf R/MFE-Profit-Protection (Kap. 3.1). | Klaerungsbedarf | **Entscheidung erforderlich:** Struktur-Trail als zusaetzlicher Stop-Kandidat in `max()` beibehalten, oder zugunsten eines einheitlichen R/MFE-basierten Exit-Systems entfernen. Empfehlung: Beibehalten als zusaetzlicher Kandidat — Struktur kann frueher schliessen als Profit-Protection, aber nie unter dem Floor. |
| K8 | "Urgency erzwingt keinen Exit" vs. TACTICAL_EXIT | Urgency CRITICAL fuehrt zu Re-Evaluation, aber **Urgency allein loest keinen Exit aus** (FK Kap. 7). | `exit_bias` aktiviert **TACTICAL_EXIT** — Exit nur bei KPI-Bestaetigung (Kap. 5.1). | Geplant | Umsetzung in P1. Eiserne Regel bleibt: **LLM allein erzwingt keinen Exit**. TACTICAL_EXIT erfordert mindestens ein bestaetigendes KPI-Signal. FK muss um dieses Konzept erweitert werden, sobald implementiert. |
| K9 | Precedence-Kette / Konfliktaufloesung | Keine formale Rangfolge zwischen Exit-Quellen beschrieben (FK Kap. 7, 9). | Precedence-Kette: Hard-Exits > Profit-Protection > LLM-Parameter > Soft-Exits. Konfliktregel: `effectiveTrailFactor = min(...)` (Kap. 4.5). | Geplant | Umsetzung in P1. Zentrale Konfliktregel, die ins FK uebernommen werden muss, sobald implementiert. |
| K10 | Subregimes: Autoritaet KPI vs. LLM | Regime-Analyse stark LLM-getrieben; Quant validiert (FK Kap. 2, 13). | Subregime-Resolver: **KPI primaer**, LLM sekundaer; bei Widerspruch konservatives Subregime (Kap. 4.7). | Geplant | Umsetzung in P2. FK muss zwischen Regime (LLM-gefuehrt) und Subregime (KPI-gefuehrt) trennen. Das Fuenf-Stufen-Regime-Modell des FK bleibt als Oberkategorie erhalten. |
| K11 | ADX als Entry-Filter | Entry-Regeln ohne ADX-Filter: EMA(9) > EMA(21), RSI < 70 (FK Kap. 7). | Vorschlag: ADX > 20–25 als Trend-Qualitaetsfilter (Kap. 3.3). | Geplant | Backtest/Plateau-Test erforderlich bevor Aktivierung. Danach FK-Entry-Guards erweitern. |
| K12 | `target_policy = TRAIL_ONLY` vs. Tranchierung | Scaling-Out in Tranchen (3–5) ist Kernmechanik der Gewinnmitnahme (FK Kap. 7). | Option: `TRAIL_ONLY` — Targets ignorieren, Trail dominiert (Kap. 4.2). | Klaerungsbedarf | **Inkompatibel** mit bestehender OMS-Tranchierungslogik ohne Anpassung. Empfehlung: `TRAIL_ONLY` hoechstens fuer Runner-Anteil zulassen oder erst nach OMS-Anpassung freigeben. |
| K13 | `scale_out_profile` (LLM steuert Teilverkaeufe) | Scaling-Out ist Rules-dominiert, LLM-unabhaengig (FK Kap. 9, LLM-Criticality). | Geplant: `scale_out_profile` zur OMS-Steuerung durch LLM-Profil (Kap. 4.4). | Geplant | Umsetzung in P3. Wenn umgesetzt: **nur Profilwahl** (OFF/CONSERVATIVE/STANDARD/AGGRESSIVE), keine freien Groessen/Preise. Rules bleiben deterministisch, LLM waehlt Profil. |
| K14 | Anti-LLM-Drift Guardrails | Keine Drift-Guardrails beschrieben (ueber Timeout/Retry hinaus) (FK Kap. 9, 11). | Guardrails: R-Mindestweiten, Mode-Hysterese, Monitoring-Reset auf NORMAL (Kap. 4.6). | Geplant | Umsetzung in P2. Muss als Betriebssicherheits-Mechanik ins FK aufgenommen werden. |
| K15 | ATR einfrieren (Faktor 2.2) | ATR-Nutzung breit beschrieben, aber ohne Freeze-Semantik (FK Kap. 7, 8, 10). | ATR-Freeze mit Faktor 2.2 als beschlossen und implementiert (Kap. 6, P0). | Ersetzt FK | FK muss exakt definieren: **welche ATR** (Entry-ATR auf 5-Min), **wann Freeze** (bei Entry), **wo Faktor wirkt** (Stop0-Berechnung, R-Normalisierung, Trail-Basis). |
| K16 | Single-Trade-Day vs. Multi-Cycle-Day | Implizites Single-Trade-Day-Modell: `MANAGING_TRADE → OBSERVING` nach Exit, aber kein Re-Entry-Mechanismus beschrieben (FK Kap. 5, 7). FK Edge-Case "Reclaim ohne Follow-through" verbietet Re-Entry explizit fuer Bull-Trap-Muster. | **Multi-Cycle-Day:** Mehrere Entry/Exit-Zyklen pro Tag auf demselben Instrument. Neuer State FLAT_INTRADAY, Cycle-Counter, LLM-gesteuerte Re-Entry-Logik (Kap. 7). | Geplant | Umsetzung in P4. **Erweiterung, kein Widerspruch:** Das FK-Verbot "kein Re-Entry fuer dieses Pattern heute" gilt weiterhin fuer fehlgeschlagene Setups (Bull Traps). Multi-Cycle adressiert den Fall, dass Zyklus 1 **erfolgreich** war und die Marktsituation einen neuen Zyklus rechtfertigt. |
| K17 | Pipeline-FSM: Fehlender FLAT_INTRADAY-Zustand | FSM kennt OBSERVING ↔ POSITIONED (via Fill/Exit), aber keinen Zwischen-Zustand fuer "flat nach profitablem Exit" (FK Kap. 5). | Neuer Zustand FLAT_INTRADAY mit eigenem Decision-Loop-Verhalten und Re-Entry-Guards (Kap. 7.5). | Geplant | Umsetzung in P4a. FLAT_INTRADAY ist eine Spezialisierung von OBSERVING mit zusaetzlichem Zyklen-Kontext (realisierter P&L, verbrauchtes Risk-Budget, Cycle-Counter). |
| K18 | Positionsgroesse: Statisch vs. Zyklen-abhaengig | Positionsgroesse basiert auf Tageskapital / aktive Pipelines, keine Zyklen-Dynamik (FK Kap. 1, 10). | Zyklus 2+ mit reduzierter Positionsgroesse (Default 60%), abhaengig vom verbrauchten Risk-Budget (Kap. 7.5). | Geplant | Umsetzung in P4c. FK-Positionsberechnung wird um Cycle-Faktor erweitert. |
| K19 | `entry_timing_bias`: ALLOW_RE_ENTRY | Feature nicht im FK beschrieben (erst in diesem Dokument als P2/P3, Kap. 4.4). | Neuer Enum-Wert `ALLOW_RE_ENTRY` mit spezifischen Guards: nur in FLAT_INTRADAY, nur nach profitablem Zyklus, Budget-Gate (Kap. 7.3). | Geplant | Umsetzung in P4b. Abhaengig von P2 (`entry_timing_bias` Feature). |

---

### Vorrangregel (Konfliktaufloesung)

1. **Diese Tabelle ist die Quelle der Wahrheit fuer Abweichungen** zwischen FK v1.5 und diesem Strategie-Dokument.
2. **Status-Auswertung ist verbindlich:**
    - **Ersetzt FK:** Die hier beschriebene Regel/Architektur **uebersteuert** FK v1.5. Das Fachkonzept gilt an dieser Stelle als veraltet und muss bei naechster Revision aktualisiert werden.
    - **Ergaenzt FK:** FK v1.5 bleibt gueltig; dieses Dokument **ergaenzt** ohne Widerspruch (zusaetzliche Normen/Definitionen).
    - **Geplant:** Noch nicht umgesetzt; **FK v1.5 gilt operativ weiter**, bis die Umsetzung erfolgt und der Status aktualisiert wird.
    - **Klaerungsbedarf:** Widerspruch ist **nicht entschieden**; **FK v1.5 gilt operativ weiter**, bis eine Entscheidung dokumentiert und der Status geaendert ist.
3. **Implementierungs-/Entscheidungs-Updates:** Jeder Wechsel von "Geplant/Klaerungsbedarf" zu "Ersetzt/Ergaenzt" muss in dieser Tabelle nachvollziehbar aktualisiert werden (inkl. Commit-/Ticket-Verweis in der Spalte "Anmerkung").
