# Anforderungskatalog & Work Packages: Web Application Transformation

**Version:** 1.0
**Stand:** 2026-02-18

---

## 1. Uebersicht

Dieses Dokument definiert den strukturierten Anforderungskatalog und die Work Packages fuer die Transformation von ODIN zu einer vollstaendigen Web-Anwendung mit Backtest-Management, Chart-Daten-API und LLM-Optimierungsloop. Fokus liegt auf dem Backend (REST-APIs). Das Frontend kommt spaeter.

**Scope:** Backend-only REST APIs. Kein Frontend-Code in diesen Work Packages.

**Ausgangslage:**

| Bereich | Status |
|---------|--------|
| Maven-Module | 10 Module (inkl. odin-backtest als separates CLI-Modul mit `WebApplicationType.NONE`) |
| REST Controller | 2 (HealthController, SimulationController) |
| JPA Entities | 9 (TradingRun, Trade, Fill, LlmCall, DecisionLog, EventRecord, ConfigSnapshot, PipelineState, DailyPerformance) |
| Flyway-Migrationen | 14 (V001-V014), inkl. TimescaleDB-Hypertable fuer intraday_bar |
| Backtest | BacktestRunner + YahooDataFetcher + IntradayBarJdbcRepository -- CLI-only |
| KPI Engine | ta4j-basiert: EMA9/21, RSI14, ATR14, Bollinger20, ADX14, ATR Decay, Volume Ratio |
| SSE | Nicht implementiert (Wiki Kap 9 definiert Endpoints) |
| CORS | Nicht konfiguriert |
| Error Handling | Kein globaler `@RestControllerAdvice` |

---

## 2. Anforderungskatalog (39 Requirements)

### REQ-WEB: Web Application Foundation

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-WEB-001 | Globaler Error Handler | `@RestControllerAdvice` mit standardisiertem JSON-Error-Format. Alle Controller-Exceptions werden einheitlich als `{ timestamp, status, error, message, path }` zurueckgegeben. Spezialbehandlung fuer: `ConstraintViolationException` (400), `IllegalStateException` (409), `EntityNotFoundException` (404), unbehandelte (500). | Alle Exceptions resultieren in strukturiertem JSON. Keine Stack-Traces an den Client. HTTP-Statuscodes korrekt. | P0 | -- |
| REQ-WEB-002 | CORS-Konfiguration | `WebMvcConfigurer` mit konfigurierbaren Origins (`odin.web.cors.allowed-origins`). Erlaubt: GET, POST, DELETE. Default: `http://localhost:3000` (Vite Dev Server). Konfigurierbar via Properties. | Frontend auf Port 3000 kann API auf Port 8080 aufrufen. Preflight (OPTIONS) funktioniert. | P0 | -- |
| REQ-WEB-003 | Jackson ObjectMapper-Konfiguration | Globaler `ObjectMapper` Bean: JavaTimeModule registriert, Dates als ISO-8601 Strings (nicht Timestamps), `FAIL_ON_UNKNOWN_PROPERTIES=false`, JSONB-Felder korrekt serialisiert. | Alle Instant/LocalDate/LocalTime-Felder in REST-Responses als ISO-8601-Strings. | P0 | -- |
| REQ-WEB-004 | Paginierungs-Konvention | Standard-Paginierung fuer alle Listen-Endpoints: `?page=0&size=20&sort=tradingDate,desc`. Spring Data `Pageable` + `Page<T>` Response-Wrapper. Maximale Page-Size: 100. | Alle Listen-Endpoints unterstuetzen Paginierung. Default-Werte konfigurierbar. | P0 | -- |
| REQ-WEB-005 | API-Versionierung | URL-Prefix `/api/v1/` fuer alle neuen Endpoints. Bestehende `/api/health` und `/api/instruments` migrieren. | Alle Endpoints unter `/api/v1/`. Alte Endpoints als Redirect oder Alias. | P1 | REQ-WEB-001 |
| REQ-WEB-006 | Request-Validation | Bean Validation auf allen Request-DTOs (`@Valid`, `@NotNull`, `@NotBlank`, `@Min`, `@Max`). Validation-Fehler resultieren in 400 mit Feld-Details. | Ungueltige Requests liefern 400 mit `{ field, message, rejectedValue }` pro Verletzung. | P0 | REQ-WEB-001 |
| REQ-WEB-007 | Async-Backtest-Execution | Backtests laufen asynchron (eigener Thread/Executor). REST-Endpoint gibt sofort eine Backtest-ID zurueck. Status-Polling via GET-Endpoint. Grund: Backtests dauern Sekunden bis Minuten. | POST gibt 202 Accepted + backtestId. GET liefert Status (RUNNING, COMPLETED, FAILED) + Ergebnis. | P0 | REQ-BT-001 |

