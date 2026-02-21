## Evaluationsdesign: MC vs. Unified (geometrisch, OHLCV-only)

### 0) Ziel / Hypothese

**Zielmetrik ist geometrisches Wachstum**, d. h. große Drawdowns zählen überproportional negativ. Kernhypothesen:

* **H1 (MC):** geringere Tail-Risk-Frequenz (seltener große Fehlentscheidungen) → stabileres geometrisches Wachstum.
* **H2 (Unified):** bessere Regime-/Szenarioerkennung → höhere Opportunity-Capture, aber potenziell mehr Tail-Risk durch LLM-Einfluss.

---

## 1) Zu vergleichende Systemvarianten (Ablation-Set)

### A) MC-Ansatz (Baseline)

* QuantEngine entscheidet (KPI-first).
* LLM nur als **Tactical Parameter Controller** (bounded enums), keine direkten Trade-Triggers.
* Einträge/Adds strikt Quant-gated, LLM modifiziert nur Profile (Trail/ExitBias/etc.) innerhalb von Clamps.

### B) Unified-Veto (symmetrisch, aber nur als „Schutzengel“)

* QuantEngine generiert Trade-Intents.
* LLM hat **Veto-Recht** für Entry/Add (z. B. `NO_TRADE/SAFETY_VETO`), aber **kein** Recht, Entry/Add zu erzwingen.
* Für Exits darf LLM `EXIT_SOON` triggern, aber `EXIT_NOW` nur mit Confirm (z. B. 1m exhaustion event oder 3m Strukturbruch).

### C) Unified-Full (Dual-Key)

* Entry/Add nur bei **QuantVote ∧ LLMVote** (beide >= Confidence-Schwelle, 10m Confirm ok).
* Exit/Scale-Out durch „one-key pending“ + Confirm/Timeout.

**Optional (nur zur Diagnose, nicht als Kandidat):**

* D) Quant-only (LLM aus): misst reinen Mehrwert/Schaden des LLM.
* E) LLM-only (nur wenn du’s explizit willst): meistens als Negativkontrolle.

---

## 2) Daten & Setup (OHLCV-only, 1m→3m, 10m Confirm)

* **Simulationsclock:** 1-Minute.
* **Entscheidungstakt:** 3-Minute Close.
* **Bestätigung:** 10-Minute.
* **Universum:** identisch über alle Varianten (gleiche Symbolliste, gleiche Handelstage).
* **Backtest-Engine muss identisch sein**; nur Arbitration/Governance ändert sich.

---

## 3) LLM-Komponente: deterministisch, replayfähig

Damit ein geometrischer Vergleich fair ist:

1. **Freeze:** Prompt-Version, Temperatur=0, Top-p fix.
2. **TTL-Regel identisch** in allen Varianten (sonst unfairer „No-trade“-Bias).
3. **Caching:** pro (Symbol, Timestamp, SnapshotHash) wird der LLM-Output gespeichert und in allen Varianten wiederverwendet.
4. **Validity Gate:** JSON-Schema strikt; invalid → zählt als „LLM unavailable“ und löst in allen Varianten denselben Degraded-Mode aus.

**Zusatztest (Model-Risk):** einmal pro Setup eine „Stochastic Audit“-Serie (z. B. Temp 0.2, 10 Replays) nur um Varianz zu messen (nicht für den Hauptvergleich).

---

## 4) Fill-/Kostenmodell (OHLCV-basiert, konservativ)

Da kein Spread/L2:

* **Fills:** frühestens nächste 1m-Bar nach Intent.
* **Limit-Fill:** wenn High/Low das Limit kreuzt.
* **Stop-Fill:** wenn Low <= StopTrigger, konservativ am Trigger oder nächste Open (Stress-Variante).
* **Kostenmodell:** Kommission + Slippage-Proxy (z. B. Anteil der 1m-Range, abhängig von Volatilitätsregime und Volumenratio).
* **Stressfaktoren:** Slippage × {1, 2, 3} und Latenz {0, 1, 2 Minuten}.

---

## 5) Bewertungsmetriken (geometrisch priorisiert)

### 5.1 Primärmetriken (entscheidend)

1. **Geometric Mean Daily Return**
   ( G = \exp(\text{mean}(\ln(1+r_d))) - 1 )
2. **Max Drawdown (MDD)** auf Equity.
3. **Expected Shortfall (CVaR 95%)** der täglichen Returns (oder log-returns).
4. **Tail-Loss Frequency:** Anteil Tage mit Return < -X% (z. B. -2σ oder feste Schwelle).
5. **Time to Recover / Drawdown Duration** (wichtiger als Volatilität).

### 5.2 Sekundär (Erklärmetriken)

* Profit Factor, Hit Rate, Payoff
* MAE/MFE pro Trade, Giveback (MFE vs realized)
* Trades/Tag, Order Churn, Cancel/Replace Rate
* Anteil „No-Trade Days“ (zu restriktiv kann Opportunity zerstören)

