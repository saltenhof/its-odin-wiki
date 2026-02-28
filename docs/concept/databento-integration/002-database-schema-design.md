# Datenbank-Schema-Erweiterung — Databento-Integration

> Status: Konzept | Erstellt: 2026-02-28
> LLM-Sparring: Claude (Lead), ChatGPT, Gemini — alle drei empfehlen übereinstimmend Option A

## 1. Design-Entscheidungen

### 1.1 Architektur: Separate Tabellen (Option A)

**Entscheidung:** Quotes (MBP-1/BBO) und Trades werden in **getrennten Hypertables** gespeichert.

**Begründung (Konsens aller drei LLMs):**

- Quotes und Trades haben **strukturell unterschiedliche Felder** — eine Unified Table erzwingt
  viele NULL-Spalten, verschlechtert Compression und macht Queries ineffizienter
- **Unterschiedliche Query-Patterns:** Quotes → "letzter BBO in Zeitfenster";
  Trades → Preis/Size-Aggregationen, Execution-Statistiken
- **Unabhängiges Tuning** von Compression, Indexes und Retention pro Stream
- **Unified Replay** bei Bedarf über eine View (`UNION ALL`) — logisch vereint,
  physisch getrennt

**Verworfen:**
- Option B (Unified Table): Sparse Columns, schlechtere Compression, ineffiziente Scans
- Option C (Nur Sampling): Macht den Kauf professioneller Tick-Daten obsolet.
  Für Slippage-Modellierung und Limit-Order-Simulation brauchen wir die echte Tick-Sequenz

### 1.2 Preise: BIGINT Fixed-Point (1e-9)

**Entscheidung:** Preise als `BIGINT` im Databento-Originalformat (1 Unit = 10^-9).

**Begründung:**
- **Verlustfrei:** Keine Rundungsfehler durch Float-Konvertierung
- **Bessere Compression:** TimescaleDB komprimiert BIGINT-Zeitreihen mit
  Delta-of-Delta-Kodierung extrem effizient
- **Konsistenz:** Databento liefert int64 — direktes Speichern ohne Konvertierung
- **Konvertierung bei Bedarf:** Views liefern `double precision` für Queries

### 1.3 Timestamps: TIMESTAMPTZ + BIGINT Nanosekunden

**Entscheidung:** Zwei Zeitfelder pro Record:

| Feld | Typ | Zweck |
|------|-----|-------|
| `ts_event` | `TIMESTAMPTZ` | Hypertable-Partitionierung, TimescaleDB-Funktionen, SQL-Queries |
| `ts_event_ns` | `BIGINT` | Exakte Nanosekunden-Ordnung, Reproduzierbarkeit im Backtest |

**Begründung:**
- PostgreSQL `TIMESTAMPTZ` hat nur Mikrosekunden-Präzision (10^-6)
- Für Tick-Daten mit potenziell mehreren Events pro Mikrosekunde ist exakte
  Nanosekunden-Ordnung nötig (Tie-Breaking)
- `TIMESTAMPTZ` bleibt nötig für native TimescaleDB-Funktionen
  (`time_bucket`, `continuous aggregates`, Chunk Exclusion)
- Gemini-Alternative (reiner BIGINT als Hypertable-Dimension) verworfen wegen
  schlechterer Ergonomie bei SQL-Queries

### 1.4 Partitionierung: 1-Day Chunks

**Entscheidung:** `chunk_time_interval => INTERVAL '1 day'` für alle Tick-Tabellen.

**Begründung:**
- ODIN-Backtests operieren tageweise (ein Handelstag = eine Backtest-Einheit)
- TimescaleDB Chunk Exclusion greift perfekt bei Tagesgrenzen-Queries
- ~500k Records pro Chunk → wenige MB pro Chunk → optimal für RAM-Budget

### 1.5 Symbole: TEXT (konsistent mit bestehendem Schema)

**Entscheidung:** `symbol TEXT` statt numerischer Instrument-IDs.

**Begründung:**
- Bestehende `odin.intraday_bar` verwendet `symbol TEXT` → Konsistenz
- Für das ODIN-Datenvolumen (selektive Downloads, ~10-20 Instrumente) ist der
  Overhead von TEXT vs. INTEGER vernachlässigbar
- Vermeidet eine zusätzliche Dimension-/Mapping-Tabelle im kritischen Pfad
- Databento's `instrument_id` ist Venue-spezifisch und kann sich ändern

---

## 2. Schema-Entwurf

### 2.1 Bestehend: `odin.intraday_bar` (unverändert)

