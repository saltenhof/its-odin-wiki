## 1) Entscheidungsregel bei fehlender Signifikanz (P&L)

Ja: Bei 140 Handelstagen ist es realistisch, dass **kein signifikanter Unterschied** im täglichen (log-)Return nachweisbar ist, obwohl echte Unterschiede existieren. Deshalb muss die Evaluations-Spec eine **deterministische Default-Entscheidungsregel** enthalten.

### 1.1 Entscheidung als „Gate-Kaskade“ (empfohlen)

**Gate 0 – Safety (hart, geometrisch):** Unified (egal ob Veto/Full) wird **abgelehnt**, wenn es gegenüber MC **Tail-Risiko** klar verschlechtert.
Konkrete Non-Inferiority-Schwellen (Default):

* `MDD_unified > MDD_mc * 1.10`  → FAIL
* `ES95_unified < ES95_mc - 0.0005` (log-return ES, d. h. deutlich schlechter) → FAIL
* `TailLossFreq_unified > TailLossFreq_mc + 20% rel.` → FAIL

**Gate 1 – P&L Signifikanz (wenn vorhanden):**

* Wenn `score_s` (oder avg log-return) Unified > MC **signifikant**, und Gate 0 bestanden → Unified gewinnt.

**Gate 2 – Indifferenz-Fall (keine Signifikanz):**
Dann wird **nicht** “random” entschieden, sondern in folgender Reihenfolge:

1. **FusionLab Proxy** (Regime/Vote-Fusion als Indikator):

   * Wenn FusionLab F2/F3 (decision-level updates / pooling) die **Regime-Accuracy** und/oder **Kostenfunktion** deutlich verbessert **und** Gate 0 bestanden ist → Unified-Veto gewinnt (nicht Full).
2. **Bucket-/Stress-Robustheit**:

   * Wenn Unified in Crash/Whipsaw/Plateau-Fade Buckets **nicht schlechter** ist und in 1–2 Kern-Buckets klar besser (z. B. Parabolic-Fade Giveback) → Unified-Veto gewinnt.
3. **Default-Regel**:

   * Wenn auch das unklar ist: **„Bei Indifferenz gilt MC“** (konservativ, sofort einsatzfähig).

### 1.2 Warum Unified-Veto als „Default-Upgrade“ sinnvoll ist

Wenn du in der Indifferenz bist, ist Unified-Full das höhere Tail-Risiko (mehr Hebel fürs LLM). Unified-Veto liefert typischerweise den Nutzen „Bad trades vermeiden“ bei deutlich geringerem zusätzlichen Risiko.

---

## 2) Drei fehlende Bucket-Schwellen (OHLCV-only, startfähig)

Die Buckets sollen **deterministisch** aus OHLCV ableitbar sein. Alle Schwellen sind **Startwerte**; sie werden per Plateau-Test stabilisiert.

### 2.1 Bucket: Parabolic + Plateau + Fade (der von dir beschriebene „Gewinn abgegeben“-Tag)

**Input:** 1m Bars eines Tages, ADR14 (aus Daily Aggregation), ATR10m/ATR3m.

**Definition (Start):**

1. **Run-up früh:**

   * `time_of_day(high_of_day) <= open + 120min`
   * und `run = (HOD - Open) / Open >= 0.04` **oder** `run >= 0.60 * ADR14`
2. **Plateau:**

   * ab `t_high` mindestens `plateau_minutes >= 30` Minuten, in denen
     `rolling_range_30m <= 0.8 * ATR10m(t_high)`
     (d. h. seitwärts nahe Hoch)
3. **Fade / Giveback:**

   * `giveback = (HOD - Close) / (HOD - Open)`
   * Bucket wenn `giveback >= 0.60`
   * Zusatzbedingung: letzter 60-Minuten Return negativ: `Close - Close(t_close-60m) <= -0.5 * ATR10m`

Damit triffst du genau die Struktur: frühe Rallye, oben „stehen“, dann Abverkauf in den Close.

### 2.2 Bucket: Compression → Breakout

**Ziel:** “Verengung” und dann „explosiver Ausbruch“.

**Definition (Start):**

1. **Compression-Phase (mind. 60 Minuten):**

   * `compression_range = (max_high_60m - min_low_60m)`
   * `baseline_range = (max_high_180m - min_low_180m)`
   * Bucket-Kandidat wenn `compression_range / baseline_range <= 0.35`
   * und `ATR10m_slope_60m < 0` (ATR fällt)
2. **Breakout-Impuls (innerhalb der nächsten 30 Minuten):**

   * `breakout_move = max_high_next30m - max_high_prev60m`
   * Bucket wenn `breakout_move >= 1.5 * ATR10m`
   * und `vol_ratio_10m >= 1.8` (Volumen bestätigt)

### 2.3 Bucket: VWAP Whipsaw (Chop um VWAP)

**Ziel:** viele VWAP-Kreuzungen, wenig Trend.

**Definition (Start, auf 3m Bars):**

1. Zähle VWAP-Crosses in einem 120-Minuten Fenster:

   * `cross_count_120m >= 8` (Signwechsel von close-vwap)
2. Trendlosigkeit:

   * `ADX10m <= 15`
   * und `abs(Close - Open) <= 0.25 * ADR14` (Tag netto klein)

