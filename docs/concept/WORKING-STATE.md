# Working State: Backtest-Optimierung ODIN

**Stand:** 2026-02-26 (nach Bereinigung unterbrochener Optimization-Agent)
**Ziel:** >=3% Gewinn im Backtest (IREN, 2026-02-23)
**Referenz-Playbook:** `T:/codebase/its_odin/playbook-backtest-optimization.md`

---

## Gesamtstand

| Metrik | Wert |
|--------|------|
| Aktuell bester PnL | $1,553 (IREN-iter2-schema-fix) |
| Ziel-PnL | $3,000 (+3.00%) |
| Delta zum Ziel | ~$1,447 |
| Bereinigt am | 2026-02-26 |

---

## Leitprinzip (Stakeholder-Klarstellung 2026-02-26)

**IREN 2026-02-23 ist Fallstudie, kein Optimierungsziel.**

Das Ziel ist NICHT: 3% auf IREN an diesem Tag um jeden Preis.
Das Ziel IST: Prinzipien und Mechanismen entwickeln, die reproduzierbar positive Rendite ueber Instrumente und Tage hinweg erzeugen.

2.5% mit generalisierbarer Strategie > 3.5% durch Overfitting.

Fuer jede Optimierungsmassnahme gilt: "Wuerde das auch bei anderen High-Beta-Trending-Days funktionieren?" — Wenn nein: verwerfen, egal was es auf IREN bringt.

---

## Bereinigung (2026-02-26)

Ein unterbrochener Optimization-Agent hatte uncommitted Aenderungen hinterlassen, die selektiv bereinigt wurden.

### Was zurueckgesetzt wurde

| Parameter | War (dirty) | Zurueckgesetzt auf | Begruendung |
|-----------|-------------|-------------------|-------------|
| `odin.execution.risk.max-position-percent` | 90 | **50** | 90% verletzt Risk-Management-Konzept |
| `odin.execution.risk.max-exposure-percent` | 95 | **80** | Konsistent mit Position-Limit |
| `odin.brain.rules.entry.volume-ratio-min` | 0.1 | **0.3** | 0.1 deaktiviert Check de facto; 0.3 als Kompromiss |

### Was behalten wurde (committete Verbesserungen)

| Aenderung | Wert | Begruendung |
|-----------|------|-------------|
| LlmAnalystOrchestrator Tactical-Schema-Migration | - | Bug-Fix: `analyze()` nutzt jetzt `parseTactical()` API + `deriveUrgencyFromTactical()` |
| `odin.brain.llm.call-timeout-s` | 90s (war 30s) | Infrastruktur-Anpassung fuer LLM-Latenz |
| `odin.brain.gates.vwap-standard-atr-factor` | 3.0 (war 1.5) | Konzeptionell korrekt: Gate und Entry-Rule konsistent |
| `odin.brain.gates.volume-standard-min-ratio` | 0.3 (war 0.6) | Workaround fuer Pre-Market-Zero-Bar-Artefakt |
| LlmAnalystOrchestratorTest | - | Tests auf Tactical-Schema migriert (kein VALID_LLM_RESPONSE mehr) |

---

## Meta-Analyse

### Was hat IREN am 2026-02-23 gemacht?

**Kursanstieg:** +8.1% (Open $39.16 -> Close $42.36, High $42.56)

**Verlaufsmuster:** Klassischer Gap-Up Trend Day
- **09:30 ET (15:30 CET):** Massiver Open-Bar: $39.16 -> $40.25 (+2.8%, 3.1M Volume)
- **09:30-09:50 ET:** Fortsetzung bis $41.17 (+5.1% in 20 Min)
- **10:00-12:30 ET:** Morning Consolidation: Range $40.50-$41.50
- **12:55-13:30 ET:** Afternoon Breakout: Push bis $42.20
- **14:55 ET:** Day High $42.56
- **15:55 ET (21:55 CET):** Close bei $42.36

**Groesster Bewegungsanteil:** Erste 20 Minuten (5% der 8% Gesamtbewegung)

### Aktuelles Backtest-Verhalten (Bester Run)

- Entry: 10:50 ET bei ~$41.12 (1286 Shares, FULL_5T)
- Stop: $40.28 (2.2x ATR)
- Keine Target-Hits, keine Stop-Hits
- Exit: EOD Forced Close (EXIT_KILL_SWITCH) bei ~$42.36
- PnL: $1,553
- **Problem:** 80 Minuten TREND_UP mit >0.92 Confidence, Gates passing -- aber QuantVote = NO_TRADE

