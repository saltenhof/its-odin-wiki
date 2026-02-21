## Testplan mit Postgres als zentrales Ergebnis- und Auswertungs-Backend

Ziel: Alle Backtest-Ergebnisse (Runs, Varianten, Trades, Tagesmetriken, Entscheidungen, LLM/Quant-Artefakte, Szenario-Buckets) so speichern, dass du **geometrische Performance** (log-returns, Tail-Risiko, Drawdowns) robust und reproduzierbar **direkt in Postgres** auswerten kannst.

---

# 1) Postgres-Schema (minimal vollständig, skalierbar)

### 1.1 Kernideen

* **Run** = ein Backtest-Durchlauf mit fixem Datenstand + Zeitfenster + Engine-Version.
* **Variante** = MC / Unified-Veto / Unified-Full (plus Stress-Parameter wie Slippage×2, Latenz, Bar-Shift).
* **Day-Metrics** = pro Variante tägliche Equity/Returns als primäre Basis für geometrische Kennzahlen.
* **Trade/Decision Trace** = optional detailliert, aber wichtig für Debug/Worst-Day-Analysen.
* **LLM Cache** = deterministische Replays: gleicher SnapshotHash ⇒ gleicher Output.

### 1.2 Tabellen (DDL-Vorschlag)

```sql
-- 1) Datensatz-Registry (welche OHLCV-Datenbasis wurde verwendet)
create table if not exists bt_dataset (
  dataset_id bigserial primary key,
  name text not null,
  source text not null,
  start_date date not null,
  end_date date not null,
  ohlcv_granularity text not null default '1m',
  checksum_sha256 text not null,          -- Integritätsanker
  created_at timestamptz not null default now()
);

-- 2) Run-Metadaten
create table if not exists bt_run (
  run_id bigserial primary key,
  dataset_id bigint not null references bt_dataset(dataset_id),
  engine_version text not null,           -- z.B. git commit oder semver
  created_at timestamptz not null default now(),
  notes text,
  global_config jsonb not null            -- z.B. trading hours, calendar, corp action mode
);

-- 3) Varianten innerhalb eines Runs
create table if not exists bt_variant (
  variant_id bigserial primary key,
  run_id bigint not null references bt_run(run_id) on delete cascade,
  variant_name text not null,             -- 'MC', 'UNIFIED_VETO', 'UNIFIED_FULL', 'QUANT_ONLY'
  governance text not null,               -- gleiche Werte wie variant_name, aber "kanonisch"
  prompt_version text,                    -- falls LLM beteiligt
  model_version text,                     -- falls LLM beteiligt
  random_seed int not null default 0,     -- deterministische Teile
  latency_mode text not null default '0m',-- '0m','1m','2m'
  slippage_mode text not null default 'base', -- 'base','x2','x3'
  bar_alignment_shift_minutes int not null default 0, -- 0 oder 1 (Robustheitstest)
  variant_config jsonb not null,          -- alle thresholds/profile-params
  created_at timestamptz not null default now()
);

create index if not exists ix_bt_variant_run on bt_variant(run_id);

-- 4) Daily-Metriken (Primärquelle für geometrische Auswertung)
create table if not exists bt_day (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  trading_date date not null,
  equity_start numeric(20,8) not null,
  equity_end numeric(20,8) not null,
  daily_return numeric(18,10) not null,   -- (end/start - 1)
  log_return numeric(18,10) not null,     -- ln(1+daily_return)
  gross_pnl numeric(20,8) not null,
  net_pnl numeric(20,8) not null,
  fees numeric(20,8) not null,
  slippage_cost numeric(20,8) not null,
  trades_count int not null,
  orders_count int not null,
  churn_score numeric(18,10) not null,    -- definieren (z.B. orders/trades oder flips/day)
  primary key (variant_id, trading_date)
);

create index if not exists ix_bt_day_date on bt_day(trading_date);

-- 5) Trades (für Drill-down + MAE/MFE + R-Multiples)
create table if not exists bt_trade (
  trade_id bigserial primary key,
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  symbol text not null,
  entry_time timestamptz not null,
  exit_time timestamptz not null,
  qty numeric(20,8) not null,
  entry_price numeric(20,8) not null,
  exit_price numeric(20,8) not null,
  initial_stop numeric(20,8),
  pnl_net numeric(20,8) not null,
  r_multiple numeric(18,10),              -- pnl / (entry-stop)*qty
  mfe numeric(20,8),                      -- max favorable excursion (currency)
  mae numeric(20,8),                      -- max adverse excursion (currency)
  cycle_no int not null,                  -- Multi-cycle day
  entry_reason text,                      -- Setup A/B/C/D, etc.
  exit_reason text,                       -- stop/target/tactical/forced_close
  tags jsonb not null default '{}'::jsonb
);

create index if not exists ix_bt_trade_variant_time on bt_trade(variant_id, entry_time);
create index if not exists ix_bt_trade_symbol on bt_trade(symbol);

-- 6) Decision Cycles (3m) für Audit/Debug (optional, aber sehr hilfreich)
create table if not exists bt_decision_cycle (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  symbol text not null,
  decision_time timestamptz not null,     -- 3m decision timestamp
  state text not null,                    -- FSM state
  regime_final text not null,
  intent text not null,                   -- ENTRY/ADD/EXIT/SCALE_OUT/NONE
  reject_reason text,                     -- falls intent verworfen
  quant_vote text,
  quant_conf numeric(6,5),
  llm_vote text,
  llm_conf numeric(6,5),
  confirm_10m text,
  snapshot_hash text not null,            -- referenziert LLM cache
  primary key (variant_id, symbol, decision_time)
);

create index if not exists ix_bt_dc_snapshot on bt_decision_cycle(snapshot_hash);

-- 7) Monitor Events (1m) für Extremfälle/Protect-Mode Analyse
create table if not exists bt_monitor_event (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  symbol text not null,
  event_time timestamptz not null,
  event_type text not null,               -- CRASH_DOWN, SPIKE_UP, EXHAUSTION_CLIMAX, ...
  severity int not null default 1,
  payload jsonb not null default '{}'::jsonb,
  primary key (variant_id, symbol, event_time, event_type)
);

-- 8) LLM Cache (replayfähig)
create table if not exists bt_llm_cache (
  snapshot_hash text primary key,
  prompt_version text not null,
  model_version text not null,
  created_at timestamptz not null default now(),
  valid boolean not null,
  output jsonb,                           -- nur wenn valid
  error text,                             -- wenn invalid/timeout
  latency_ms int
);
```