### REQ-BT: Backtest Management

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-BT-001 | Backtest via REST starten | `POST /api/v1/backtests` mit Body: `{ instruments: ["AAPL","MSFT"], startDate: "2026-01-01", endDate: "2026-01-31", barInterval: "FIVE_MINUTES", autoDownload: true }`. BacktestRunner-Logik (aktuell in odin-backtest CLI) wird in odin-app integriert. Response: 202 Accepted + `{ backtestId }`. | Backtest startet asynchron. CLI-Funktionalitaet erhalten. odin-backtest Modul wird zu Library (kein eigener Spring Boot Main). | P0 | REQ-WEB-001, REQ-WEB-007 |
| REQ-BT-002 | Backtest-Status abfragen | `GET /api/v1/backtests/{backtestId}`. Response enthaelt: Status (QUEUED, RUNNING, COMPLETED, FAILED), Progress (aktueller Tag / Gesamttage), Startzeit, Optional: Report bei COMPLETED. | Status-Polling funktioniert. Progress wird bei jedem Tag-Abschluss aktualisiert. | P0 | REQ-BT-001 |
| REQ-BT-003 | Backtest-Ergebnisse abfragen | `GET /api/v1/backtests/{backtestId}/report`. Response: Vollstaendiger `BacktestReport` (batchId, dailyResults, summary). Separater Endpoint fuer Daily-Details: `GET /api/v1/backtests/{backtestId}/days?page=0&size=10`. | Report mit allen Summary-Feldern (totalPnl, sharpe, maxDrawdown, winRate). Tages-Details paginiert. | P0 | REQ-BT-002 |
| REQ-BT-004 | Backtest-Historie | `GET /api/v1/backtests?status=COMPLETED&page=0&size=20`. Listet alle bisherigen Backtest-Laeufe mit Kurzinformationen (backtestId, instruments, dateRange, status, summary.totalPnl). Filterbar nach Status, Instrument, Datumsbereich. | Liste paginiert, filterbar. Sortierbar nach Datum oder PnL. | P1 | REQ-BT-003 |
| REQ-BT-005 | Backtest-Persistierung | BacktestReport wird in DB persistiert (neue Entity `BacktestRunEntity`). Felder: backtestId (PK), instruments, startDate, endDate, barInterval, status, summaryJson (JSONB), startedAt, completedAt. Referenz auf TradingRun-Entities ueber batchId. | Backtest-Ergebnisse ueberleben Server-Restart. Alte Backtests abrufbar. | P0 | REQ-BT-001 |
| REQ-BT-006 | Daten-Download-Status | Vor dem Backtest: Pruefung ob Daten vorhanden. `GET /api/v1/data/availability?instruments=AAPL,MSFT&startDate=...&endDate=...&barInterval=FIVE_MINUTES`. Response: `{ available: {AAPL: 20, MSFT: 18}, missing: {AAPL: 2, MSFT: 4} }`. Download-Fortschritt ueber Backtest-Status. | Benutzer sieht vor dem Start, ob Daten fehlen. autoDownload-Flag steuert automatischen Download. | P1 | REQ-BT-001 |
| REQ-BT-007 | Backtest-Abbruch | `POST /api/v1/backtests/{backtestId}/cancel`. Setzt KillSwitch fuer den laufenden Backtest. Status wechselt zu CANCELLED. Bereits fertige Tages-Ergebnisse bleiben erhalten. | Laufender Backtest kann abgebrochen werden. Partielle Ergebnisse verfuegbar. | P2 | REQ-BT-002 |

### REQ-CHART: Chart Data API

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-CHART-001 | Intraday-Bars Endpoint | `GET /api/v1/charts/{symbol}/bars?tradingDate=2026-01-15&interval=FIVE_MINUTES`. Response: Sortierte Liste von `{ openTime, closeTime, open, high, low, close, volume }`. Datenquelle: `odin.intraday_bar` Tabelle (bestehend). | OHLCV-Bars fuer ein Symbol/Datum/Interval abrufbar. Sortiert nach openTime. | P0 | -- |
| REQ-CHART-002 | KPI-Overlays Endpoint | `GET /api/v1/charts/{symbol}/indicators?tradingDate=2026-01-15&runId=...&indicators=ema9,ema21,rsi14,atr14,bollinger`. Response: Zeitreihe von `{ barTime, ema9, ema21, rsi14, atr14, bollingerUpper, bollingerMiddle, bollingerLower, vwap }`. Datenquelle: Entweder persisted IndicatorResults ODER on-the-fly Berechnung aus Bars. | Indikatoren als Zeitreihe abrufbar. NaN-Werte waehrend Warmup korrekt markiert (null in JSON). | P1 | REQ-CHART-001, REQ-KPI-001 |
| REQ-CHART-003 | Entry/Exit Marker | `GET /api/v1/charts/{symbol}/trades?runId=...` ODER `GET /api/v1/runs/{runId}/trades`. Response: Liste von `{ tradeId, instrumentId, entryTime, entryPrice, exitTime, exitPrice, quantity, realizedPnl, exitReason, regime, tranchenVariant }`. Fuer Chart-Marker-Darstellung. | Trades mit Timestamps und Preisen abrufbar. Filtern nach runId. | P0 | -- |
| REQ-CHART-004 | Multi-Timeframe Bars | Bars in verschiedenen Timeframes: 1m, 5m, 15m. 1m und 5m aus DB. 15m-Bars on-the-fly aus 5m aggregiert (3 x 5m = 15m). Alternative: Backend speichert alle Timeframes. | Mindestens 1m und 5m direkt. 15m als berechneter Timeframe. | P2 | REQ-CHART-001 |
| REQ-CHART-005 | DecisionLog als Chart-Layer | `GET /api/v1/runs/{runId}/decisions?page=0&size=100`. Response mit `barCloseTime`, `intentType`, `finalDecision`, `rejectReason`, `quantScore` -- damit das Frontend Decision-Punkte auf dem Chart darstellen kann. Existierende DecisionLogEntity nutzen. | DecisionLog paginiert abrufbar. Filtern nach intentType moeglich. | P1 | -- |

### REQ-KPI: Extended KPI Engine

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-KPI-001 | Indikator-Persistierung | IndicatorResult-Werte pro Bar in DB speichern. Neue Entity `IndicatorSnapshotEntity` oder Erweiterung der bestehenden intraday_bar Tabelle (zusaetzliche Spalten). Felder: runId, instrumentId, barTime, ema9, ema21, rsi14, atr14, bollingerUpper/Mid/Lower, adx14, atrDecay, volumeRatio, vwap. | Indikatoren nach Backtest/Live-Run aus DB abrufbar. Kein erneutes Berechnen noetig. | P1 | -- |
| REQ-KPI-002 | EMA50 und EMA100 | KpiEngine um EMA(50) und EMA(100) auf 5m-Bars erweitern. Konfigurierbar ueber `BrainProperties.KpiProperties`. Warmup-Pruefung anpassen. IndicatorResult-Record erweitern. | EMA50 und EMA100 in IndicatorResult vorhanden. Konfigurierbar. Warmup beruecksichtigt. | P1 | -- |
| REQ-KPI-003 | Anchored VWAP (AVWAP) | AVWAP berechnen: VWAP ab einem definierten Ankerpunkt (z.B. Tages-Open, letzter Swing-Low). Quelle: odin-data liefert kumulierten VWAP. AVWAP ist ein Reset-faehiger VWAP ab Ankerzeitpunkt. | AVWAP als zusaetzliches Feld in IndicatorResult. Ankerpunkt konfigurierbar (mindestens: Tages-Open). | P2 | -- |
| REQ-KPI-004 | On-the-fly KPI-Berechnung fuer historische Daten | Service der aus gespeicherten Bars (intraday_bar Tabelle) alle KPIs on-the-fly berechnet. Fuer Chart-Overlay wenn keine persisted IndicatorResults vorhanden (z.B. bei Yahoo-Only-Daten ohne Run). | Chart-Indikatoren auch ohne Run verfuegbar (pure Bar-Daten). | P1 | REQ-CHART-001, REQ-CHART-002 |