### 5.3 Ein gemeinsamer „Geometric Score“ (für Ranking)

Definiere eine einzige Scorefunktion für Go/No-Go, z. B.:

[
S = \text{mean}(\ln(1+r_d)) ;-; \lambda \cdot \text{ES}_{95}(\ln(1+r_d)) ;-; \mu \cdot \text{MDD}
]

* (\lambda, \mu) so wählen, dass 1 großer Tail-Day den Score spürbar dominiert (geometrische Sicht).

---

## 6) Scenario-/Bucket-Evaluation (Challenger Suite)

Zusätzlich zum „durchschnittlichen“ Backtest wird jeder Tag in Buckets klassifiziert (nur aus OHLCV):

* **Opening Shock Down** (z. B. >X% in Y Minuten + Volumen spike)
* **Parabolic + Plateau + Fade** (Run → Momentum decay → late dump)
* **Compression → Breakout**
* **VWAP Whipsaw Chop**
* **Trend Day Grind Up**
* **Halt/Resume Proxy** (missing bars / große Gaps)

**Für jeden Bucket** reporten:

* Geometric Mean log-return
* Tail-Frequency
* MDD & Recovery Time
* Overtrade Score

Das zeigt, *wo* MC vs Unified gewinnt/verliert (z. B. Unified besser bei „Plateau→Distribution“, MC besser bei „Chop“).

---

## 7) Walk-Forward & Kalibrierung (Overfit-Schutz)

### 7.1 Splits

* Rolling Walk-Forward: z. B. 60 Handelstage Kalibrierung, 20 Handelstage Test; rollierend über 2–3 Jahre.

### 7.2 Was wird kalibriert?

* Nur Schwellen/Parameter, die *alle Varianten teilen* (z. B. crash thresholds, ATR-decay).
* LLM-Arbitrationschwellen (min_conf) dürfen variieren, aber müssen per WF auf OOS optimiert werden.

### 7.3 „Keine Varianten-Optimierung im Test“

Parameter werden pro Fold nur auf dem Train-Fenster gewählt und dann fix auf Test angewandt.

---

## 8) Robustheits- und Stress-Tests (geometrisch kritisch)

Für jede Variante zusätzlich:

1. **Slippage ×2/×3**
2. **Latenz 1–2 Minuten**
3. **Bar-Alignment Shift** (3m/10m Aggregation um 1 Minute versetzt)
4. **Cooldown härter/weicher** (Overtrade-Sensitivität)
5. **LLM Outage Simulation** (z. B. 5%/10% random missing outputs → Degraded Mode)

Erfolgskriterium: Score und Tail-Frequenz dürfen nicht kollabieren.

---

## 9) Statistische Vergleichsmethodik (ohne Selbstbetrug)

* **Paired day comparison:** gleiche Tage, gleiche Symbole → Differenz der daily log returns pro Tag.
* **Bootstrap** über Tage (block bootstrap, um Autokorrelation zu respektieren).
* Reporte:

  * Mittel/Median der Differenzen
  * 5/50/95-Perzentile
  * Wahrscheinlichkeit, dass Unified > MC nach Score (Posterior/Bootstrap)

Wichtig: nicht nur Signifikanz, sondern Effektgröße + Tail-Verhalten.

---

## 10) Entscheidungslogik: welcher Ansatz „gewinnt“?

Du definierst vorab Go/No-Go Kriterien, z. B.:

**MC gewinnt, wenn:**

* Tail-Loss Frequency signifikant geringer und
* Score S stabil unter Stress bleibt,
* Unified keinen klaren Vorteil in „guten“ Buckets liefert.

**Unified gewinnt, wenn:**

* Score S klar höher *und*
* Tail-Loss Frequency nicht steigt (oder sinkt) *und*
* Stress-Tests stabil.

**Typisches realistisches Ergebnis:**
Unified-Veto ist oft der Sweet Spot: mehr Schutz + kaum zusätzliche Tail-Gefahr.

---

## 11) Deliverables (was am Ende rausfallen muss)

1. Ergebnis-Report pro Variante: Score, MDD, ES95, Tail-Freq, Recovery Time.
2. Bucket-Report (Szenarien) + Top 10 Best/Worst Days mit Audit.
3. „Error Budget“: wie oft LLM invalid/stale, wie oft Degraded Mode, Impact auf Performance.
4. Regression Pack: 20 Challenger Days als fixe Abnahmetests.

---

Wenn du willst, kann ich dir das als konkrete Ordner-/Artefaktstruktur (Config-Files, JSON-Schemas, Ergebnis-Tabellenfelder) und als „Testplan“ (Given/When/Then) formulieren, so dass ein Umsetzungsteam direkt damit arbeiten kann.
