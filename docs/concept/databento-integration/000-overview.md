# Databento-Integration — Konzeptübersicht

> Status: Konzept | Erstellt: 2026-02-28

## Motivation

Die bisherige Datenquelle Yahoo Finance hat zwei strukturelle Schwächen:

1. **Kein Volume für Pre-/After-Market:** Yahoo liefert `volume = 0` außerhalb der RTH
2. **Keine Bid/Ask-Daten:** Kein Level 1 (BBO), kein Spread, keine Orderbook-Tiefe

Diese Limitierungen beeinträchtigen die Backtest-Qualität:
- Slippage-Modellierung ohne Spread-Daten ist ungenau
- Volume-basierte Indikatoren (VWAP, Volume Ratio) sind für Extended Hours unbrauchbar
- Limit-Order-Fills können nicht realistisch simuliert werden

**Lösung:** Databento als professioneller Datenprovider für institutionelle Marktdaten.

## Scope

### In Scope
- **Level 1 (BBO):** Best Bid/Ask per MBP-1 oder TBBO Schema
- **Trades:** Tick-by-tick Ausführungen mit Aggressor-Seite
- **OHLCV-Bars:** Professionelle Bars mit echtem Volume (alle Sessions)
- **Selektiver Download:** Einzelne Instrumente, einzelne Tage oder kurze Ranges
- **Primäres Instrument:** IREN (NASDAQ), weitere nach Bedarf
- **Dataset:** XNAS.ITCH (NASDAQ TotalView-ITCH)

### Out of Scope
- Level 2 (MBP-10, Top-10 Orderbook) — vorgesehen, aber nicht in Phase 1
- Level 3 (MBO, Full Depth) — explizit ausgeschlossen (Kosten, Komplexität)
- Massendownloads (ganze Börsen, ganze Jahre)
- Echtzeit-Streaming (Databento Live API) — Zukunft, separates Konzept
- Automatisierte tägliche Downloads — manuell getriggert

## Konzeptdokumente

| # | Dokument | Inhalt |
|---|----------|--------|
| 001 | [API Exploration](001-databento-api-exploration.md) | Databento-API-Analyse: Schemas, Datasets, Endpoints, Preismodell, Datenvolumen |
| 002 | [DB Schema Design](002-database-schema-design.md) | Neue Tabellen (quote_tick, trade_tick), TimescaleDB-Optimierung, Flyway-Migrationen |
| 003 | [API Client Design](003-api-client-design.md) | Java HTTP-Client, CSV-Parser, Persistierung, Download-Workflow |

## Architektur-Entscheidungen (Kurzfassung)

| Entscheidung | Gewählt | Begründung |
|-------------|---------|-----------|
| Tabellen-Layout | Separate Tabellen (quote_tick + trade_tick) | Unterschiedliche Schemas, Query-Patterns, Compression |
| Preis-Datentyp | BIGINT (Fixed-Point 1e-9) | Verlustfrei, bessere Compression |
| Timestamp-Datentyp | TIMESTAMPTZ + BIGINT (ns) | TimescaleDB-Kompatibilität + exakte Ordnung |
| Chunk-Interval | 1 Tag | Backtest-Zugriff ist tageweise |
| Java-Client | Retrofit2 + OkHttp3 (kein offizielles SDK) | Standard-Stack, deklaratives Interface, Streaming, Interceptor-Chain |
| Encoding | CSV (initial) | Einfach parsbar, debuggbar, für ODIN-Volumina ausreichend |
| Symbol-Format | TEXT (wie bestehend) | Konsistenz mit intraday_bar |

## LLM-Sparring-Ergebnis

Alle drei LLMs (Claude, ChatGPT, Gemini) empfehlen übereinstimmend:
- **Option A** (separate Tabellen) statt Unified Table oder Sampling
- **BIGINT Fixed-Point** statt DOUBLE PRECISION für Preise
- **1-Day Chunks** für die TimescaleDB-Partitionierung
- **Compression** mit `segmentby=symbol, orderby=ts_event` und aggressiver Policy (2-3 Tage)
- Speicherbedarf bei 10 Instrumenten: **~1-1.5 GB/Monat** (komprimiert) — trivial

Unterschiede zwischen den LLMs:
- ChatGPT schlägt eine Instrument-Dimensionstabelle (integer IDs) vor — verworfen zugunsten
  von TEXT-Symbolen (Konsistenz mit bestehendem Schema)
- Gemini bevorzugt BIGINT als primäre Hypertable-Zeitdimension — verworfen zugunsten von
  TIMESTAMPTZ (bessere SQL-Ergonomie, native TimescaleDB-Funktionen)
- ChatGPT empfiehlt TBBO als kosteneffiziente Alternative zu vollem MBP-1 — guter Punkt,
  als Optimierungsoption vorgemerkt