### REQ-OPT: LLM Optimization Loop

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-OPT-001 | Optimization-Request via REST | `POST /api/v1/backtests/{backtestId}/optimize`. Startet einen LLM-Analyse-und-Optimierungs-Zyklus: Claude Agent SDK erhaelt den BacktestReport + aktuelle Config, analysiert Schwaechen, schlaegt Parameter-Aenderungen vor. Response: 202 Accepted + `{ optimizationId }`. | Optimierung startet asynchron. Status abfragbar. | P1 | REQ-BT-003 |
| REQ-OPT-002 | Optimierungs-Ergebnis | `GET /api/v1/optimizations/{optimizationId}`. Response: `{ status, suggestedParams: {...}, reasoning: "...", deltaFromBaseline: {...}, optimizedBacktestId }`. Das LLM liefert: welche Parameter geaendert, warum, erwarteter Effekt. | Optimierungsvorschlag mit Begruendung abrufbar. | P1 | REQ-OPT-001 |
| REQ-OPT-003 | Auto-Rerun nach Optimierung | Nach erfolgreicher Optimierung: automatischer Backtest-Rerun mit den vorgeschlagenen Parametern. Der neue Backtest referenziert den Optimierungsvorschlag (parentOptimizationId). Ergebnis vergleichbar mit Baseline. | Neuer Backtest startet automatisch. Vergleich Baseline vs. Optimiert moeglich. | P1 | REQ-OPT-002, REQ-BT-001 |
| REQ-OPT-004 | Parameterset-Verwaltung | Entity `ParameterSetEntity`: id, name, description, createdAt, sourceType (MANUAL, LLM_OPTIMIZED), sourceOptimizationId, params (JSONB), performance (JSONB -- letzte Summary). CRUD: `GET/POST/PUT/DELETE /api/v1/parameter-sets`. Backtest kann mit einem ParameterSet gestartet werden. | ParameterSets persistent. Auswahlbar beim Backtest-Start. Vergleich zwischen Sets. | P1 | REQ-OPT-002 |
| REQ-OPT-005 | Claude Agent SDK Integration | LLM-Client fuer Optimierung: Sendet Report + Config an Claude, erhaelt strukturierten JSON-Response mit Parameteraenderungen. Prompt-Template versioniert. Rate-Limiting und Token-Budget wie bestehende LLM-Integration. | Claude-Aufruf funktioniert. Strukturierter Response geparst. Fehlerbehandlung (Timeout, Invalid Response). | P1 | -- |
| REQ-OPT-006 | Iterative Optimierung | Mehrfach-Optimierung: Ergebnis von Run N wird Input fuer Optimierung N+1. Kette von optimizationId -> backtestId -> optimizationId -> backtestId. Max. Iterationen konfigurierbar (Default: 5). | Ketten-Optimierung funktioniert. Konvergenz erkennbar (PnL stabilisiert oder verschlechtert sich). | P2 | REQ-OPT-003 |

### REQ-LIVE: Live Monitoring

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-LIVE-001 | SSE Instrument Stream | `GET /api/v1/stream/instruments/{instrumentId}` (SseEmitter). Event-Typen laut Wiki Kap 9: snapshot, state-change, trade-update, order-update, llm-update, alert, pnl. Reconnect mit `Last-Event-ID` + Replay aus In-Memory-Ringbuffer (60s). | SSE-Stream liefert Events. Reconnect mit Replay funktioniert. Events enthalten `eventId`. | P1 | REQ-WEB-001 |
| REQ-LIVE-002 | SSE Global Stream | `GET /api/v1/stream/global`. Event-Typen: risk-update (AccountRiskState), aggregierte P&L, Kill-Switch-State, Alerts. Update alle 5s (konfigurierbar). | Global-Stream liefert Account-weite Daten. Kill-Switch-State sichtbar. | P1 | REQ-LIVE-001 |
| REQ-LIVE-003 | Control Endpoints | `POST /api/v1/controls/kill`, `POST /api/v1/controls/pause/{instrumentId}`, `POST /api/v1/controls/resume/{instrumentId}`. Response-Kontrakt laut Wiki Kap 9 (ACCEPTED/REJECTED/NOOP mit reasonCode). HTTP 409 bei unzulaessigem State. | Controls funktionieren. FSM-Restriktionen eingehalten. | P1 | REQ-LIVE-001 |
| REQ-LIVE-004 | Live-Bars in DB | Waehrend Live-Trading: Intraday-Bars (1m) aus dem BarBuilder (odin-data) in `odin.intraday_bar` persistieren. Source: "LIVE_IB". Damit Chart-Daten auch fuer Live-Runs verfuegbar. | Live-Bars in DB. Chart-API liefert Live-Daten. | P2 | REQ-CHART-001 |

### REQ-DATA: Data Management

