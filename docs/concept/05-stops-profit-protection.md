# 05 -- Stops, Trailing, Profit Protection, Exit-Priorisierung, Scaling-Out

> **Quellen:** Master-Konzept v1.1 (Trailing Stop & Profit Protection, R-basierte Stufentabelle), Build-Spec v3.0 (Kap. 10: Stops, Trail-Mechanik, Exhaustion-Detection), Strategie-Sparring (ATR-Freeze 2.2x, Highwater-Mark, Runner-Trail, TRAIL_ONLY, Profit-Protection als Floor), Stakeholder-Feedback 1+2 (Trailing-Stop = Fallnetz, keine statischen Pauschalregeln)

---

## 1. Grundprinzipien (Stakeholder-Entscheidungen)

| Prinzip | Regelung | Status |
|---------|----------|--------|
| **Highwater-Mark** | Der Trailing-Stop DARF nur steigen, nie fallen. `effectiveStop = max(prevStop, stop0, trailCandidate, mfeLockStop)` | Implementiert |
| **ATR-Freeze** | ATR wird bei Entry eingefroren (`entryAtr`). Faktor **2.2** fuer Stop-Berechnung, R-Normalisierung, Trail-Basis | Implementiert |
| **Runner-Trail bleibt** | Der bisherige Runner-Trail (EMA(9)/Higher-Low) bleibt als zusaetzlicher Stop-Kandidat in `max()` bestehen | Bestaetigt |
| **Profit-Protection als Floor** | Die R-basierte Stufentabelle definiert Mindest-Stop-Level. LLM-Parameter KOENNEN innerhalb des Floors variieren, aber nie darunter | Bestaetigt |
| **Trailing-Stop = Fallnetz** | Der Trailing-Stop ist ein Sicherheitsmechanismus, kein Timing-Tool. Er darf nur steigen (Highwater-Mark) und DARF NICHT zeitbasiert variiert werden | Stakeholder-Entscheidung |

> **Leitentscheidung:** "ODIN ist kein reines Algo-Trading-System. Das LLM muss unterstuetzende Entscheidungsgewalt haben -- aber gebunden an deterministische Leitplanken." Die Profit-Protection bildet den deterministischen Floor, den das LLM nicht unterschreiten kann.

---

## 2. ATR-Freeze und Faktor 2.2 (Stakeholder-Entscheidung)

Der ATR-Wert wird zum Zeitpunkt des Entries eingefroren. Der Faktor 2.2 (statt vormals 1.5) wird fuer drei Berechnungen verwendet:

| Verwendung | Formel |
|------------|--------|
| Initiale Stop-Distance | `stop0 = entry - 2.2 x entryAtr` (Long) |
| R-Normalisierung | `1R = 2.2 x entryAtr` |
| Trail-Basis | Profit-Protection-Trail-Berechnung basiert auf `entryAtr` |

**Begruendung:** Der breitere Faktor verhindert vorzeitiges Ausstoppen bei normalem Intraday-Noise. Der ATR-Freeze stellt sicher, dass die Stop-Distance waehrend des gesamten Trades konsistent bleibt -- unabhaengig davon, ob die Volatilitaet im Tagesverlauf faellt oder steigt.

> **ATR-Quelle:** ATR(14) auf 3m-Bars (Decision-Frame). Der ATR wird bei Entry eingefroren und fuer die gesamte Trade-Dauer verwendet.

---

## 3. Initial Stop (Normativ)

Der Initial Stop wird aus mehreren Kandidaten berechnet. Fuer Long-Positionen gilt: der tiefere (breitere) Kandidat gewinnt, da er dem Trade mehr Raum gibt.

### 3.1 Stop-Kandidaten

| Kandidat | Berechnung | Anwendung |
|----------|-----------|-----------|
| **Structure Stop** | Unter letztem validen Swing-Low / Flush-Low | Bietet strukturellen Schutz an charttechnisch relevanter Stelle |
| **ATR Stop** | `entry - 2.2 x entryAtr` | Standard-Stop basierend auf ATR-Freeze |
| **VWAP-Offset Stop** (optional) | `vwap - vwap_offset_atr x ATR` | Wenn Trend-VWAP als Anker relevant ist |