### 1.3 Szenario-/Bucket-Labels (für Challenger Suite)

```sql
create table if not exists bt_day_bucket (
  variant_id bigint not null references bt_variant(variant_id) on delete cascade,
  trading_date date not null,
  symbol text not null,
  bucket text not null,                   -- z.B. 'PARABOLIC_PLATEAU_FADE'
  bucket_score numeric(18,10),            -- optional: “wie stark” der Bucket
  details jsonb not null default '{}'::jsonb,
  primary key (variant_id, trading_date, symbol, bucket)
);

create index if not exists ix_bt_bucket_bucket on bt_day_bucket(bucket);
```

Hinweis: Falls Datenmengen groß werden, ist **Partitionierung** sinnvoll (z. B. `bt_trade` und `bt_decision_cycle` nach `variant_id` oder nach `trading_date`).

---

# 2) Testplan (Given/When/Then) – mit Postgres als Abnahmequelle

## 2.1 Testgruppenübersicht (Reihenfolge ist Absicht)

1. **Data Integrity & Resampling**
2. **Determinismus & Replay (inkl. LLM Cache)**
3. **Execution/Fill/Kostenmodell**
4. **Risk/Guardrails**
5. **Governance-Logik (MC vs Unified)**
6. **Challenger Suite (20 Szenarien)**
7. **Geometrische KPI-Auswertung (SQL-Checks)**

---

## 2.2 Data Integrity & Resampling

### T1 — Dataset Checksum / Reproduzierbarkeit

**Given** ein importiertes OHLCV-1m Dataset
**When** `bt_dataset.checksum_sha256` berechnet und gespeichert wird
**Then** muss jeder Backtest-Run denselben `dataset_id`+`checksum_sha256` referenzieren, sonst **Run abort**.

### T2 — 1m-Bar Vollständigkeit

**Given** 1m Bars pro Symbol/Tag
**When** Bars validiert werden
**Then**:

* keine fehlenden Minuten in RTH (oder markiert als Gap/Halt)
* keine Duplikate (unique: symbol+timestamp)
* keine OHLC Inkonsistenzen (high < max(open,close) etc.)

### T3 — Resampling Korrektheit (1m→3m, 10m)

**Given** 1m Sequenzen
**When** 3m/10m resampled wird
**Then** muss für jede aggregierte Bar gelten:

* O = Open erster 1m
* C = Close letzter 1m
* H = max High
* L = min Low
* V = sum Volume

---

## 2.3 Determinismus & Replay

### T4 — Deterministische Simulation ohne LLM