| ID | Titel | Beschreibung | Akzeptanzkriterien | Prio | Abhaengigkeiten |
|----|-------|-------------|-------------------|------|----------------|
| REQ-DATA-001 | TradingRun-Liste (erweitert) | `GET /api/v1/runs?mode=SIMULATION&tradingDate=2026-01-15&instrumentId=AAPL&batchId=...&status=COMPLETED&page=0&size=20`. Filter laut Wiki Kap 9. | TradingRuns paginiert, filterbar nach mode, tradingDate, instrumentId, batchId, status. | P0 | REQ-WEB-004 |
| REQ-DATA-002 | TradingRun-Details | `GET /api/v1/runs/{runId}`. Response: Vollstaendige TradingRunEntity als DTO + Trade-Count + Summary-Felder. | Run-Details mit allen Feldern abrufbar. | P0 | -- |
| REQ-DATA-003 | Trades eines Runs | `GET /api/v1/runs/{runId}/trades`. Response: Liste aller Trades des Runs mit allen Feldern (entryTime, exitTime, entryPrice, exitPrice, realizedPnl, exitReason, regime, tranchenVariant). | Alle Trades eines Runs abrufbar. | P0 | -- |
| REQ-DATA-004 | LLM-History eines Runs | `GET /api/v1/runs/{runId}/llm-history?page=0&size=20`. Response: LlmCall-Metadaten (OHNE inputPayload/outputPayload -- zu gross). Felder: callId, provider, model, regime, regimeConfidence, requestMarketTime, latencyMs, validationResult. | LLM-History paginiert. Ohne Payload-Felder. | P1 | -- |
| REQ-DATA-005 | Data-Download Endpoint | `POST /api/v1/data/download` mit Body: `{ instruments: ["AAPL"], startDate, endDate, barInterval, exchange }`. Nutzt bestehenden `DataDownloadService` + `YahooDataFetcher`. Asynchron (202 Accepted). | Daten-Download ueber API ausloesbar. Progress trackbar. | P1 | REQ-BT-006 |
| REQ-DATA-006 | Symbol-Suche und Metadata | `GET /api/v1/symbols/search?q=AAPL`. Gibt Basis-Informationen zurueck (Symbol, Name, Exchange). V1: Statische Liste oder Yahoo-API-Lookup. | Symbolsuche funktioniert fuer gaengige US-Ticker. | P2 | -- |

---

## 3. Work Packages (WP-01 bis WP-10)

### WP-01: Web Application Foundation

**Scope:** REQ-WEB-001, REQ-WEB-002, REQ-WEB-003, REQ-WEB-004, REQ-WEB-006
**Betroffene Module:** odin-app
**Komplexitaet:** S
**Abhaengigkeiten:** Keine. Grundlage fuer alle weiteren WPs.

**Inhalt:**

1. `GlobalExceptionHandler` (`@RestControllerAdvice`) in `de.its.odin.app.controller`
    - Standardisiertes Error-Response-Record: `ApiError(Instant timestamp, int status, String error, String message, String path)`
    - Handler fuer `ConstraintViolationException`, `MethodArgumentNotValidException`, `EntityNotFoundException`, `IllegalStateException`, generischer Fallback
2. `WebConfig` (`WebMvcConfigurer`) in `de.its.odin.app.config`
    - CORS-Konfiguration mit `CorsProperties` Record (`odin.web.cors.allowed-origins`, Default: `http://localhost:3000`)
3. `JacksonConfig` (`@Configuration`) in `de.its.odin.app.config`
    - `ObjectMapper` Bean: JavaTimeModule, ISO-8601 Dates, `FAIL_ON_UNKNOWN_PROPERTIES=false`
4. Paginierungs-Defaults konfigurieren (`spring.data.web.pageable.default-page-size=20`, `max-page-size=100`)
5. Request-Validation: `spring-boot-starter-validation` Dependency (falls noch nicht vorhanden)

**Lieferergebnis:** Alle REST-Endpoints haben einheitliches Error-Handling, CORS, JSON-Serialisierung, Paginierung und Validation.

---

### WP-02: Backtest Integration in odin-app

**Scope:** REQ-BT-001, REQ-BT-002, REQ-BT-005, REQ-WEB-007
**Betroffene Module:** odin-app, odin-backtest, odin-persistence
**Komplexitaet:** L
**Abhaengigkeiten:** WP-01

**Inhalt:**

1. **odin-backtest als Library umbauen:**
    - `BacktestApplication` als `@SpringBootApplication` entfernen (oder `@Profile("cli")` aktivieren)
    - spring-boot-maven-plugin in odin-backtest entfernen
    - Bestehende Klassen (`BacktestRunner`, `DataDownloadService`, `YahooDataFetcher`, `IntradayBarJdbcRepository`, `TradingCalendar`, `BarInterval`, `BacktestReport`) bleiben unveraendert
    - odin-app erhaelt `odin-backtest` als Maven-Dependency

2. **Neue Entity: `BacktestRunEntity`** (in odin-backtest -- Domain-Owner)
    - Flyway-Migration V015
    - Felder: `backtestId` (UUID, PK), `instruments` (TEXT[]), `startDate`, `endDate`, `barInterval`, `status` (QUEUED, RUNNING, COMPLETED, FAILED, CANCELLED), `summaryJson` (JSONB), `startedAt`, `completedAt`, `errorMessage`, `batchId` (FK zu TradingRun.batchId)
    - Repository: `BacktestRunRepository`

3. **Async-Execution-Service:** `BacktestExecutionService` in odin-app
    - `@Async` oder manueller `ExecutorService` (Thread-Pool: `odin.web.backtest.thread-pool-size`, Default: 2)
    - Orchestriert: BacktestRun Entity erstellen (QUEUED) -> Status RUNNING -> BacktestRunner.run() -> Status COMPLETED + summaryJson -> Error: Status FAILED + errorMessage
    - Progress-Tracking: Callback-Mechanismus (BacktestRunner muss Progress melden koennen -- aktueller Tag / Gesamttage)

4. **REST Controller:** `BacktestController` in `de.its.odin.app.controller`
    - `POST /api/v1/backtests` -> 202 Accepted + `{ backtestId }`
    - `GET /api/v1/backtests/{backtestId}` -> Status + Progress + optionaler Report
    - `GET /api/v1/backtests/{backtestId}/report` -> Vollstaendiger BacktestReport
    - Request-DTO mit Validation: `BacktestRequest(List<String> instruments, LocalDate startDate, LocalDate endDate, BarInterval barInterval, boolean autoDownload)`

5. **BacktestRunner erweitern:** Progress-Callback (z.B. `Consumer<BacktestProgress>` im Konstruktor oder als Parameter in `run()`)