### Stakeholder-Input 2026-02-26

**Pre-Market Volume:** Bekannte Limitation, Datenprovider liefern das nicht. Pre-Market hat zu geringe Liquiditaet und ist KEIN verlaesslicher Indikator fuer RTH-Verlauf.

**Das entscheidende Signal: Die ersten 5-10 Minuten RTH ("Upper/Lower Half"-Pattern)**

Beobachtung: Volatile Titel machen oft einen starken Opening-Move, dann eine Korrektur.
Entscheidend ist: Wo hoert die Korrektur auf?
- Ruecksetzer bleibt in der "oberen Haelfte" -> **bullisches Bestaetigungssignal**
- Gegenbewegung wird sofort abverkauft -> **baerisches Signal**

**IREN 23.02. konkret:**
- 15:30 CET: RTH-Open -> sofort scharf nach oben
- ~15:38 CET: Korrektur setzt ein
- ~15:40 CET: Korrektur stoppt — deutlich UEBER dem Opening-Niveau
- **Optimaler Entry: ~15:44-15:46 CET** — 4.6%+ bis Close

**Zweite Einstiegsgelegenheit: ~17:15-17:20 CET (Intraday-Dip)**

**Support/Resistance Level Detection — kritischster Optimierungspunkt:**

IREN 23.02. — praezise S/R-Bounce-Sequenz:
- **17:15 CET:** 1. Bounce -> zurueck nach oben
- **17:58 CET:** 2. Bounce am gleichen Level
- **18:37 CET:** 3. Bounce -> **optimaler Einstieg** (2 vorherige Bounces bestaetigt)
- **18:51 CET:** 4. Versuch nach unten -> **abgewiesen** -> IREN explodiert (+4.4%+)

Praezision: Zwischen 1. und 3. Rejection nur **0.16% Kursabstand**.

### Ist 3% realistisch?

**Theoretisches Maximum:** $4,689 (4.34%) bei Perfect Entry/Exit.
**Realistisches Maximum:** ~$2,349 (2.17%) bei bestaetiger Trend-Entry + Trail.
**Bei aktuellem Entry-Timing (10:50 ET):** max $1,595 -> **3% ist bei aktuellem Timing NICHT erreichbar.**

3% ist **moeglich**, WENN der Entry 30-60 Minuten frueher erfolgt.

---

## Ergebnis-Log

| Iter | Was getestet | PnL | Delta | Bewertung |
|------|-------------|-----|-------|-----------|
| Iter 10 | RSI 75 | $1,414.08 | Baseline | Kein Effekt -- RSI war nicht der Blocker |
| Iter 11 | RSI 80 | $1,414.08 | 0 | Kein Effekt -- selbe Ursache |
| Iter 12 | VolumeRatio 0.5 | $1,414.08 | 0 | Kein Effekt -- bereits 0.5 in Baseline |
| Cleanup | Schema-Fix + Parameter-Bereinigung | $1,553 | +$139 | Tactical-Schema-Migration + kalibrierte Parameter |

---

## Naechste Phase

S/R-Level-Engine Implementation — Details vom naechsten Optimization-Agent.

---

## Bekannte Einschraenkungen (konzeptkonform)

- Strategie steigt erst ein wenn Trend bestaetigt -> fruehe Bewegung verpasst (Opening Buffer 15 Min)
- Take-Profit-Tranchen schmaelern Gesamtrendite (im aktuellen besten Run nicht relevant, da EOD-Close)
- Long-Only -> nur Aufwaertstrends nutzbar

---

## Learnings

1. **Alle bisherigen Iterationen (10-12) waren wirkungslos** weil sie den falschen Parameter aenderten
2. **Die wahren Blocker sind Volume-Ratio UND intermittierender VWAP-Gate-Fail**
3. **$1,414 PnL kommt ausschliesslich vom EOD-Forced-Close** -- nicht von Trade-Exits
4. **Pre-Market Zero-Volume Bars verfaelschen KPI-Berechnungen** im Backtest
5. **3% erfordert 30-60 Min frueheren Entry** -- kein anderer Hebel reicht
6. **Dirty State von unterbrochenen Agents muss bereinigt werden** bevor weiteroptimiert wird
7. **90% Position-Sizing verletzt das Risk-Management-Konzept** — nie ueber 50% gehen