**Given** Quant-only Variante D mit fixem Seed
**When** derselbe Run zweimal ausgeführt wird
**Then** müssen `bt_day` und `bt_trade` bitgleich übereinstimmen (Equity-Ende pro Tag identisch).

### T5 — LLM Cache Replay

**Given** eine LLM-Variante (A/B/C) mit `snapshot_hash`
**When** der Run erneut ausgeführt wird
**Then**:

* `bt_llm_cache` wird wiederverwendet (keine neuen Einträge)
* `bt_day` identisch zum ersten Lauf
* bei invalid/stale muss Degraded Mode identisch greifen (keine neuen Entries)

### T6 — LLM Schema Validation

**Given** ein LLM Output, der Schema verletzt
**When** gespeichert werden soll
**Then**:

* `bt_llm_cache.valid=false`, `error` gefüllt
* Variante verhält sich gemäß Policy (z. B. keine neuen Entries)

---

## 2.4 Execution / Fill / Kostenmodell

### T7 — Next-Bar Fill Regel

**Given** ein Entry-Intent am 3m Decision Close
**When** Order simuliert wird
**Then** darf der Fill frühestens in der **nächsten 1m-Bar** stattfinden.

### T8 — Limit Fill OHLCV

**Given** Limitpreis L
**When** nächste 1m-Bar `low<=L<=high`
**Then** fill zum Limit (oder konservativer: worse-of-limit) gemäß konfigurierter Fill-Policy.

### T9 — Stop Fill + Gap

**Given** StopTrigger S
**When** Bar open < S (Gap unter Stop)
**Then** fill zum Open (oder worst-case) und `exit_reason='STOP_GAP'`.

### T10 — Slippage/Latency Stress Modes

**Given** identische Signale
**When** `slippage_mode = x2/x3` oder `latency_mode=1m/2m`
**Then** muss PnL sinken und Tail-Risk tendenziell steigen (Sanity Check), ohne Logikfehler.

---

## 2.5 Risk & Guardrails

### T11 — EOD Flat

**Given** offene Positionen kurz vor Close
**When** Forced Close Window erreicht wird
**Then**:

* alle Positionen geschlossen
* `exit_reason='FORCED_CLOSE'`
* keine Trades danach

### T12 — Max Daily Drawdown Stop

**Given** Tages-DD Limit
**When** überschritten
**Then**:

* `DAY_STOPPED` gesetzt
* keine neuen Trades
* offene Positionen werden gemäß Policy reduziert/geschlossen

### T13 — Max Cycles pro Tag

**Given** `max_cycles_per_day=3`
**When** CycleCounter==3
**Then** `BLOCK_REENTRY` und keine weiteren Entries.

---

## 2.6 Governance Tests: MC vs Unified

### T14 — MC: LLM darf nichts direkt triggern

**Given** MC-Variante A
**When** LLM `STRONG_ENTRY` liefert, Quant aber `NO_TRADE`
**Then** **kein Entry**; nur Tactical Profiles dürfen (wenn erlaubt) Parameter ändern.

### T15 — Unified-Veto: LLM blockt

**Given** Unified-Veto Variante B
**When** Quant `ALLOW_ENTRY`, LLM `NO_TRADE/SAFETY_VETO`
**Then** kein Entry; `reject_reason` muss im `bt_decision_cycle` stehen.

### T16 — Unified-Full: Dual-Key erforderlich

**Given** Unified-Full Variante C
**When** nur eine Seite `ALLOW_ENTRY`
**Then** kein Entry.

### T17 — Exit-Pending + Confirm

**Given** LLM fordert `EXIT_NOW` ohne Struktur-/Event-Confirm
**When** in Unified Varianten
**Then** `PENDING_EXIT` wird gesetzt, Exit nur bei Confirm/Timeout (gemäß Regel).

---

## 2.7 Challenger Suite (20 Szenarien) als Abnahmetests

Für jedes Szenario:

* **Given** Tag/Symbol erfüllt Bucket-Definition (OHLCV-Labeler)
* **When** Variante ausgeführt
* **Then** erwartetes Verhalten wird geprüft (Pass/Fail), z. B.:

  * Plateau-Fade: „Profit Protection greift, Giveback reduziert“
  * Dump→Recovery: „kein frühes Catching falling knife, Reclaim Entry möglich“
  * Coil→Breakout: „Entry nach Confirm, Fakeout-Exit schnell“

Implementationshinweis:

* Der Bucket-Labeler schreibt `bt_day_bucket` pro (variant_id, date, symbol, bucket).

---

# 3) SQL-Auswertung: geometrische Metriken, Drawdowns, ES95