### 3.2 Berechnung

```
initial_stop = min(structure_stop, atr_stop, vwap_offset_stop)  -- fuer Long: "tiefer" = mehr Raum
```

**Risk-Gate-Begrenzung:** Die maximale Stop-Distance wird durch das Risk-Gate begrenzt. Wenn die Stop-Distance zu gross ist, wird die Positionsgroesse entsprechend reduziert (Fixed Fractional Risk), um das max. Risiko pro Trade einzuhalten.

### 3.3 Stop-Platzierung (Normativ)

- Initial Stop MUSS sofort nach Fill gesetzt werden (Bracket-Logik im OMS)
- Stop wird als **eigenstaendige Stop-Order** auf die **gesamte Restposition** platziert
- Stop ist **GTC beim Broker** (ueberlebt System-Crash)
- Stop darf **nie gelockert** werden (nur tighten)

---

## 4. R-basierte Profit-Protection Stufentabelle (Stakeholder-Entscheidung)

Der Kern des Stop-Managements ist eine R-basierte Stufentabelle, die den Trailing-Stop progressiv enger zieht, je weiter der Trade im Gewinn liegt. **R** bezeichnet das Vielfache des initialen Risikos: `1R = entry - initial_stop = 2.2 x entryAtr`.

| R-Schwelle | Stop-Typ | Trail-ATR-Factor | MFE-Lock % | Verhalten |
|------------|----------|-----------------|------------|-----------|
| **< 1.0R** | Nur initialer Stop (`stop0`) | -- | -- | Kein Trail, kein Tightening. Position hat Raum zum Entwickeln |
| **>= 1.0R** | Break-Even + Puffer | -- | -- | `stop >= entry + 0.10R` (Puffer fuer Fees/Slippage) |
| **>= 1.5R** | Trail aktiv (wide) | 2.75 | -- | Erster Trail-Schritt, bewusst weit um Trendkontinuation zu erlauben |
| **>= 2.0R** | Trail enger + MFE-Lock | 2.00 | 35% | MFE-Lock sichert mindestens 35% des bisher erreichten Gewinns |
| **>= 3.0R** | Aggressiver Trail + Lock | 1.25 | 60% | Signifikanter Gewinn wird aggressiv geschuetzt |
| **>= 4.0R** | Final Protect | 1.00 | 75% | Maximaler Schutz, Trail auf 1.0x ATR |

### 4.1 Berechnung des effektiven Stops

```
trailCandidate = currentHigh - (trail_atr_factor x entryAtr)
mfeLockStop    = entry + (mfe_lock_pct x MFE)
effectiveStop  = max(prevStop, stop0, trailCandidate, mfeLockStop, runnerTrailCandidate)
```

**Highwater-Mark-Invariante:** Der `effectiveStop` kann ausschliesslich steigen, nie fallen. Diese Invariante MUSS im Audit-Log verifizierbar sein.

### 4.2 MFE-Lock

MFE = Maximum Favorable Excursion seit Entry (hoechster unrealisierter Gewinn in Waehrung).

Der MFE-Lock sichert einen Mindestprozentsatz des bisher erreichten MFE ab:

```
mfeLockStop = entry + (mfe_lock_pct x MFE)
```

**Beispiel:** Entry bei 42.00, MFE bei 44.00 (= +2.00 USD), MFE-Lock 35% → `mfeLockStop = 42.00 + 0.35 x 2.00 = 42.70`. Der Stop darf nicht unter 42.70 fallen, auch wenn der ATR-Trail einen niedrigeren Wert liefert.

### 4.3 Profit-Protection-Profile (LLM-steuerbar)

Das LLM kann ueber `profit_protection_profile` das Profil waehlen:

| Profil | Verhalten |
|--------|-----------|
| `OFF` | Keine Profit-Protection (nur initialer Stop + manueller Trail) |
| `STANDARD` | R-basierte Stufentabelle wie oben, Trail ab 1.5R |
| `AGGRESSIVE` | Engere Faktoren, Trail bereits ab 1.0R mit `factor = 3.0`, MFE-Lock ab 1.5R mit 25% |

**Konfliktaufloesung:** Es gilt IMMER der engere (konservativere) Wert:

```
effectiveTrailFactor = min(factor_from_llm_trail_mode, factor_from_profit_protection)
```

---

## 5. Regime-spezifische Trail-Weite

Die Trail-Weite (ATR-Faktor `k`) wird zusaetzlich an das aktuelle Marktregime angepasst:

| Regime | Trail-Faktor `k` | Begruendung |
|--------|------------------|-------------|
| TREND_UP stark (ADX hoch) | 2.0 -- 3.0 | Breiter Trail laesst dem Trend Luft |
| TREND_UP schwach / Chop | 1.2 -- 2.0 | Engerer Trail, da Trendkontinuation unsicher |
| HIGH_VOLATILITY | Initial 3.0, dann Umschaltung per Profit-Protection-Stufen | Anfangs breit gegen Noise, dann progressives Tightening |
| RANGE_BOUND | 1.2 -- 1.5 | Enger, da Mean-Reversion erwartet wird |

**Interaktion mit LLM `trail_mode`:** Das LLM liefert `trail_mode` (WIDE/NORMAL/TIGHT) als Multiplikator auf den Base-Trail-Faktor:

| trail_mode | Multiplikator |
|------------|--------------|
| `WIDE` | 1.5x |
| `NORMAL` | 1.0x |
| `TIGHT` | 0.75x |

Der resultierende Trail-Faktor wird auf `[minTrail, maxTrail]` geclampt.

---

## 6. Runner-Trailing-Regeln

Der Runner ist der letzte Restbestand einer Position (typisch 10-20%), der mit einem speziellen Trailing-Stop fuer grosse Trend-Days gehalten wird. Die Runner-Trailing-Regeln liefern einen zusaetzlichen Stop-Kandidaten im `max()`-Resolver.

| Situation | Trailing-Regel |
|-----------|---------------|
| **Standard** | Trail unter EMA(9) oder letztes Higher-Low -- je nachdem was naeher am Kurs liegt |
| **Ueberhitzt** (RSI > 80, oberes Bollinger-Band) | Engerer Trail, unter letzte Kerze |
| **Ruecksetzer gesund** (Preis haelt MA-Zone) | Runner halten, Trail nicht veraendern |
| **Ruecksetzer kippt** (Close unter MA-Zone, Lower-Low) | Runner sofort schliessen |

**Tightening bei RSI > 80:** Wenn RSI(14) > 80 UND Close >= oberes Bollinger-Band: Runner-Trail wird auf `closeOfLastBar - 0.5 x entryAtr` gesetzt (sehr eng). Dies schuetzt vor Blow-off-Tops.

**TRAIL_ONLY Override (Stakeholder-Entscheidung):** Wenn `target_policy = TRAIL_ONLY` vom LLM geliefert wird, MUSS das OMS alle Target-Orders stornieren und die gesamte Restposition ausschliesslich per Trailing-Stop managen. Dies deaktiviert implizit das Scaling-Out ueber Target-Orders. Beabsichtigt fuer Trend-Days, bei denen partielle Gewinnmitnahmen den Gesamtprofit schmaelern wuerden.

---

## 7. Exhaustion-Detection (Drei-Saeulen-Modell)

Die Exhaustion-Detection basiert auf drei unabhaengigen Saeulen. ALLE drei MUESSEN mindestens ein true-Kriterium liefern. Dies stellt sicher, dass Exhaustion nur bei Konvergenz mehrerer unabhaengiger Signale erkannt wird -- kein einzelner Indikator reicht aus.

### 7.1 Saeule A: Extension (mind. 1 true)

| Kriterium | Schwellwert | Beschreibung |
|-----------|-------------|-------------|
| RSI-Extension | RSI(14, 3m) >= 78 | Momentum-Ueberhitzung |
| Bollinger-Extension | Close >= Bollinger Upper UND `(Close - BollingerUpper) / ATR >= 0.25` | Preis ueber dem oberen Band hinausgeschossen |
| VWAP-Extension | `(High - VWAP) / ATR >= 2.5` | Preis extrem weit vom Tages-VWAP entfernt |

### 7.2 Saeule B: Climax (mind. 1 true)

