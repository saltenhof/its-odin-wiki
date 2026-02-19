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
7. [Appendix: User-Entscheidungen](#7-appendix-user-entscheidungen)
8. [Relation zum Fachkonzept v1.5](#8-relation-zum-fachkonzept-v15)

---

## 1. Executive Summary

Dieses Dokument fasst die Ergebnisse eines strukturierten Sparrings zusammen, das zwei fundamentale Weiterentwicklungen fuer ODIN erarbeitet hat:

1. **R-basierte Profit-Protection:** Ein mehrstufiges, deterministisches Trail-System, das Gewinne proportional zum erreichten R-Multiple absichert. Ersetzt den bisherigen einstufigen Trailing-Stop durch eine dynamische Stufentabelle mit Regime-Anpassung.

2. **LLM als Tactical Parameter Controller:** Evolution der LLM-Rolle vom reinen Analyst zum Parameter-Lieferanten. Das LLM entscheidet nicht mehr BUY/SELL, sondern liefert gebundene, strukturierte Parameter (exit_bias, trail_mode, etc.), die deterministisch in Regeln uebersetzt werden. Dies bewahrt die Reproduzierbarkeit bei gleichzeitig adaptiver Intelligenz.

Beide Konzepte sind komplementaer: Die Profit-Protection bildet den deterministischen Floor, den das LLM nicht unterschreiten kann. Das LLM kann innerhalb dieses Floors taktisch agieren — aber niemals gegen die Sicherheitsmechanismen.

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

## 7. Appendix: User-Entscheidungen

Die folgenden Entscheidungen wurden im Rahmen des Sparrings vom User getroffen und sind bindend:

| Vorschlag | Entscheidung | Begruendung |
|-----------|-------------|-------------|
| P0 Highwater-Mark Trail | **Angenommen** | Bereits implementiert (Commit `75a1042`) |
| P1 ATR einfrieren, Faktor 2.2 | **Angenommen** | Bereits implementiert (Commit `75a1042`) |
| P3 Extension-Filter (kein Entry >3% vom Low) | **Abgelehnt** | Zu statisch. V-Recovery ist normal bei High-Beta-Aktien. Ein fixer Prozent-Schwellwert wuerde valide Entries in volatilen Phasen verhindern |
| P6 Time-of-Day Trail | **Abgelehnt** | Trail ist ein Fallnetz, kein Timing-Tool. Zeitbasierte Trail-Anpassung fuehrt zu Overfitting an historische Intraday-Muster |

> **Leitentscheidung:** "ODIN ist kein reines Algo-Trading. Das LLM muss unterstuetzende Entscheidungsgewalt haben." — Dies begruendet die gesamte Architektur des Tactical Parameter Controllers und die Abkehr von einem rein deterministischen System.

---

## 8. Relation zum Fachkonzept v1.5

Dieser Abschnitt dokumentiert **alle identifizierten Konflikte (K1–K15)** zwischen dem **Fachkonzept v1.5 (FK)** und diesem **Strategie-Dokument**. Die **Status-Spalte** ist normativ und definiert, ob und wie FK v1.5 durch dieses Dokument uebersteuert wird.

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

---

### Vorrangregel (Konfliktaufloesung)

1. **Diese Tabelle ist die Quelle der Wahrheit fuer Abweichungen** zwischen FK v1.5 und diesem Strategie-Dokument.
2. **Status-Auswertung ist verbindlich:**
    - **Ersetzt FK:** Die hier beschriebene Regel/Architektur **uebersteuert** FK v1.5. Das Fachkonzept gilt an dieser Stelle als veraltet und muss bei naechster Revision aktualisiert werden.
    - **Ergaenzt FK:** FK v1.5 bleibt gueltig; dieses Dokument **ergaenzt** ohne Widerspruch (zusaetzliche Normen/Definitionen).
    - **Geplant:** Noch nicht umgesetzt; **FK v1.5 gilt operativ weiter**, bis die Umsetzung erfolgt und der Status aktualisiert wird.
    - **Klaerungsbedarf:** Widerspruch ist **nicht entschieden**; **FK v1.5 gilt operativ weiter**, bis eine Entscheidung dokumentiert und der Status geaendert ist.
3. **Implementierungs-/Entscheidungs-Updates:** Jeder Wechsel von "Geplant/Klaerungsbedarf" zu "Ersetzt/Ergaenzt" muss in dieser Tabelle nachvollziehbar aktualisiert werden (inkl. Commit-/Ticket-Verweis in der Spalte "Anmerkung").