## 3.1 Geometric Mean Daily Return (pro Variante)

```sql
select
  v.variant_id,
  min(v.variant_name) as variant,
  exp(avg(d.log_return)) - 1 as geo_mean_daily_return
from bt_day d
join bt_variant v on v.variant_id = d.variant_id
group by v.variant_id
order by geo_mean_daily_return desc;
```

## 3.2 Max Drawdown (MDD) aus bt_day

```sql
with eq as (
  select
    variant_id,
    trading_date,
    equity_end,
    max(equity_end) over (partition by variant_id order by trading_date) as running_peak
  from bt_day
),
dd as (
  select
    variant_id,
    trading_date,
    (equity_end / nullif(running_peak,0) - 1) as drawdown
  from eq
)
select
  v.variant_id,
  min(v.variant_name) as variant,
  min(drawdown) as max_drawdown
from dd
join bt_variant v on v.variant_id = dd.variant_id
group by v.variant_id
order by max_drawdown asc;
```

## 3.3 ES95 (Expected Shortfall) auf log-returns

```sql
with r as (
  select variant_id, log_return
  from bt_day
),
p as (
  select
    variant_id,
    percentile_cont(0.05) within group (order by log_return) as p05
  from r
  group by variant_id
)
select
  v.variant_id,
  min(v.variant_name) as variant,
  avg(r.log_return) filter (where r.log_return <= p.p05) as es95_log_return
from r
join p on p.variant_id = r.variant_id
join bt_variant v on v.variant_id = r.variant_id
group by v.variant_id
order by es95_log_return asc;
```

## 3.4 Geometric Score S (dein Ranking)

```sql
with base as (
  select variant_id, avg(log_return) as avg_log
  from bt_day
  group by variant_id
),
mdd as (
  with eq as (
    select variant_id, trading_date, equity_end,
           max(equity_end) over (partition by variant_id order by trading_date) as peak
    from bt_day
  )
  select variant_id, min(equity_end/peak - 1) as max_dd
  from eq
  group by variant_id
),
es as (
  with r as (select variant_id, log_return from bt_day),
  p as (
    select variant_id,
           percentile_cont(0.05) within group (order by log_return) as p05
    from r group by variant_id
  )
  select r.variant_id,
         avg(r.log_return) filter (where r.log_return <= p.p05) as es95
  from r join p using (variant_id)
  group by r.variant_id
)
select
  v.variant_id,
  min(v.variant_name) as variant,
  b.avg_log,
  es.es95,
  mdd.max_dd,
  (b.avg_log - (0.8 * es.es95) - (1.2 * abs(mdd.max_dd))) as score_s
from base b
join es using (variant_id)
join mdd using (variant_id)
join bt_variant v on v.variant_id = b.variant_id
group by v.variant_id, b.avg_log, es.es95, mdd.max_dd
order by score_s desc;
```

(* 0.8/1.2 sind Platzhalter für λ/μ; du setzt sie so, dass Tail-Loss spürbar dominiert.)

---

# 4) Paired Day Comparison (MC vs Unified) direkt in SQL

```sql
with mc as (
  select trading_date, log_return
  from bt_day d join bt_variant v using (variant_id)
  where v.variant_name = 'MC'
),
uv as (
  select trading_date, log_return
  from bt_day d join bt_variant v using (variant_id)
  where v.variant_name = 'UNIFIED_VETO'
)
select
  mc.trading_date,
  (uv.log_return - mc.log_return) as delta_log_return
from mc
join uv using (trading_date)
order by mc.trading_date;
```

Damit kannst du Worst-Days identifizieren und dann per `bt_trade` / `bt_decision_cycle` drillen.

---

# 5) Output-Disziplin (damit das Team “bauen kann”)

Für jede Testausführung MUSS der Runner am Ende:

1. `bt_run` anlegen
2. pro Variante `bt_variant` anlegen
3. pro Tag `bt_day` schreiben (mindestens)
4. optional: Trades/Decision Cycles für Debug-Level schreiben
5. Bucket-Labels erzeugen und in `bt_day_bucket` persistieren
6. Standard-Views/Queries laufen lassen und einen “Run Summary” Report abspeichern (z. B. als JSON in `bt_run.global_config` oder separate `bt_run_summary`-Tabelle)

Wenn du willst, liefere ich als nächsten Schritt ein **ReasonCode-/EventType-Vokabular** (Enums) und eine minimale Sammlung von **SQL Views** (z. B. `v_bt_variant_summary`, `v_bt_bucket_summary`, `v_bt_worst_days`) als „Standardauswertungspaket“.