**Architekturentscheidung:** `BacktestRunEntity` gehoert zu odin-backtest (dort liegt die Domaene). odin-backtest erhaelt dann `spring-boot-starter-data-jpa` als Dependency fuer die Entity/Repository. Alternative: Neue Entity in odin-execution (weil TradingRun dort ist). Empfehlung: **odin-backtest** -- das ist die Domaene.

**Lieferergebnis:** Backtests ueber REST startbar, Status abfragbar, Ergebnisse persistent.

---

### WP-03: TradingRun & Trade REST API

**Scope:** REQ-DATA-001, REQ-DATA-002, REQ-DATA-003, REQ-CHART-003
**Betroffene Module:** odin-app, odin-execution
**Komplexitaet:** M
**Abhaengigkeiten:** WP-01

**Inhalt:**

1. **Response-DTOs** in odin-app:
    - `TradingRunDto` Record (Subset von TradingRunEntity fuer API)
    - `TradeDto` Record (Subset von TradeEntity)
    - `FillDto` Record (optional: bei Bedarf)
    - DTO-Mapper (manuell, kein MapStruct -- Lean-Prinzip)

2. **Repository-Erweiterungen** (TradingRunRepository):
    - Neue Methode: `Page<TradingRunEntity> findByModeAndTradingDateAndInstrumentIdAndStatus(...)` mit `Pageable`
    - Oder `@Query` fuer flexible Filterung mit optionalen Parametern (Specification Pattern oder `@Query` mit Nullcheck)
    - `findByBatchId` existiert bereits

3. **REST Controller:** `TradingRunController` in `de.its.odin.app.controller`
    - `GET /api/v1/runs` -- paginiert, filterbar (mode, tradingDate, instrumentId, batchId, status)
    - `GET /api/v1/runs/current` -- aktuelle Live-Runs (heute, LIVE, ACTIVE)
    - `GET /api/v1/runs/{runId}` -- Detail
    - `GET /api/v1/runs/{runId}/trades` -- Trades des Runs

**Lieferergebnis:** TradingRun- und Trade-Daten per REST abrufbar. Chart-Marker-Daten (Entry/Exit) verfuegbar.

---

### WP-04: Chart Data API (Bars)

**Scope:** REQ-CHART-001, REQ-CHART-004
**Betroffene Module:** odin-app, odin-backtest (IntradayBarJdbcRepository)
**Komplexitaet:** M
**Abhaengigkeiten:** WP-01

**Inhalt:**

1. **IntradayBarJdbcRepository erweitern** (oder neuen Service in odin-app):
    - Methode: `loadBars(String symbol, LocalDate tradingDate, int barIntervalSec)` -- liefert sortierte Bar-Liste
    - Methode: `loadBarRange(String symbol, LocalDate startDate, LocalDate endDate, int barIntervalSec)` -- fuer Multi-Day-Charts
    - Bestehende `loadDay()` liefert bereits `Map<String, List<Bar>>` -- kann als Basis dienen

2. **Bar-Aggregation-Service** (fuer 15m aus 5m):
    - `BarAggregationService` der 5m-Bars zu 15m gruppiert (3 Bars -> 1: open=first.open, close=last.close, high=max, low=min, volume=sum)
    - Wiederverwendung des bestehenden `BarAggregator` in odin-data wenn passend

3. **REST Controller:** `ChartController` in `de.its.odin.app.controller`
    - `GET /api/v1/charts/{symbol}/bars?tradingDate=...&interval=FIVE_MINUTES` -- einzelner Tag
    - `GET /api/v1/charts/{symbol}/bars?startDate=...&endDate=...&interval=FIVE_MINUTES` -- Range
    - Response: `List<BarDto>` (Record: openTime, closeTime, open, high, low, close, volume)

4. **BarDto** als API-Response-Record (kann direkt `Bar` aus odin-api sein, da bereits Record)

**Lieferergebnis:** OHLCV-Bars ueber REST abrufbar, Multi-Timeframe (1m, 5m, 15m).

---

### WP-05: Backtest-Historie und Vergleich

**Scope:** REQ-BT-003 (Detail), REQ-BT-004, REQ-BT-006, REQ-BT-007
**Betroffene Module:** odin-app, odin-backtest
**Komplexitaet:** M
**Abhaengigkeiten:** WP-02

**Inhalt:**

1. **BacktestController erweitern:**
    - `GET /api/v1/backtests?status=COMPLETED&page=0&size=20` -- Historie
    - `GET /api/v1/backtests/{backtestId}/days?page=0&size=10` -- Tages-Details paginiert
    - `POST /api/v1/backtests/{backtestId}/cancel` -- Abbruch

2. **Daten-Verfuegbarkeits-Endpoint:**
    - `GET /api/v1/data/availability?instruments=...&startDate=...&endDate=...&barInterval=...`
    - Nutzt bestehenden `IntradayBarJdbcRepository.findMissingDays()`

3. **Data-Download-Endpoint:**
    - `POST /api/v1/data/download` -- asynchron
    - Nutzt bestehenden `DataDownloadService.ensureDataAvailable()`
    - Status-Tracking analog zu Backtest (eigener Status oder in Backtest integriert)

4. **Cancel-Mechanismus:** BacktestRunner braucht ein `AtomicBoolean cancelled` Flag. Der for-loop ueber tradingDays prueft dieses Flag.

**Lieferergebnis:** Volle Backtest-Verwaltung inkl. Historie, Daten-Check, Download, Abbruch.

---

### WP-06: KPI Erweiterung und Persistierung

**Scope:** REQ-KPI-001, REQ-KPI-002, REQ-KPI-004, REQ-CHART-002
**Betroffene Module:** odin-brain, odin-backtest (oder odin-data fuer Persistierung), odin-persistence, odin-app
**Komplexitaet:** L
**Abhaengigkeiten:** WP-01, WP-04

**Inhalt:**

1. **KpiEngine erweitern (odin-brain):**
    - EMA50 und EMA100 auf 5m-Bars hinzufuegen
    - `BrainProperties.KpiProperties` um `emaLongPeriod50` (default 50) und `emaLongPeriod100` (default 100) erweitern
    - `IndicatorResult` Record erweitern um `ema50_5m` und `ema100_5m`
    - Warmup-Pruefung anpassen