| Kriterium | Schwellwert | Beschreibung |
|-----------|-------------|-------------|
| Volumen-Climax | `volumeRatio >= 2.5` | Ueberdurchschnittliches Volumen (Kapitulations-Kaeufer) |
| Range-Climax | `Bar-Range / ATR >= 1.8` | Einzelbar extrem gross relativ zur normalen Volatilitaet |

### 7.3 Saeule C: Rejection / Momentum Loss (mind. 1 true)

| Kriterium | Schwellwert | Beschreibung |
|-----------|-------------|-------------|
| BB Re-Entry | `prev_close > BB_upper UND close < BB_upper` | Preis faellt zurueck unter das obere Bollinger-Band |
| VWAP Snapback | `high > VWAP + 2.5 x ATR UND close <= VWAP + 0.5 x ATR` | Starke Ablehnung vom Extension-Level |
| Fast RSI Drop | RSI mind. 8 Punkte unter RSI vor 2 Bars | Schneller Momentum-Verlust |
| Trend Stall | `EMA(9) - EMA(21)` schrumpft 2 Bars in Folge | Trend-Spread verengt sich, Impuls laesst nach |

### 7.4 Gesamtkriterium

```
exhaustion_kpi_confirmed = (Saeule_A_has_true) AND (Saeule_B_has_true) AND (Saeule_C_has_true)
```

### 7.5 Exhaustion-Signal als LLM-Feature

Das LLM liefert `exhaustion_signal` als Enum:

| Wert | Bedeutung | Auswirkung |
|------|-----------|-----------|
| `NONE` | Keine Exhaustion erkannt | Kein Einfluss |
| `EARLY_WARNING` | Fruehe Warnsignale (1-2 Saeulen aktiv) | `exit_bias` wird mindestens `EXIT_SOON`, `trail_mode` wird mindestens `TIGHT` |
| `CONFIRMED` | Alle drei Saeulen aktiv (KPI-bestaetigt) | `exit_bias = EXIT_NOW`, TACTICAL_EXIT wird geprueft |

### 7.6 Exhaustion-Reaktion im OMS

Wenn `exhaustion_kpi_confirmed = true`:

- Scale-Out-Profil wird auf AGGRESSIVE geschaltet
- Trail wird sofort auf die engste verfuegbare Stufe gesetzt
- Adds werden blockiert
- TACTICAL_EXIT wird mit hoechster Prioritaet geprueft

---

## 8. TACTICAL_EXIT (Stakeholder-Entscheidung)

TACTICAL_EXIT ist eine Exit-Prioritaet zwischen REGIME_CHANGE (Prio 4) und zeitbasiertem Exit (Prio 6). Es ist der primaere Mechanismus, ueber den das LLM Einfluss auf Exit-Timing nehmen kann -- **gebunden an KPI-Bestaetigung**.

### 8.1 KPI-Signale (alle aus IndicatorResult/MarketSnapshot)

| Signal | Bedingung |
|--------|-----------|
| `rsi_reversal` | RSI war > 70, kreuzt jetzt unter 60 |
| `ema_bear_cross` | EMA(9) < EMA(21) |
| `vwap_loss` | Close < VWAP bei hoher volumeRatio |
| `structure_break` | Lower-Low / Close unter Vorbar-Low |

### 8.2 Logik nach exit_bias

| exit_bias | Verhalten |
|-----------|-----------|
| `NEUTRAL` | TACTICAL_EXIT deaktiviert |
| `HOLD` | TACTICAL_EXIT deaktiviert |
| `EXIT_SOON` | Exit wenn mind. **2 von 4** Signalen true ODER 1 Signal + `exhaustion_signal >= EARLY_WARNING` |
| `EXIT_NOW` | Exit wenn mind. **1 Signal** true UND (`exhaustion_signal == CONFIRMED` ODER MFE >= 1R) |

> **Eiserne Regel:** Das LLM allein erzwingt KEINEN Exit. Ohne mindestens ein bestaetigendes KPI-Signal bleibt die Position offen, unabhaengig vom `exit_bias`.

---

## 9. Exit-Prioritaetsordnung (Normativ)

Exit-Trigger werden in fester Rangfolge geprueft. Hoehere Prioritaet uebersteuert niedrigere:

| Prio | Trigger | LLM-Abhaengigkeit | Beschreibung |
|------|---------|-------------------|-------------|
| **1** | **Stop-Loss** | Keine (Broker-seitig, GTC) | `Preis <= entry - 2.2 x entryAtr` |
| **2** | **Forced Close** | Keine | `MarketClock > forcedCloseStart` (15:45 ET) |
| **3** | **Trailing-Stop** | Keine | Preis faellt unter effectiveStop |
| **4** | **Regime-Wechsel** | Wuenschenswert | Regime-Wechsel 2x bestaetigt, Position passt nicht zum neuen Regime |
| **5** | **TACTICAL_EXIT** | Nur als Signal | LLM `exit_bias` + KPI-Bestaetigung (siehe Kap. 8) |
| **6** | **Zeitbasiert** | Kontext | `hold_duration_bars` ueberschritten |
| **7** | **Gewinnmitnahmen** | Keine | R-basierte Tranchen-Targets |

**Hard-Exits (Prio 1-3) sind IMMER LLM-unabhaengig.** Sie werden vom OMS autonom ausgefuehrt, auch wenn das LLM nicht erreichbar ist.

---

## 10. Scaling-Out (Multi-Tranchen-Modell)

### 10.1 Tranchen-Profile

Die Tranchenanzahl ist **budgetabhaengig** (konfigurierbar):

| Variante | Bedingung | Tranchen | Aufteilung |
|----------|----------|----------|------------|
| **Kompakt (3T)** | < 5.000 USD | 3 | 40% @ 1R, 40% @ 2R, 20% Runner |
| **Standard (4T)** | 5.000 -- 15.000 USD | 4 | 30% @ 1R, 30% @ Key-Level, 25% @ 2R, 15% Runner |
| **Voll (5T)** | > 15.000 USD | 5 | 25% @ 1R, 25% @ Key-Level, 25% @ 2R, 15% @ 3R, 10% Runner |

**USD-Schwellenwerte** gelten fuer US-Aktien. Fuer EU-Aktien werden die Schwellenwerte in EUR mit dem aktuellen EUR/USD-Spot umgerechnet.

### 10.2 Scale-Out-Profile (LLM-steuerbar, P3)

Das LLM kann ueber `scale_out_profile` das Profil waehlen:

| Profil | Verhalten |
|--------|-----------|
| `OFF` | Kein Scaling-Out (nur manueller Exit oder Trail) |
| `CONSERVATIVE` | Fruehe Gewinnmitnahme: 40% @ 1R, Rest Standard |
| `STANDARD` | Standard-Profil gemaess Tabelle oben |
| `AGGRESSIVE` | Spaetere Gewinnmitnahme: 20% @ 1R, mehr Runner-Anteil |

**Nur Profilwahl:** Das LLM waehlt ein Profil, keine freien Groessen/Preise. Rules bleiben deterministisch.

### 10.3 Position Building: Starter + Add

Fuer Setup A und B wird eine gestufte Positionsaufbau-Strategie verwendet:

| Phase | Anteil | Zeitpunkt | Stop |
|-------|--------|-----------|------|
| **Starter** | ~40% der Zielgroesse | RECLAIM-State (Setup A) / erste Bestaetigung | Unter Flush-Low / Swing-Low |
| **Add** | ~60% der Zielgroesse | RUN-State bestaetigt / Higher-Low | Unter letztes Higher-Low |

Gesamtrisiko (Starter + Add) DARF den `max_risk` pro Trade (3% Tageskapital) NICHT ueberschreiten.

---

## 11. Precedence-Kette und Konfliktaufloesung (Stakeholder-Entscheidung)

Die strikte Rangfolge aller Einflussquellen auf Stop- und Trail-Parameter:

1. **Hard-Exits** (KillSwitch, ForcedClose, StopLoss, TrailingStop) -- IMMER zuerst, LLM-unabhaengig
2. **Profit-Protection** setzt Floor der Tightness (R-basierte Stufentabelle)
3. **LLM `trail_mode`** darf innerhalb der Grenzen variieren, aber nie gegen (2)
4. **LLM `exit_bias`** kann nur Soft-Exit-Schwellen veraendern, nicht Hard-Exits