Die bestehende Tabelle bleibt für OHLCV-Bars. Databento-Daten werden über
`source = 'DATABENTO'` unterschieden von Yahoo-Daten (`source = 'YAHOO'`).
Keine Strukturänderung nötig.

### 2.2 Neu: `odin.quote_tick` (Level 1 BBO)

Speichert jeden Top-of-Book-Update aus dem MBP-1 Schema.

```sql
CREATE TABLE IF NOT EXISTS odin.quote_tick (
    -- Zeitdimension (dual: TIMESTAMPTZ für TimescaleDB + BIGINT für exakte Ordnung)
    ts_event            TIMESTAMPTZ     NOT NULL,
    ts_event_ns         BIGINT          NOT NULL,

    -- Instrument
    symbol              TEXT            NOT NULL,
    exchange            TEXT            NOT NULL,
    trading_day         DATE            NOT NULL,

    -- Event-Kontext
    action              CHAR(1)         NOT NULL,    -- A/C/M/R/T/F/N
    side                CHAR(1)         NOT NULL,    -- A(sk)/B(id)/N(one)
    flags               SMALLINT        NOT NULL DEFAULT 0,
    sequence            BIGINT          NOT NULL,
    publisher_id        SMALLINT        NOT NULL,

    -- Order-Daten (die auslösende Order)
    price               BIGINT          NOT NULL,    -- Fixed-Point 1e-9
    size                INTEGER         NOT NULL,

    -- Best Bid/Offer Snapshot
    bid_px              BIGINT,                      -- Fixed-Point 1e-9
    ask_px              BIGINT,
    bid_sz              INTEGER,
    ask_sz              INTEGER,
    bid_ct              INTEGER,
    ask_ct              INTEGER,

    -- Capture-Timestamp
    ts_recv             TIMESTAMPTZ,
    ts_recv_ns          BIGINT,

    -- Provenienz
    source              TEXT            NOT NULL DEFAULT 'DATABENTO',
    ingested_at         TIMESTAMPTZ     NOT NULL DEFAULT now()
);

-- TimescaleDB Hypertable: 1-Day Chunks
SELECT create_hypertable('odin.quote_tick', 'ts_event',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Primärer Zugriffsindex: Instrument + Zeit (Backtest-Replay)
CREATE INDEX IF NOT EXISTS idx_quote_tick_replay
    ON odin.quote_tick (symbol, ts_event, sequence);

-- Tages-Lookup (für "gibt es Daten für diesen Tag?")
CREATE INDEX IF NOT EXISTS idx_quote_tick_day
    ON odin.quote_tick (trading_day, symbol);

-- Deduplizierung bei Re-Import
CREATE UNIQUE INDEX IF NOT EXISTS idx_quote_tick_dedup
    ON odin.quote_tick (symbol, ts_event_ns, sequence, publisher_id);

-- TimescaleDB Compression
ALTER TABLE odin.quote_tick SET (
    timescaledb.compress,
    timescaledb.compress_segmentby  = 'symbol',
    timescaledb.compress_orderby    = 'ts_event, sequence'
);

-- Automatische Compression nach 3 Tagen (Backtest-Daten sind Read-Only)
SELECT add_compression_policy('odin.quote_tick', INTERVAL '3 days');
```

### 2.3 Neu: `odin.trade_tick` (Trades / Time & Sales)

Speichert jeden einzelnen Trade (Ausführung).

```sql
CREATE TABLE IF NOT EXISTS odin.trade_tick (
    -- Zeitdimension
    ts_event            TIMESTAMPTZ     NOT NULL,
    ts_event_ns         BIGINT          NOT NULL,

    -- Instrument
    symbol              TEXT            NOT NULL,
    exchange            TEXT            NOT NULL,
    trading_day         DATE            NOT NULL,

    -- Trade-Daten
    price               BIGINT          NOT NULL,    -- Fixed-Point 1e-9
    size                INTEGER         NOT NULL,
    side                CHAR(1)         NOT NULL,    -- Aggressor: A(sk-Taker=Buyer) / B(id-Taker=Seller)
    flags               SMALLINT        NOT NULL DEFAULT 0,
    sequence            BIGINT          NOT NULL,
    publisher_id        SMALLINT        NOT NULL,

    -- Capture-Timestamp
    ts_recv             TIMESTAMPTZ,
    ts_recv_ns          BIGINT,

    -- Provenienz
    source              TEXT            NOT NULL DEFAULT 'DATABENTO',
    ingested_at         TIMESTAMPTZ     NOT NULL DEFAULT now()
);

-- TimescaleDB Hypertable: 1-Day Chunks
SELECT create_hypertable('odin.trade_tick', 'ts_event',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Primärer Zugriffsindex: Instrument + Zeit (Backtest-Replay)
CREATE INDEX IF NOT EXISTS idx_trade_tick_replay
    ON odin.trade_tick (symbol, ts_event, sequence);

-- Tages-Lookup
CREATE INDEX IF NOT EXISTS idx_trade_tick_day
    ON odin.trade_tick (trading_day, symbol);

-- Deduplizierung
CREATE UNIQUE INDEX IF NOT EXISTS idx_trade_tick_dedup
    ON odin.trade_tick (symbol, ts_event_ns, sequence, publisher_id);

-- TimescaleDB Compression
ALTER TABLE odin.trade_tick SET (
    timescaledb.compress,
    timescaledb.compress_segmentby  = 'symbol',
    timescaledb.compress_orderby    = 'ts_event, sequence'
);

SELECT add_compression_policy('odin.trade_tick', INTERVAL '3 days');
```