2. **Indikator-Persistierung:**
    - Neue Entity `IndicatorSnapshotEntity` in odin-brain
    - Flyway V016: Tabelle `odin.indicator_snapshot` (runId, instrumentId, barTime, ema9, ema21, ema50, ema100, rsi14, atr14, bollingerUpper/Mid/Lower, adx14, atrDecay, volumeRatio, vwap)
    - Index: (runId, instrumentId, barTime)
    - KpiEngine bekommt optionalen Persistierungs-Callback (Repository-Referenz als Konstruktor-Parameter)

3. **On-the-fly KPI-Berechnung:**
    - `IndicatorCalculationService` in odin-brain
    - Nimmt `List<Bar>` als Input, erstellt temporaeres KpiEngine, fuettert Bars sequentiell, sammelt IndicatorResults
    - Fuer Chart-Overlays wenn keine gespeicherten Indikatoren vorhanden

4. **REST Controller Erweiterung:** `ChartController`
    - `GET /api/v1/charts/{symbol}/indicators?tradingDate=...&runId=...` -- persisted Indicators
    - `GET /api/v1/charts/{symbol}/indicators?tradingDate=...&interval=FIVE_MINUTES` -- on-the-fly (ohne runId)

**Lieferergebnis:** Erweiterte KPIs (EMA50/100), persistierte Indikatoren, Chart-Overlays.

---

### WP-07: DecisionLog und LLM-History API

**Scope:** REQ-CHART-005, REQ-DATA-004
**Betroffene Module:** odin-app, odin-brain
**Komplexitaet:** S
**Abhaengigkeiten:** WP-01

**Inhalt:**

1. **Repository-Erweiterungen:**
    - `DecisionLogRepository`: `Page<DecisionLogEntity> findByRunId(UUID runId, Pageable pageable)` und `findByRunIdAndIntentType(...)`
    - `LlmCallRepository`: `Page<LlmCallEntity> findByRunId(UUID runId, Pageable pageable)` -- Projection OHNE inputPayload/outputPayload

2. **DTOs:**
    - `DecisionLogDto` Record (alle Felder)
    - `LlmCallSummaryDto` Record (ohne Payloads: callId, provider, model, regime, regimeConfidence, requestMarketTime, latencyMs, validationResult)

3. **Controller-Erweiterung** (in `TradingRunController` oder eigener Controller):
    - `GET /api/v1/runs/{runId}/decisions?page=0&size=100&intentType=ENTRY`
    - `GET /api/v1/runs/{runId}/llm-history?page=0&size=20`

**Lieferergebnis:** DecisionLog und LLM-History per REST abrufbar.

---

### WP-08: SSE Live Monitoring

**Scope:** REQ-LIVE-001, REQ-LIVE-002, REQ-LIVE-003
**Betroffene Module:** odin-app, odin-core
**Komplexitaet:** XL
**Abhaengigkeiten:** WP-01, WP-03

**Inhalt:**

1. **SSE-Infrastruktur:**
    - `SseEmitterManager` Service: verwaltet aktive SseEmitter-Instanzen pro instrumentId und global
    - In-Memory-Ringbuffer pro Stream (60s Kapazitaet, konfigurierbar)
    - Event-ID-Sequenzierung (monoton steigend pro Stream)
    - Reconnect-Handling: `Last-Event-ID` Header -> Replay aus Ringbuffer

2. **SSE Event-Typen** (laut Wiki Kap 9):
    - Event-Records als Java-Records: `SnapshotEvent`, `StateChangeEvent`, `TradeUpdateEvent`, `OrderUpdateEvent`, `LlmUpdateEvent`, `AlertEvent`, `PnlEvent`, `RiskUpdateEvent`
    - Discriminated Union via `event:` SSE-Feld (Spring SseEmitter.event().name())

3. **Pipeline-Integration:**
    - odin-core muss Events an den SseEmitterManager liefern
    - Neuer Port/Listener in odin-api: `MonitoringEventPublisher` Interface
    - odin-core implementiert: publiziert bei State-Changes, Trade-Updates, etc.
    - odin-app registriert SseEmitterManager als Listener

4. **SSE Controller:** `StreamController`
    - `GET /api/v1/stream/instruments/{instrumentId}` -> `SseEmitter`
    - `GET /api/v1/stream/global` -> `SseEmitter`
    - Timeout-Konfiguration, Heartbeat (keep-alive alle 15s)

5. **Control Controller:** `ControlController`
    - `POST /api/v1/controls/kill` -> delegiert an KillSwitchService
    - `POST /api/v1/controls/pause/{instrumentId}` -> delegiert an PipelineStateMachine
    - `POST /api/v1/controls/resume/{instrumentId}` -> delegiert an PipelineStateMachine
    - Response-Kontrakt: `{ status: "ACCEPTED"|"REJECTED"|"NOOP", reasonCode, marketClockTimestamp }`

**Lieferergebnis:** Vollstaendiges Live-Monitoring mit SSE-Streams und Control-Endpoints.

---

### WP-09: LLM Optimization Loop

**Scope:** REQ-OPT-001, REQ-OPT-002, REQ-OPT-003, REQ-OPT-004, REQ-OPT-005
**Betroffene Module:** odin-app, odin-brain, odin-backtest, odin-persistence
**Komplexitaet:** XL
**Abhaengigkeiten:** WP-02, WP-05

**Inhalt:**

1. **Neue Entities:**
    - `OptimizationRunEntity` (odin-brain): optimizationId, backtestId (Source), status, suggestedParamsJson (JSONB), reasoning, parentOptimizationId, resultBacktestId, createdAt, completedAt
    - `ParameterSetEntity` (odin-brain oder odin-core): id, name, description, sourceType (MANUAL, LLM_OPTIMIZED), sourceOptimizationId, paramsJson (JSONB), performanceJson (JSONB), createdAt
    - Flyway V017 + V018

2. **Claude Agent SDK Client:**
    - `OptimizationAnalystClient` in odin-brain
    - Prompt-Template: Nimmt BacktestReport + aktuelle BrainProperties/ExecutionProperties, analysiert Schwachstellen, schlaegt Parameteraenderungen vor
    - Strukturierter Response: `{ parameters: {...}, reasoning: "...", expectedImpact: "..." }`
    - Token-Budget und Rate-Limiting

