# Databento API — Exploration & Schnittstellenanalyse

> Status: Konzept | Erstellt: 2026-02-28

## 1. Databento im Überblick

Databento ist ein professioneller Marktdaten-Provider mit Fokus auf institutionelle Qualität.
Im Gegensatz zu Yahoo Finance liefert Databento:

- **Nanosekunden-Timestamps** (Exchange-Timestamps, nicht Polling-basiert)
- **Bid/Ask-Daten** (Level 1 BBO, Level 2 Top-10, Level 3 Full Depth)
- **Vollständige Volume-Daten** inklusive Pre-Market und After-Hours
- **Tick-by-Tick Trades** mit Aggressor-Seite
- **Professionelle OHLCV-Bars** mit echtem Volumen

### 1.1 Verfügbare Client-Bibliotheken

| Sprache | Offizieller Client | Status |
|---------|-------------------|--------|
| Python  | `databento` (PyPI) | Offiziell, vollständig |
| Rust    | `databento` (crates.io) | Offiziell, vollständig |
| C++     | `databento-cpp` | Offiziell |
| **Java** | **Nicht vorhanden** | **HTTP REST API als Alternative** |
| Go      | `dbn-go` (Community) | Inoffiziell, nur DBN-Parsing |

**Konsequenz für ODIN:** Kein offizieller Java-Client. Zwei Optionen:
1. HTTP REST API direkt aus Java ansprechen (empfohlen)
2. Python-Bridge: Databento-Daten per Python herunterladen, per JDBC/COPY in PostgreSQL laden

### 1.2 Authentifizierung

- API-Key-basiert (32-Zeichen, Präfix `db-`)
- HTTP Basic Auth: Username = API-Key, Passwort leer
- Alternativ: `Authorization: Basic <base64(apikey:)>` Header

### 1.3 Preismodell

- **$125 kostenlose Credits** bei Registrierung
- Pay-as-you-go, kein Abo erforderlich
- Kosten abhängig von Schema, Datenmenge, Venue
- Beispiel: 1.7M Trade-Ticks (5 Tage, ES Futures) ≈ $2.17
- Kostenschätzung vorab möglich: `metadata.get_cost` API-Endpoint

---

## 2. Schemas (Datenformate)

Databento organisiert Marktdaten in **Schemas** — jedes Schema liefert eine andere Granularität.

### 2.1 Schema-Übersicht

| Schema | Beschreibung | Granularität | ODIN-Relevanz |
|--------|-------------|-------------|---------------|
| **MBO** | Market By Order (L3) | Einzelne Orders | Out of Scope |
| **MBP-1** | Market By Price, 1 Level (L1 BBO) | Top-of-Book Updates | **Kernbedarf** |
| **MBP-10** | Market By Price, 10 Levels (L2) | Top-10 Preislevel | Optional/Zukunft |
| **TBBO** | Trade + Best Bid/Offer | Trade mit BBO-Kontext | **Sehr relevant** |
| **Trades** | Tick-by-Tick Trades | Jede Ausführung | **Kernbedarf** |
| **OHLCV-1s/1m/1h/1d** | Aggregierte Bars | 1s bis 1d Intervalle | **Kernbedarf** |
| Definition | Instrumenten-Metadaten | Statisch | Hilfstabelle |
| Statistics | Venue-Statistiken | Tagesbasis | Nachrangig |
| Imbalance | Auktions-Imbalance | Event-basiert | Nachrangig |

### 2.2 MBP-1 Record (Level 1 — Top of Book)

Das zentrale Schema für Bid/Ask-Daten. Jeder Record beschreibt eine Änderung am Top-of-Book.

