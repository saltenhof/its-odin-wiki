Das Feedback trifft wesentliche Punkte. Hier ist die Stellungnahme – plus die konkreten Ergänzungen, damit das Evaluationsdesign implementierbar und statistisch belastbarer wird.

---

## 1) Regime-Fusion als eigener Testgegenstand (statt “Sub-Experiment”)

Ja: Wenn Reliability-weighted Fusion (Build-Spec Kap. 27) nur als Teil von “Variante C” läuft, kannst du Verbesserungen wie „Decision-Level Updates“ oder „hierarchisches Pooling“ nicht sauber isolieren.

**Ergänzung im Evaluationsdesign: eigener “FusionLab”-Versuchsblock** (unabhängig vom Trading-P&L):

**Ziel:** Regime-/Vote-Fusion messen, ohne dass Execution/Stops/Targets alles überdeckt.

**Setup:**

* Input: gespeicherte Snapshots pro 3-Minuten-Decision-Zeitpunkt (inkl. Quant-Output und LLM-Output aus Cache).
* Ground-Truth: das ex-post 30-Minuten-Labeling (das ist bereits als Konzept vorhanden). 
* Output: Regime-Accuracy/Kalibrierung und *Fehlerkosten* (z. B. “TREND_UP vorhergesagt, aber später TREND_DOWN”).

**FusionLab-Varianten (nur Fusion unterscheidet sich):**

* F0: MC-Regel (KPI-first, konservativ bei Widerspruch) – sofort einsatzfähig.
* F1: Reliability-weighted (EOD-Update).
* F2: Reliability-weighted (Decision-Level-Update statt 1×/Tag).
* F3: Reliability-weighted + hierarchisches Pooling (globaler Score + Regime-Offsets).

Damit ist das angesprochene Problem (1) sauber adressiert.

---

## 2) Konkrete Schwellenwerte (λ/μ, Bucket-Schwellen)

Richtig: Ohne Default-Werte ist es nicht ausführbar. Lösung: **Startwerte + Kalibrierpfad** (Plateau-Test), so wie es in den Konzepten sowieso als Best Practice gefordert wird.  

### 2.1 Startwerte für Geometric Score (λ/μ)

Score:
(S = \overline{\log(1+r_d)} - \lambda\cdot |ES_{95}| - \mu\cdot |MDD|)

**Startwerte (praktisch, geometrisch konservativ):**

* λ = **1.0**
* μ = **2.0** (MDD stärker bestrafen als ES95, weil Drawdown geometrisch dominiert)

**Kalibrierpfad (verhindert Overfit):**

* Grid nur auf Train-Fenstern:

  * λ ∈ {0.5, 1.0, 1.5, 2.0}
  * μ ∈ {1.0, 2.0, 3.0}
* Auswahlregel: nicht “Bestpunkt”, sondern **Plateau** (robust gegen ±10%). 

### 2.2 Startwerte für Bucket-Klassifikation (OHLCV-only)

Hier kannst du dich an bestehenden Konzept-Schwellen orientieren, statt “X/Y frei”:

* **Crash/Spike (1m Event):** Preis > **5%** in < 1 Minute **und** Volume > **3×** Durchschnitt → CrashSignal (nicht verwerfen). 
* **ATR-Decay Trigger:** Ratio < **0.40** (Decays > 60%) → taktischer Switch. 

Für “Opening Shock” empfehle ich als Start:

* innerhalb der ersten 10 Minuten: `drawdown_from_open <= -4%` **oder** (`<= -3%` und `vol_ratio >= 3.0`)
  (weil du kein L2 hast, muss Volumen als Bestätigung dienen)

Wichtig: diese Schwellen sind Startwerte; die Bucket-Labeler werden genauso per Plateau-Test stabilisiert.

---

## 3) Sample-Size / Power: was realistisch nachweisbar ist

Das Feedback ist korrekt: Ohne grobe Power-Abschätzung ist “Konfidenz” unklar.

### 3.1 Für P&L-Unterschiede (daily log returns)

Für einen **paired day test** (gleiche Tage, gleiche Symbole) gilt grob:

[
n \approx \left(\frac{(z_{\alpha/2}+z_{\beta})\cdot \sigma_{\Delta}}{\delta}\right)^2
]

* (\delta) = gewünschter Effekt (z. B. 10 bps täglicher log-Return Vorteil = 0.001)
* (\sigma_{\Delta}) = Stddev der Tages-Differenzen zwischen zwei Varianten (typisch 0.4%–1.0%, je nach Strategie)

**Grobe Orientierung (α=5%, Power 80%, z≈2.8):**

* wenn (\sigma_{\Delta}=0.6%):

  * δ=10 bps → n≈ **282 Handelstage**
  * δ=5 bps → n≈ **1129 Handelstage**