3. **Optimization-Service:**
    - `OptimizationService` in odin-app (oder odin-brain)
    - Workflow: Load Report -> Prepare Prompt -> Call Claude -> Parse Response -> Save OptimizationRun -> Optional: Trigger Rerun

4. **Parameter-Override-Mechanismus:**
    - BacktestRunner muss mit Override-Parametern laufen koennen (nicht nur aus application.properties)
    - ParameterSet -> ConfigProperties Override -> BacktestRunner

5. **REST Controller:**
    - `OptimizationController`:
        - `POST /api/v1/backtests/{backtestId}/optimize` -> 202
        - `GET /api/v1/optimizations/{optimizationId}` -> Status + Ergebnis
    - `ParameterSetController`:
        - CRUD: `GET/POST/PUT/DELETE /api/v1/parameter-sets`
        - `POST /api/v1/backtests` erweitern um optionales `parameterSetId`

**Lieferergebnis:** LLM-gestuetzte Optimierung mit Parameter-Verwaltung und Auto-Rerun.

---

### WP-10: API-Versionierung und Migration

**Scope:** REQ-WEB-005
**Betroffene Module:** odin-app
**Komplexitaet:** S
**Abhaengigkeiten:** WP-01, WP-03, WP-04 (sollte nach den ersten Controllern kommen)

**Inhalt:**

1. Alle neuen Endpoints unter `/api/v1/`
2. Bestehende Endpoints (`/api/health`, `/api/instruments`, `/api/runs/simulation`) auf `/api/v1/` migrieren
3. Alte Pfade als Redirect oder Alias (302 oder `@RequestMapping` Alias)
4. `RequestMappingHandlerMapping` konfigurieren oder Base-Klasse mit Prefix

**Lieferergebnis:** Konsistente API-Versionierung.

---

## 4. Implementierungsreihenfolge und Parallelisierungsmatrix

### Phasen-Plan

```
Phase 1: Foundation (Woche 1)
+-- WP-01: Web Foundation [S]          <- ERSTE PRIORITAET, blockiert alles

Phase 2: Core Data APIs (Woche 2-3)
+-- WP-03: TradingRun & Trade API [M]  <- parallel-faehig
+-- WP-04: Chart Data API (Bars) [M]   <- parallel-faehig
+-- WP-07: DecisionLog & LLM-History [S] <- parallel-faehig

Phase 3: Backtest Web Integration (Woche 3-5)
+-- WP-02: Backtest in odin-app [L]    <- blockiert WP-05, WP-09
|   +-- WP-05: Backtest-Historie [M]   <- nach WP-02

Phase 4: KPI & Charts (Woche 5-6)
+-- WP-06: KPI Erweiterung [L]         <- nach WP-04
+-- WP-10: API-Versionierung [S]       <- flexibel, nach Phase 2-3

Phase 5: Live & Optimization (Woche 7-10)
+-- WP-08: SSE Live Monitoring [XL]    <- nach WP-03
+-- WP-09: LLM Optimization [XL]       <- nach WP-02 + WP-05
```

### Parallelisierungsmatrix

| WP | Kann parallel mit | Muss nach |
|----|-------------------|-----------|
| WP-01 | -- | -- (Start) |
| WP-02 | WP-03, WP-04, WP-07 | WP-01 |
| WP-03 | WP-02, WP-04, WP-07 | WP-01 |
| WP-04 | WP-02, WP-03, WP-07 | WP-01 |
| WP-05 | WP-06, WP-07 | WP-02 |
| WP-06 | WP-05 | WP-01, WP-04 |
| WP-07 | WP-02, WP-03, WP-04 | WP-01 |
| WP-08 | WP-09 | WP-01, WP-03 |
| WP-09 | WP-08 | WP-02, WP-05 |
| WP-10 | WP-05, WP-06 | WP-01, WP-03 |

### Kritischer Pfad

```
WP-01 -> WP-02 -> WP-05 -> WP-09
  \-> WP-03 -> WP-08
  \-> WP-04 -> WP-06
```

### Empfohlene Implementierungssequenz (seriell, 1 Entwickler)

1. WP-01 -- Web Foundation (1-2 Tage)
2. WP-03 -- TradingRun & Trade API (2-3 Tage)
3. WP-04 -- Chart Data API (2-3 Tage)
4. WP-07 -- DecisionLog & LLM-History (1-2 Tage)
5. WP-02 -- Backtest Integration (4-5 Tage)
6. WP-05 -- Backtest-Historie (2-3 Tage)
7. WP-06 -- KPI Erweiterung (3-4 Tage)
8. WP-10 -- API-Versionierung (0.5-1 Tag)
9. WP-08 -- SSE Live Monitoring (5-7 Tage)
10. WP-09 -- LLM Optimization (5-7 Tage)

---

## 5. Neue DB-Migrationen (Flyway V015-V018)

| Migration | Tabelle | WP |
|-----------|---------|----|
| V015 | `odin.backtest_run` | WP-02 |
| V016 | `odin.indicator_snapshot` | WP-06 |
| V017 | `odin.optimization_run` | WP-09 |
| V018 | `odin.parameter_set` | WP-09 |

### V015 -- `odin.backtest_run`

Entity: `BacktestRunEntity`

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `backtest_id` | UUID | PK |
| `instruments` | TEXT[] | Instrument-Liste |
| `start_date` | DATE | Backtest-Startdatum |
| `end_date` | DATE | Backtest-Enddatum |
| `bar_interval` | VARCHAR(20) | BarInterval-Enum |
| `status` | VARCHAR(20) | QUEUED, RUNNING, COMPLETED, FAILED, CANCELLED |
| `summary_json` | JSONB | Vollstaendiger BacktestReport-Summary |
| `started_at` | TIMESTAMPTZ | Startzeitpunkt |
| `completed_at` | TIMESTAMPTZ | Endzeitpunkt |
| `error_message` | TEXT | Fehlermeldung bei FAILED |
| `batch_id` | UUID | FK zu TradingRun.batchId |

### V016 -- `odin.indicator_snapshot`