```
Mbp1Msg {
  Header:
    ts_event        uint64    // Exchange-Timestamp (Nanosekunden seit Epoch)
    rtype           uint8     // Record-Typ
    publisher_id    uint16    // Publisher/Venue-ID
    instrument_id   uint32    // Numerische Instrument-ID (Databento-intern)

  Body:
    price           int64     // Orderpreis (Fixed-Point, Skalierung 1e-9)
    size            uint32    // Ordergröße
    action          byte      // A=Add, C=Cancel, M=Modify, R=Clear, T=Trade, F=Fill, N=None
    side            byte      // A=Ask, B=Bid, N=None
    flags           uint8     // Bit-Flags (F_LAST, F_TOB, F_SNAPSHOT, etc.)
    ts_recv         uint64    // Capture-Server-Timestamp (ns)

  Level[0] (BidAskPair):
    bid_px          int64     // Best Bid Preis (Fixed-Point 1e-9)
    ask_px          int64     // Best Ask Preis (Fixed-Point 1e-9)
    bid_sz          uint32    // Best Bid Größe
    ask_sz          uint32    // Best Ask Größe
    bid_ct          uint32    // Anzahl Bid-Orders auf diesem Level
    ask_ct          uint32    // Anzahl Ask-Orders auf diesem Level
}
```

**Preiskonvertierung:** `double_price = fixed_price / 1_000_000_000.0`

**Datenvolumen:** ~100k–500k Records pro Instrument pro Handelstag (Aktien, NASDAQ)

### 2.3 Trades Record

```
TradeMsg {
  Header:
    ts_event, publisher_id, instrument_id  (wie MBP-1)

  Body:
    price           int64     // Ausführungspreis (Fixed-Point 1e-9)
    size            uint32    // Ausführungsgröße
    action          byte      // Immer 'T' (Trade)
    side            byte      // Aggressor-Seite (A=Ask-Taker/Buyer, B=Bid-Taker/Seller)
    flags           uint8
    ts_recv         uint64
    sequence        uint64    // Venue-Sequenznummer
}
```

**Datenvolumen:** ~50k–200k Records pro Instrument pro Handelstag

### 2.4 OHLCV Record

```
OhlcvMsg {
  Header:
    ts_event        uint64    // Bar-Open-Timestamp (ns)

  Body:
    open            int64     // Open (Fixed-Point 1e-9)
    high            int64     // High
    low             int64     // Low
    close           int64     // Close
    volume          uint64    // Aggregiertes Volumen
}
```

**Verfügbare Intervalle:** 1s (`ohlcv-1s`), 1m (`ohlcv-1m`), 1h (`ohlcv-1h`), 1d (`ohlcv-1d`)

### 2.5 TBBO — Trade mit Best Bid/Offer

Kompromiss zwischen MBP-1 (jedes BBO-Update) und reinen Trades.
Liefert zu **jedem Trade** den BBO-Kontext unmittelbar vor der Ausführung.
Deutlich weniger Records als MBP-1, aber ausreichend für Spread-Analyse und Slippage-Modellierung.

---

## 3. Datasets und Venues

### 3.1 Für ODIN relevante Datasets

| Dataset-ID | Venue | Beschreibung | Relevanz |
|-----------|-------|-------------|----------|
| **XNAS.ITCH** | NASDAQ | TotalView-ITCH Full Depth | **Primär (IREN, etc.)** |
| XNAS.BASIC | NASDAQ | Basic Quote/Trade Feed | Günstigere Alternative |
| GLBX.MDP3 | CME Globex | ES, NQ Futures | Futures-Trading |
| XNYS.PILLAR | NYSE | NYSE Equities | Falls NYSE-Aktien nötig |

### 3.2 Symbol-Typen (stype_in)

Databento verwendet numerische `instrument_id`s intern. Für API-Anfragen gibt es verschiedene
Eingabe-Symboltypen:

| stype_in | Beschreibung | Beispiel |
|---------|-------------|---------|
| `raw_symbol` | Venue-natives Symbol | `IREN` |
| `parent` | Underlying (für Futures) | `ES.FUT` |
| `continuous` | Endlos-Kontrakt | `ES.c.0` (Front-Month) |

**Für NASDAQ-Aktien:** `stype_in = "raw_symbol"`, Symbol = `"IREN"`