* wenn (\sigma_{\Delta}=1.0%):

  * δ=10 bps → n≈ **784 Handelstage**

Konsequenz: kleine Verbesserungen in Geo-Return sind schwer “sauber” zu beweisen.

### 3.2 Shortcut: Fusion/Regime-Qualität braucht viel weniger Tage

Darum der FusionLab-Block: du hast pro Tag viele 3-Minuten-Entscheidungen → viel mehr Samples, und Ground-Truth-Labeling ist ohnehin 30-Minuten-ex-post definiert. 
Damit kannst du Fusion-Verbesserungen deutlich schneller statistisch sehen, bevor du sie in P&L “beweisen” musst.

**Zusatz:** Im Fachkonzept wird selbst die erste Kalibrierprüfung erst nach **100 Handelstagen** als sinnvoll bezeichnet.  Das sollte als Mindestgröße für “reliability calibration claims” gelten.

---

## 4) Brücke zur vorhandenen Infrastruktur (Runner, CachedAnalyst, DB)

Das ist ein valider Kritikpunkt – muss im Plan explizit werden.

Die vorhandene Architektur beschreibt bereits:

* Simulation über **SimClock**, **HistoricalMarketDataFeed**, **BrokerSimulator**
* LLM im Backtest über **CachedAnalyst**
* Logging über **PostgresEventLog** (live + sim). 

Und es ist normativ gefordert, dass LLM-Responses für Replay gecacht werden (Cache-Key Hash aus Prompt+Kontext+Payload) – das passt direkt zu einem DB-basierten Evaluationssetup. 
Außerdem: Im Backtesting **muss** der CachedAnalyst verwendet werden für deterministische Reproduzierbarkeit. 

**Ergänzung im Evaluationsdesign:**

* BacktestRunner schreibt pro Run/Variante die Kerntabellen (bt_run/bt_variant/bt_day/bt_trade/bt_decision_cycle).
* CachedAnalyst bedient sich aus `bt_llm_cache` (oder kompatiblem Cache-Table), so dass die Varianten **dieselben** LLM-Outputs wiederverwenden.
* Zusätzlich FusionLab schreibt eine eigene Tabelle `bt_regime_eval` (snapshot_hash, quant_regime, llm_regime, fused_regime, ground_truth, loss_cost).

Wenn du schon eine bestehende Postgres-EventLog-Struktur hast: Dann ist der pragmatische Weg **Views** zu bauen, die dein EventLog auf diese Auswertungsschemata projizieren (statt alles doppelt zu speichern).

---

## 5) Datenbedarf & LLM-Kosten (und wie man das praktisch klein hält)

Richtig: “2–3 Jahre” pauschal ist unscharf.

**Korrektur im Plan (stufenweise, kostensparend):**

### Phase 0 — Ohne LLM (billig, schnell)

* Quant-only auf dem maximal verfügbaren Zeitraum (OHLCV ist günstig).
* Ziel: Bucket-Labeler stabilisieren, Kostenmodell stabilisieren, Baseline-Volatilität der daily returns schätzen (für Power-Annahmen).

### Phase 1 — LLM nur auf Subset + Cache (einmal zahlen, oft nutzen)

* LLM-Calls eventgetrieben / im festen Takt (z. B. alle 6–12 Minuten), und **einmalig** in Cache persistieren. 
* Dann: **alle** Governance-Varianten (MC/Unified-Veto/Unified-Full + FusionLab F0–F3) laufen im Replay über den gleichen Cache → Kosten steigen **nicht** linear mit Varianten.

### Phase 2 — Walk-Forward Umfang realistisch ableiten

Walk-Forward ist in den Konzepten festgelegt (60/20 rollierend, “3 von 4 profitabel”).  
Das Minimum für 4 Validierungsfenster ist tatsächlich ~140 Handelstage. Das ist als “Research-Minimum” ok, aber für harte statistische Claims bei P&L oft nicht genug → deshalb FusionLab + Bucket-Stabilität + Stress-Tests als zusätzliche Evidenz.

---

## Ergebnis: wie der Plan nach den Ergänzungen aussieht (kurz)

1. **FusionLab (F0–F3)**: isoliert Regime-Fusion & Reliability-Mechanik (schnell, viele Samples).
2. **Governance-Backtests (MC vs Unified-Veto vs Unified-Full)**: P&L-Vergleich mit fixierten λ/μ Startwerten + Plateau-Kalibrierpfad.
3. **Power-Reporting**: grobe Effekt-/n-Tabelle aus Phase-0 Daten (σΔ gemessen, nicht geraten).
4. **Infrastruktur-Fit**: BacktestRunner + CachedAnalyst + PostgresEventLog/Cache werden explizit genutzt.  
 