**Konfliktaufloesung bei Trail-Widerspruch:**

```
effectiveTrailFactor = min(factor_from_llm, factor_from_profit_protection)
```

Es gilt IMMER der engere (konservativere) Wert. Das LLM kann den Trail nie weiter setzen als die Profit-Protection-Stufe vorgibt.

---

## 12. Anti-LLM-Drift Guardrails fuer Stop/Trail

Drei Mechanismen verhindern, dass das LLM systematisch zu aggressive oder zu passive Trail-Parameter liefert:

| Guardrail | Regel | Begruendung |
|-----------|-------|-------------|
| **R-abhaengige Mindestweiten** | Vor 1.5R ist Trail verboten. 1.5-2R: mindestens 2.5 als Factor | Verhindert vorzeitiges Ausstoppen in fruehen Trade-Phasen |
| **Mode-Hysterese** | `trail_mode` darf nur wechseln, wenn 2 consecutive Bars den gleichen Mode liefern | Verhindert Flipping zwischen WIDE und TIGHT |
| **Monitoring** | Wenn `pct_tight > 80%` UND `median_R_captured < 0.35R`: LLM-Influence zurueck auf NORMAL | Erkennt systematische Drift und korrigiert automatisch |

---

## 13. Parabolic/Exhaustion-Handling im OMS

Wenn Parabolic Acceleration oder Exhaustion erkannt wird:

| Triggerquelle | Aktion |
|--------------|--------|
| 1m Event `EXHAUSTION_CLIMAX` | Scale-Out-Ladder triggert sofort, Trail tighten |
| 1m Event `PARABOLIC_ACCEL` | Arbiter bevorzugt SCALE_OUT (nicht sofort Voll-Exit) |
| Regime `EXHAUSTION_RISK` | Scale-Out-Profil aggressiver, Trail enger, Adds blockieren |
| ADR14-Extension (`intraday_range / ADR14 > threshold`) | Verschaerfte Exit-Pruefung |

Wenn Struktur bricht (Lower-Low, Close unter Vorbar-Low): sofortiger Exit der gesamten Restposition.

---

## 14. Stop-Nachfuehrung im OMS (Normativ)

Die Stop-Nachfuehrung erfolgt auf **1m-Bar-Close** (nicht nur bei 3m Decision-Bar-Close). Sie ist OMS-Maintenance, kein TradeIntent.

**Update-Cadence:**

| Trigger | Aktion |
|---------|--------|
| Jeder 1m-Bar-Close (wenn Trailing aktiv) | Stop-Kandidaten berechnen, `effectiveStop = max(...)` anwenden |
| CRITICAL Monitor Event (Blow-off, Crash) | Sofort tighten, ausserhalb des normalen 1m-Takts |
| Target-Tranche gefuellt | Stop-Quantity auf Restposition reduzieren, ggf. Stop auf Break-Even |

**Bei Tranche-1-Fill:** Stop wird auf Break-Even gesetzt (`entry + 0.10R`).

---

## 15. Sensitivitaetsanalyse: Plateau-Test

Fuer die kritischen Parameter `stopLossAtrFactor` (2.2), `trailingStopAtrFactor` und `vwapOffsetAtrFactor` SOLL ein systematischer Plateau-Test durchgefuehrt werden:

- **Methode:** Jeden Parameter um +/- 10% variieren und die Performance-Aenderung messen
- **Plateau = robust:** Wenn die Performance-Kurve flach ist, ist der Parameter-Wert stabil und nicht ueberoptimiert
- **Nadel = Overfit:** Wenn die Performance bei minimaler Aenderung stark einbricht, ist der Wert fragil und wahrscheinlich an historische Daten ueberangepasst
- **Kriterium:** P&L-Ergebnis darf bei +/- 10% Variation nicht um > 30% schwanken

> **Keine statischen Pauschalregeln (Stakeholder-Entscheidung):** Regeln wie "kein Entry wenn > 3% vom Tagestief" sind abgelehnt. V-Recovery ist normal bei High-Beta-Aktien. Alle Optimierungen MUESSEN allgemeingueltig sein und duerfen nicht auf einen Chartverlauf oder eine einzelne Aktie optimiert werden.