### 3.3 Handelszeiten NASDAQ

- **Pre-Market:** 04:00–09:30 ET
- **Regular Trading Hours (RTH):** 09:30–16:00 ET
- **After-Hours:** 16:00–20:00 ET
- Databento liefert Daten für **alle Sessions** inklusive Pre/After-Market

---

## 4. HTTP REST API — Historische Daten

### 4.1 Hauptendpoint: `timeseries.get_range`

```
GET https://hist.databento.com/v0/timeseries.get_range
```

**Parameter:**

| Parameter | Typ | Pflicht | Beschreibung |
|-----------|-----|---------|-------------|
| `dataset` | String | Ja | z.B. `XNAS.ITCH` |
| `symbols` | String | Ja | Komma-separiert, z.B. `IREN` |
| `schema` | String | Ja | z.B. `mbp-1`, `trades`, `ohlcv-1m` |
| `start` | String | Ja | ISO 8601, z.B. `2026-02-23T00:00:00Z` |
| `end` | String | Ja | ISO 8601, exklusiv |
| `stype_in` | String | Nein | Default: `raw_symbol` |
| `encoding` | String | Nein | `dbn` (default), `csv`, `json` |
| `compression` | String | Nein | `zstd` (default für DBN), `none` |

**Authentifizierung:** HTTP Basic Auth mit API-Key als Username.

**Response:** Streaming Binary (DBN) oder Text (CSV/JSON).

### 4.2 Kostenschätzung

```
GET https://hist.databento.com/v0/metadata.get_cost
```

Gleiche Parameter wie `get_range`. Liefert geschätzte Kosten in USD, **ohne Daten herunterzuladen**.

### 4.3 Batch-Download-API

Für größere Anfragen (mehrere Tage, mehrere Instrumente):

1. `POST /v0/batch.submit_job` — Job anlegen
2. `GET /v0/batch.list_jobs` — Status prüfen
3. `GET /v0/batch.download` — Ergebnis-Dateien herunterladen

Batch-Jobs sind asynchron und eignen sich für Wochen-/Monats-Downloads.

### 4.4 Encoding-Formate

| Format | Komprimierung | Eignung für Java |
|--------|--------------|-----------------|
| **CSV** | Keine (groß) | Einfach zu parsen, aber langsam bei großen Mengen |
| **JSON** | Keine (sehr groß) | Einfach, aber ineffizient |
| **DBN** | Zstd (sehr kompakt) | Binär, braucht eigenen Parser oder Community-Lib |

**Empfehlung für ODIN:** CSV für den initialen Client (einfach zu implementieren in Java).
Später optional DBN für Performance bei größeren Downloads.

---

## 5. Datenvolumen-Abschätzung für IREN (NASDAQ)

### 5.1 Erwartetes Volumen pro Handelstag

| Schema | Records/Tag (geschätzt) | Unkomprimiert | Mit TimescaleDB-Compression |
|--------|------------------------|---------------|---------------------------|
| MBP-1 | 100k–300k | ~30–60 MB | ~3–6 MB |
| Trades | 30k–100k | ~5–15 MB | ~0.5–1.5 MB |
| OHLCV-1m | ~390 (6.5h RTH) + ~660 (Pre/After) | ~100 KB | ~10 KB |
| OHLCV-1s | ~23.400 (6.5h) + ~39.600 | ~4 MB | ~400 KB |

### 5.2 Speicher-Projektion (10 Instrumente, 1 Monat)

- **MBP-1:** 10 × 22 Tage × ~4 MB (komprimiert) = **~880 MB/Monat**
- **Trades:** 10 × 22 × ~1 MB = **~220 MB/Monat**
- **OHLCV-1m:** 10 × 22 × ~10 KB = **~2.2 MB/Monat**

**Fazit:** Selbst bei vollem MBP-1 für 10 Instrumente bleibt der Speicherbedarf mit
TimescaleDB-Compression unter 1.5 GB/Monat — absolut handhabbar.