Entity: `IndicatorSnapshotEntity` (Modul: odin-brain)

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `id` | BIGSERIAL | PK |
| `run_id` | UUID | FK zu TradingRun |
| `instrument_id` | VARCHAR(20) | Instrument |
| `bar_time` | TIMESTAMPTZ | Bar-Zeitpunkt |
| `ema9` | DOUBLE PRECISION | EMA(9) |
| `ema21` | DOUBLE PRECISION | EMA(21) |
| `ema50` | DOUBLE PRECISION | EMA(50) |
| `ema100` | DOUBLE PRECISION | EMA(100) |
| `rsi14` | DOUBLE PRECISION | RSI(14) |
| `atr14` | DOUBLE PRECISION | ATR(14) |
| `bollinger_upper` | DOUBLE PRECISION | Bollinger Upper |
| `bollinger_mid` | DOUBLE PRECISION | Bollinger Middle |
| `bollinger_lower` | DOUBLE PRECISION | Bollinger Lower |
| `adx14` | DOUBLE PRECISION | ADX(14) |
| `atr_decay` | DOUBLE PRECISION | ATR Decay |
| `volume_ratio` | DOUBLE PRECISION | Volume Ratio |
| `vwap` | DOUBLE PRECISION | VWAP |

Index: `(run_id, instrument_id, bar_time)`

### V017 -- `odin.optimization_run`

Entity: `OptimizationRunEntity` (Modul: odin-brain)

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `optimization_id` | UUID | PK |
| `backtest_id` | UUID | FK Source-Backtest |
| `status` | VARCHAR(20) | PENDING, RUNNING, COMPLETED, FAILED |
| `suggested_params_json` | JSONB | Vorgeschlagene Parameter |
| `reasoning` | TEXT | LLM-Begruendung |
| `parent_optimization_id` | UUID | FK fuer iterative Optimierung |
| `result_backtest_id` | UUID | FK Ergebnis-Backtest |
| `created_at` | TIMESTAMPTZ | Erstellt |
| `completed_at` | TIMESTAMPTZ | Abgeschlossen |

### V018 -- `odin.parameter_set`

Entity: `ParameterSetEntity` (Modul: odin-brain oder odin-core)

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `id` | UUID | PK |
| `name` | VARCHAR(100) | Name des Sets |
| `description` | TEXT | Beschreibung |
| `source_type` | VARCHAR(20) | MANUAL, LLM_OPTIMIZED |
| `source_optimization_id` | UUID | FK falls LLM-generiert |
| `params_json` | JSONB | Parameter-Konfiguration |
| `performance_json` | JSONB | Letzte Performance-Summary |
| `created_at` | TIMESTAMPTZ | Erstellt |

---

## 6. Architekturentscheidungen

### A1: odin-backtest als Library

odin-backtest verliert seinen eigenen Spring Boot Main und wird zur Library. `BacktestRunner`, `DataDownloadService`, `YahooDataFetcher`, `IntradayBarJdbcRepository` bleiben unveraendert. odin-app importiert odin-backtest als Dependency. Die neue `BacktestRunEntity` lebt in odin-backtest (Domain-Owner).

**Alternative:** Backtest-Logik nach odin-core verschieben. Nicht empfohlen -- vergroessert odin-core unnoetig und vermischt Verantwortlichkeiten.

### A2: BarInterval nach odin-api verschieben

`BarInterval` liegt aktuell in odin-backtest. Da Chart-API und andere Module es benoetigen, sollte es nach `de.its.odin.api.model` verschoben werden.

### A3: Indikator-Persistierung in odin-brain

`IndicatorSnapshotEntity` gehoert zu odin-brain (dort liegt die KPI-Domaene). Nicht in odin-data (das sind Roh-Daten, keine interpretierten Indikatoren) und nicht in odin-persistence (keine fachlichen Inhalte).

### A4: REST-DTOs Lokation

Response-DTOs in `de.its.odin.app.dto` (odin-app-spezifisch, nicht in odin-api). Request-DTOs ebenfalls in odin-app. Grund: DTOs sind API-Layer-Konzepte, nicht Domaenen-Objekte.

### A5: Async Backtest Execution

`@Async` mit dediziertem `ThreadPoolTaskExecutor` (nicht Default). Konfigurierbar: `odin.web.backtest.pool-size=2`, `queue-capacity=10`. Alternative: `CompletableFuture` mit manuellem Executor. Empfehlung: Spring `@Async` -- einfacher, ausreichend fuer V1.

---

## 7. Risiken und Mitigationen

| Risiko | Wahrscheinlichkeit | Impact | Mitigation |
|--------|-------------------|--------|-----------|
| BacktestRunner-Thread-Safety | Mittel | Hoch | Jeder Backtest bekommt eigene POJO-Instanzen (bereits so implementiert). ExecutorService mit begrenztem Pool. |
| odin-backtest Dependency-Konflikte bei Integration in odin-app | Niedrig | Mittel | odin-backtest hat keine Web-Dependencies. Nur spring-boot-starter-json und starter-jdbc. Kompatibel. |
| SSE-Skalierung bei vielen Instrumenten | Niedrig | Mittel | V1: Max 2-3 Instrumente (Kap 0). SseEmitter mit Timeout und Heartbeat. |
| LLM-Optimierung: unpredictable Claude Responses | Mittel | Mittel | Strukturierter JSON-Response mit Schema-Validation. Fallback bei Invalid Response. |
| Indikator-Persistierung Performance | Niedrig | Niedrig | Batch-Insert. Pro Trading-Tag ~390 Rows (1m) oder ~78 Rows (5m). Vernachlaessigbar. |

---

## Requirements Count Summary

| Kategorie | Anzahl |
|-----------|--------|
| REQ-WEB: Web Application Foundation | 7 |
| REQ-BT: Backtest Management | 7 |
| REQ-CHART: Chart Data API | 5 |
| REQ-KPI: Extended KPI Engine | 4 |
| REQ-OPT: LLM Optimization Loop | 6 |
| REQ-LIVE: Live Monitoring | 4 |
| REQ-DATA: Data Management | 6 |
| **Total** | **39** |