### 2.4 Neu: `odin.data_ingest_log` (Import-Audit)

Protokolliert jeden Datenimport für Reproduzierbarkeit und Idempotenz.

```sql
CREATE TABLE IF NOT EXISTS odin.data_ingest_log (
    ingest_id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    provider            TEXT            NOT NULL,    -- 'DATABENTO', 'YAHOO'
    dataset             TEXT,                        -- 'XNAS.ITCH', NULL für Yahoo
    schema_name         TEXT            NOT NULL,    -- 'mbp-1', 'trades', 'ohlcv-1m'
    symbol              TEXT            NOT NULL,
    trading_day         DATE            NOT NULL,
    row_count           BIGINT,
    cost_usd            NUMERIC(10,4),               -- API-Kosten
    file_hash           TEXT,                        -- SHA-256 der Quelldatei
    status              TEXT            NOT NULL DEFAULT 'COMPLETED',
    error_message       TEXT,
    started_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,

    UNIQUE (provider, dataset, schema_name, symbol, trading_day)
);
```

### 2.5 Convenience-Views

```sql
-- Preise als Double für SQL-Queries
CREATE OR REPLACE VIEW odin.v_quote_tick AS
SELECT
    ts_event, ts_event_ns, symbol, exchange, trading_day,
    action, side, flags, sequence,
    price   / 1e9::double precision AS price_f64,
    size,
    bid_px  / 1e9::double precision AS bid_f64,
    ask_px  / 1e9::double precision AS ask_f64,
    bid_sz, ask_sz, bid_ct, ask_ct,
    (ask_px - bid_px) / 1e9::double precision AS spread_f64,
    ts_recv
FROM odin.quote_tick;

CREATE OR REPLACE VIEW odin.v_trade_tick AS
SELECT
    ts_event, ts_event_ns, symbol, exchange, trading_day,
    price   / 1e9::double precision AS price_f64,
    size, side, flags, sequence,
    ts_recv
FROM odin.trade_tick;

-- Unified Replay Stream für Backtester
CREATE OR REPLACE VIEW odin.v_tick_stream AS
SELECT
    ts_event, ts_event_ns, symbol,
    'QUOTE'::text AS tick_type,
    price   / 1e9::double precision AS price_f64,
    size,
    side,
    bid_px  / 1e9::double precision AS bid_f64,
    ask_px  / 1e9::double precision AS ask_f64,
    bid_sz, ask_sz,
    sequence
FROM odin.quote_tick
UNION ALL
SELECT
    ts_event, ts_event_ns, symbol,
    'TRADE'::text AS tick_type,
    price   / 1e9::double precision AS price_f64,
    size,
    side,
    NULL::double precision AS bid_f64,
    NULL::double precision AS ask_f64,
    NULL::integer AS bid_sz,
    NULL::integer AS ask_sz,
    sequence
FROM odin.trade_tick;
```

---

## 3. TimescaleDB-Optimierung

### 3.1 Compression-Strategie

| Einstellung | Wert | Begründung |
|------------|------|-----------|
| `compress_segmentby` | `symbol` | Queries filtern fast immer nach Instrument |
| `compress_orderby` | `ts_event, sequence` | Zeitgeordneter Scan für Replay |
| Compression Policy | Nach 3 Tagen | Backtest-Daten sind nach Import Read-Only |

**Erwartete Compression-Ratio:** 10:1 bis 20:1 für Tick-Daten (Delta-of-Delta auf BIGINT-Preisen
und Timestamps, RLE auf wiederholten Symbolen/Flags).

### 3.2 Index-Strategie

Minimale Indexierung — nur das Nötigste:

| Index | Spalten | Zweck |
|-------|---------|-------|
| Replay | `(symbol, ts_event, sequence)` | Backtest: "Alle Ticks für IREN am 23.02." |
| Day-Lookup | `(trading_day, symbol)` | "Haben wir Daten für diesen Tag?" |
| Dedup | `(symbol, ts_event_ns, sequence, publisher_id)` UNIQUE | Re-Import-Sicherheit |

**Bewusst kein** Index auf `action`, `side`, `flags` — zu selten als Filter genutzt,
würde nur Insert-Performance kosten.

### 3.3 Batch-Insert-Strategie

- **COPY** (PostgreSQL native) für maximale Insert-Performance
- Mindestens 10.000 Rows pro Batch bei JDBC-Inserts
- Niemals Row-by-Row Inserts für Tick-Daten

---

## 4. Datenfluss: Bestehende vs. neue Tabellen

```
                    ┌─────────────────────┐
                    │    Databento API     │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │  DatabentoCl ient    │
                    │  (Java, HTTP/CSV)    │
                    └────┬────┬────┬──────┘
                         │    │    │
              ┌──────────┘    │    └──────────┐
              ▼               ▼               ▼
    ┌─────────────────┐ ┌──────────┐ ┌───────────────┐
    │ odin.quote_tick  │ │odin.trade│ │odin.intraday_ │
    │ (MBP-1 / BBO)   │ │  _tick   │ │    bar        │
    │ TimescaleDB HT   │ │ TS HT   │ │ TS HT (exist) │
    └────────┬────────┘ └────┬─────┘ └───────┬───────┘
             │               │               │
             └───────┬───────┘               │
                     ▼                       │
           ┌─────────────────┐               │
           │ v_tick_stream   │               │
           │ (Unified View)  │               │
           └────────┬────────┘               │
                    │                        │
                    ▼                        ▼
           ┌────────────────────────────────────┐
           │        Backtesting Pipeline         │
           │   (HistoricalMarketDataFeed etc.)   │
           └────────────────────────────────────┘
```

### 4.1 Koexistenz mit Yahoo-Daten

- `odin.intraday_bar` bleibt unverändert
- Yahoo-Daten: `source = 'YAHOO'` (volume = 0 für Pre/After-Market)
- Databento OHLCV: `source = 'DATABENTO'` (echtes Volume für alle Sessions)
- Databento-Bars werden in die **bestehende** `odin.intraday_bar` geschrieben —
  kein neues Schema nötig
- `ON CONFLICT (symbol, bar_interval_sec, open_time) DO NOTHING` für Idempotenz

---

## 5. Downsampling-Strategie

### 5.1 Zwei Stufen

| Stufe | Datenmenge | Zweck |
|-------|-----------|-------|
| **Raw MBP-1** | ~100k–500k Ticks/Tag | Mikrostruktur-Analyse, Spread-Dynamik, L1-Replay |
| **1s BBO Snapshot** | ~23k–60k Rows/Tag | Schnelle Backtests, Spread-Approximation |

### 5.2 Empfohlenes Vorgehen

1. **Default:** Immer alle MBP-1 Ticks speichern (Speicher ist kein Engpass)
2. **Optional:** 1-Sekunden-BBO als TimescaleDB Continuous Aggregate ableiten:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS odin.ca_bbo_1s
WITH (timescaledb.continuous) AS
SELECT
    time_bucket(INTERVAL '1 second', ts_event) AS bucket,
    symbol,
    last(bid_px, ts_event) AS bid_px,
    last(ask_px, ts_event) AS ask_px,
    last(bid_sz, ts_event) AS bid_sz,
    last(ask_sz, ts_event) AS ask_sz
FROM odin.quote_tick
GROUP BY bucket, symbol;
```

3. **Retention:** Keine automatische Löschung — Backtest-Daten sind Investition.
   Bei Platzmangel: manuell alte Daten für nicht mehr benötigte Instrumente/Tage entfernen.

---

## 6. Migration: Flyway-Migrationen

Folgende Flyway-Migrationen sind für die Implementierung nötig:

| Migration | Inhalt |
|-----------|--------|
| `V030__create_quote_tick.sql` | `odin.quote_tick` + Hypertable + Indexes + Compression |
| `V031__create_trade_tick.sql` | `odin.trade_tick` + Hypertable + Indexes + Compression |
| `V032__create_data_ingest_log.sql` | `odin.data_ingest_log` |
| `V033__create_tick_views.sql` | `v_quote_tick`, `v_trade_tick`, `v_tick_stream` |

**Hinweis:** TimescaleDB-Stub in Test-Modulen muss um `add_compression_policy`-Stub erweitert
werden, analog zum bestehenden `create_hypertable`-Stub.