Dieser Bucket ist wichtig, weil er Overtrading entlarvt.

---

## 3) FusionLab: Schema + Ground-Truth-Labeling (ausdesignt)

### 3.1 Tabellen: Ground Truth und Fusion Evaluation

#### A) Ground-Truth Labels pro Decision-Snapshot

```sql
create table if not exists bt_ground_truth (
  snapshot_hash text primary key,
  symbol text not null,
  decision_time timestamptz not null,
  lookforward_minutes int not null default 30,
  atr_ref numeric(20,8) not null,          -- ATR(3m) oder ATR(10m) zum Zeitpunkt t
  c_t numeric(20,8) not null,
  c_t_fwd numeric(20,8) not null,
  fwd_max_high numeric(20,8) not null,
  fwd_min_low numeric(20,8) not null,
  gt_regime text not null,                 -- TREND_UP/TREND_DOWN/RANGE/HIGH_VOL/UNCERTAIN
  gt_conf numeric(6,5) not null,           -- optional, z.B. 1.0 wenn klar
  details jsonb not null default '{}'::jsonb
);

create index if not exists ix_bt_gt_time on bt_ground_truth(symbol, decision_time);
```

#### B) FusionLab Evaluation pro Fusion-Mechanik

```sql
create table if not exists bt_regime_eval (
  run_id bigint not null,
  variant_id bigint not null,              -- MC/Unified etc. (optional)
  fusion_id text not null,                 -- F0/F1/F2/F3
  snapshot_hash text not null references bt_ground_truth(snapshot_hash),
  symbol text not null,
  decision_time timestamptz not null,

  quant_regime text not null,
  llm_regime text,
  fused_regime text not null,
  gt_regime text not null,

  correct boolean not null,
  loss_cost numeric(18,10) not null,       -- Kostenmatrix
  quant_conf numeric(6,5),
  llm_conf numeric(6,5),
  fused_conf numeric(6,5),

  r_quant numeric(6,5),                    -- Reliability zum Zeitpunkt t (wenn verwendet)
  r_llm numeric(6,5),
  meta jsonb not null default '{}'::jsonb,

  primary key (fusion_id, snapshot_hash)
);

create index if not exists ix_bt_re_fusion_time on bt_regime_eval(fusion_id, decision_time);
```

### 3.2 Ground-Truth-Labeling Logik (30-Min Lookforward, ATR-basiert)

**Inputs pro Snapshot t:**

* `C_t` (Close 3m oder 1m, aber konsistent)
* `ATR_ref = ATR(3m,14)` am Zeitpunkt t (nur Vergangenheit)
* In [t, t+30m]: `C_fwd`, `H_fwd_max`, `L_fwd_min`

**Start-Regimes (5 Klassen, bewusst grob):**

1. **TREND_UP**

   * `C_fwd >= C_t + 0.50 * ATR_ref`
   * und `drawdown_in_window = (H_fwd_max - C_fwd) <= 0.60 * ATR_ref` (nicht komplett zurückgegeben)

2. **TREND_DOWN**

   * `C_fwd <= C_t - 0.50 * ATR_ref`

3. **HIGH_VOLATILITY**

   * `range_fwd = H_fwd_max - L_fwd_min >= 2.0 * ATR_ref`
   * und weder TREND_UP noch TREND_DOWN eindeutig (sonst hat Trend Priorität)

4. **RANGE_BOUND**

   * `abs(C_fwd - C_t) <= 0.25 * ATR_ref`
   * und `range_fwd <= 1.0 * ATR_ref`

5. **UNCERTAIN**

   * alles andere (Übergangszonen / Mischformen)

**Tie-Break Reihenfolge (deterministisch):**
TREND_UP / TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > UNCERTAIN

Damit bekommst du ein Ground-Truth-Signal, das stabil ist und ohne L2 funktioniert.

### 3.3 Loss-Cost Matrix (für “geometrische” Fehlerkosten)

Du willst Fehler gewichten, nicht nur Accuracy. Default-Kosten (Start):

* GT=TREND_DOWN, Pred=TREND_UP → **3.0** (gefährlich im Long-only)
* GT=HIGH_VOL, Pred=TREND_UP → **2.0**
* GT=RANGE, Pred=TREND_UP → **1.0**
* GT=TREND_UP, Pred=RANGE → **0.5** (Opportunity-Kosten)
* korrekt → **0.0**

Diese Cost-Matrix ist der Kern, um FusionLab als Proxy für geometrische Robustheit zu nutzen.

---

## Zusammenfassung der Stellungnahme

* **(1) Indifferenz-Fall** braucht eine feste Regel: Gate 0 (Tail nicht schlechter), dann FusionLab/Buckets, sonst Default MC.
* **(2) Bucketschwellen** sind jetzt konkret und OHLCV-only startfähig.
* **(3) FusionLab** ist jetzt als DB-Schema + Ground-Truth-Labeling + Kostenmatrix vollständig spezifiziert.

Wenn du willst, kann ich dir als nächsten Schritt die dazugehörigen **SQL Views** liefern (Accuracy/Loss pro fusion_id, pro Bucket, Worst-Days Drilldown) und die **ETL-Reihenfolge** (welche Tabellen wann befüllt werden) als “Runner Contract”.
