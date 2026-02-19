# API Interface Specification

**Version:** 1.0
**Stand:** 2026-02-19

Vollstaendige Schnittstellenspezifikation aller Frontend-Backend-Endpoints. Dient als autorative Quelle fuer die Frontend-Implementierung und das Backend-API-Design.

---

## 1. Existing Endpoints (Inventory)

### 1.1 Trading Run Management

#### `GET /api/v1/runs` — Paginated, Filtered Run List

**REQ:** REQ-DATA-001

| Aspect | Detail |
|--------|--------|
| **Controller** | `TradingRunController` |
| **Module** | odin-app |

**Request Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `mode` | `RuntimeMode` enum (`LIVE`, `SIMULATION`) | No | all | Filter by runtime mode |
| `tradingDate` | `LocalDate` (ISO) | No | all | Filter by trading date |
| `instrumentId` | `String` | No | all | Filter by instrument symbol |
| `status` | `TradingRunStatus` enum (`ACTIVE`, `COMPLETED`, `DAY_STOPPED`, `ERROR`) | No | all | Filter by run status |
| `batchId` | `String` | No | all | Filter by batch ID |
| `page` | `int` | No | 0 | Page number |
| `size` | `int` | No | 20 | Page size (max 100) |
| `sort` | `String` | No | `tradingDate,desc` | Sort field and direction |

**Response:** `Page<TradingRunDto>`

```typescript
type TradingRunDto = {
  runId: string;               // UUID
  mode: 'LIVE' | 'SIMULATION';
  instrumentId: string;
  exchange: string;
  tradingDate: string;         // ISO date
  promptVersion: string;
  modelId: string;
  configSnapshotId: string | null;
  startTime: string;           // ISO-8601 instant
  endTime: string | null;
  finalPnl: number | null;     // BigDecimal
  tradeCount: number;
  status: 'ACTIVE' | 'COMPLETED' | 'DAY_STOPPED' | 'ERROR';
  contractCurrency: string;
  batchId: string | null;
};
```

---

#### `GET /api/v1/runs/{runId}` — Run Detail

**REQ:** REQ-DATA-002

**Path Parameters:** `runId` (UUID)

**Response:** `TradingRunDto` (same as above)

**Errors:** 404 if run not found

---

#### `GET /api/v1/runs/{runId}/trades` — Trades of a Run

**REQ:** REQ-DATA-003, REQ-CHART-003

**Path Parameters:** `runId` (UUID)

**Response:** `List<TradeDto>` (unpaginated — naturally bounded to 1-3 trades per day)

```typescript
type TradeDto = {
  tradeId: string;             // UUID
  runId: string;               // UUID
  instrumentId: string;
  direction: 'LONG';           // v1: LONG only
  entryTime: string;           // ISO-8601 instant
  exitTime: string | null;
  entryPrice: number;          // BigDecimal
  exitPrice: number | null;
  quantity: number;
  realizedPnl: number | null;  // BigDecimal
  grossPnl: number | null;     // BigDecimal
  totalCost: number | null;    // BigDecimal
  regime: 'TRENDING' | 'MEAN_REVERTING' | 'VOLATILE' | 'LOW_VOLATILITY' | 'UNKNOWN';
  quantScore: number;
  stopLevel: number;           // BigDecimal
  riskRewardRatio: number;
  exitReason: string | null;   // ExitReason enum
  tranchenVariant: 'COMPACT_3T' | 'STANDARD_4T' | 'FULL_5T';
};
```

**Errors:** 404 if run not found

---

#### `GET /api/v1/runs/{runId}/decisions` — Decision Log Entries

**REQ:** REQ-CHART-005

**Path Parameters:** `runId` (UUID)

**Request Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `intentType` | `IntentType` enum (`ENTRY`, `EXIT`, `NO_ACTION`) | No | all | Filter by intent type |
| `page` | `int` | No | 0 | Page number |
| `size` | `int` | No | 100 | Page size |
| `sort` | `String` | No | `barCloseTime,asc` | Sort field and direction |

**Response:** `Page<DecisionLogDto>`

```typescript
type DecisionLogDto = {
  decisionId: string;          // UUID
  runId: string;               // UUID
  instrumentId: string;
  barCloseTime: string;        // ISO-8601 instant
  tradeId: string | null;      // UUID
  intentType: 'ENTRY' | 'EXIT' | 'NO_ACTION';
  pipelineState: 'INITIALIZING' | 'WARMUP' | 'OBSERVING' | 'POSITIONED' | 'PENDING_FILL' | 'HALTED' | 'DAY_STOPPED';
  quantScore: number;
  regimeAtDecision: string;    // Regime enum
  regimeConfidence: number;
  riskGateResult: string | null; // JSON string
  finalDecision: 'EXECUTE' | 'REJECT';
  rejectReason: string | null; // RejectReason enum
  emitter: string | null;
  marketClockTimestamp: string; // ISO-8601 instant
};
```

**Errors:** 404 if run not found

---

#### `GET /api/v1/runs/{runId}/llm-history` — LLM Call Metadata

**REQ:** REQ-DATA-004

**Path Parameters:** `runId` (UUID)

**Request Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | `int` | No | 0 | Page number |
| `size` | `int` | No | 20 | Page size |
| `sort` | `String` | No | `requestMarketTime,asc` | Sort field and direction |

**Response:** `Page<LlmCallSummaryDto>` (without inputPayload/outputPayload)

```typescript
type LlmCallSummaryDto = {
  callId: string;              // UUID
  runId: string;               // UUID
  instrumentId: string;
  provider: 'CLAUDE' | 'OPENAI' | 'CACHED';
  model: string;
  promptVersion: string;
  regime: string;              // Regime enum
  regimeConfidence: number;
  requestMarketTime: string;   // ISO-8601 instant
  responseMarketTime: string;  // ISO-8601 instant
  latencyMs: number;
  validationResult: 'VALID' | 'SCHEMA_ERROR' | 'PLAUSIBILITY_ERROR' | 'CONSISTENCY_ERROR';
};
```

**Errors:** 404 if run not found

---

### 1.2 Backtesting

#### `POST /api/v1/backtests` — Start New Backtest

**REQ:** REQ-BT-001

**Request Body:**

```typescript
type BacktestRequest = {
  instruments: string[];       // @NotEmpty
  startDate: string;           // @NotNull, ISO date
  endDate: string;             // @NotNull, ISO date
  barInterval: 'ONE_MINUTE' | 'FIVE_MINUTES'; // @NotNull, BarInterval enum
  autoDownload: boolean;       // default false
  parameterSetId?: string;     // UUID, nullable
};
```

**Response:** HTTP 202 Accepted

```typescript
type BacktestStartResponse = {
  backtestId: string;          // UUID
};
```

**Errors:** 400 if startDate > endDate, 404 if parameterSetId not found

---

#### `GET /api/v1/backtests/{backtestId}` — Backtest Status and Progress

**REQ:** REQ-BT-002

**Path Parameters:** `backtestId` (UUID)

**Response:** `BacktestStatusDto`

```typescript
type BacktestStatusDto = {
  backtestId: string;          // UUID
  instruments: string[];
  startDate: string;           // ISO date
  endDate: string;             // ISO date
  barInterval: 'ONE_MINUTE' | 'FIVE_MINUTES';
  status: 'QUEUED' | 'RUNNING' | 'COMPLETED' | 'FAILED' | 'CANCELLED';
  completedDays: number;
  totalDays: number;
  startedAt: string | null;    // ISO-8601 instant
  completedAt: string | null;
  errorMessage: string | null;
  batchId: string | null;      // UUID
  summaryJson: string | null;  // JSON string, populated when COMPLETED
};
```

**Errors:** 404 if backtest not found

---

#### `GET /api/v1/backtests/{backtestId}/report` — Full Backtest Report

**REQ:** REQ-BT-003

**Path Parameters:** `backtestId` (UUID)

**Response:** `BacktestStatusDto` (same shape; report available via `summaryJson`)

**Errors:** 404 if not found, 409 if not yet completed (no summaryJson)

---

#### `GET /api/v1/backtests` — Backtest History (Paginated)

**REQ:** REQ-BT-004

**Request Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `status` | `List<BacktestRunStatus>` | No | all | Filter by status(es) |
| `startFrom` | `LocalDate` | No | all | Lower bound for backtest start date |
| `endTo` | `LocalDate` | No | all | Upper bound for backtest end date |
| `page` | `int` | No | 0 | Page number |
| `size` | `int` | No | 20 | Page size |

**Response:** `Page<BacktestHistoryDto>`

```typescript
type BacktestHistoryDto = {
  backtestId: string;          // UUID
  instruments: string[];
  startDate: string;           // ISO date
  endDate: string;             // ISO date
  barInterval: 'ONE_MINUTE' | 'FIVE_MINUTES';
  status: 'QUEUED' | 'RUNNING' | 'COMPLETED' | 'FAILED' | 'CANCELLED';
  completedDays: number;
  totalDays: number;
  startedAt: string | null;    // ISO-8601 instant
  completedAt: string | null;
  summaryJson: string | null;
};
```

---

#### `GET /api/v1/backtests/{backtestId}/days` — Paginated Daily Details

**REQ:** REQ-BT-003 (detail)

**Path Parameters:** `backtestId` (UUID)

**Request Parameters:**

| Parameter | Type | Required | Default |
|-----------|------|----------|---------|
| `page` | `int` | No | 0 |
| `size` | `int` | No | 20 |

**Response:** `Page<BacktestDayResultDto>`

```typescript
type BacktestDayResultDto = {
  tradingDate: string;         // ISO date
  totalTrades: number;
  totalPnl: number;
  maxDrawdown: number;
  instruments: string[];
};
```

**Errors:** 404 if not found, 409 if no summary data available

---

#### `POST /api/v1/backtests/{backtestId}/cancel` — Cancel Running Backtest

**REQ:** REQ-BT-007

**Path Parameters:** `backtestId` (UUID)

**Response:**

```typescript
type CancelBacktestResponse = {
  backtestId: string;          // UUID
  accepted: boolean;
  message: string;
};
```

HTTP 200 if accepted, HTTP 409 if not currently running.

---

### 1.3 Data & Charts

#### `GET /api/v1/data/availability` — Check Data Availability

**REQ:** REQ-BT-006

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `instruments` | `List<String>` | Yes | Ticker symbols |
| `startDate` | `LocalDate` | Yes | Start date (inclusive) |
| `endDate` | `LocalDate` | Yes | End date (inclusive) |
| `barInterval` | `BarInterval` enum | Yes | Bar interval to check |

**Response:**

```typescript
type DataAvailabilityResponse = {
  available: Record<string, number>;        // instrument -> available days count
  missing: Record<string, number>;          // instrument -> missing days count
  missingDays: Record<string, string[]>;    // instrument -> list of missing ISO dates
};
```

---

#### `POST /api/v1/data/download` — Trigger Data Download

**REQ:** REQ-DATA-005

**Request Body:**

```typescript
type DataDownloadRequest = {
  instruments: string[];       // @NotEmpty
  startDate: string;           // @NotNull, ISO date
  endDate: string;             // @NotNull, ISO date
  barInterval: 'ONE_MINUTE' | 'FIVE_MINUTES'; // @NotNull
  exchange?: string;           // defaults to "SMART"
};
```

**Response:** HTTP 202 Accepted

```typescript
type DataDownloadResponse = {
  status: 'COMPLETED' | 'ALREADY_AVAILABLE';
  message: string;
};
```

**Note:** Current implementation is synchronous despite returning 202. The frontend should treat it as async.

---

#### `GET /api/v1/charts/{symbol}/bars` — OHLCV Bars

**REQ:** REQ-CHART-001, REQ-CHART-004

**Path Parameters:** `symbol` (String)

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tradingDate` | `LocalDate` | Conditional | Single day (takes precedence) |
| `startDate` | `LocalDate` | Conditional | Range start (with endDate) |
| `endDate` | `LocalDate` | Conditional | Range end (with startDate) |
| `interval` | `ChartInterval` enum | Yes | `ONE_MINUTE`, `FIVE_MINUTES`, `FIFTEEN_MINUTES` |

Either `tradingDate` OR both `startDate`/`endDate` must be provided.

**Response:** `List<BarDto>` (sorted by openTime ascending)

```typescript
type BarDto = {
  openTime: string;            // ISO-8601 instant
  closeTime: string;           // ISO-8601 instant
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;              // long
};
```

15-minute bars are computed on-the-fly from stored 5-minute bars.

---

#### `GET /api/v1/charts/{symbol}/indicators` — Indicator Overlays

**REQ:** REQ-CHART-002, REQ-KPI-004

**Path Parameters:** `symbol` (String)

**Request Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `tradingDate` | `LocalDate` | Yes | — | Trading date |
| `runId` | `UUID` | No | null | If provided: load persisted indicators. If omitted: on-the-fly calculation |
| `interval` | `ChartInterval` | No | `FIVE_MINUTES` | Bar interval for on-the-fly calculation |

**Response:** `List<IndicatorDto>` (sorted by barTime ascending)

```typescript
type IndicatorDto = {
  barTime: string;             // ISO-8601 instant
  ema9: number | null;         // null during warmup
  ema21: number | null;
  ema50: number | null;
  ema100: number | null;
  rsi14: number | null;
  atr14: number | null;
  bollingerUpper: number | null;
  bollingerMiddle: number | null;
  bollingerLower: number | null;
  adx14: number | null;
  atrDecay: number | null;
  volumeRatio: number | null;
  vwap: number | null;
};
```

---

### 1.4 Controls

#### `POST /api/v1/controls/kill` — Activate Global Kill Switch

**REQ:** REQ-LIVE-003

**Request Body:** None

**Response:** `ControlResponse`

```typescript
type ControlResponse = {
  status: 'ACCEPTED' | 'REJECTED' | 'NOOP';
  reasonCode: string | null;   // e.g. "ALREADY_ACTIVE"
  marketClockTimestamp: string; // ISO-8601 instant
};
```

- `ACCEPTED` (200): Kill switch activated
- `NOOP` (200): Already active (`reasonCode: "ALREADY_ACTIVE"`)

---

#### `POST /api/v1/controls/pause/{instrumentId}` — Pause Pipeline

**REQ:** REQ-LIVE-003

**Path Parameters:** `instrumentId` (String)

**Response:** `ControlResponse`

- `ACCEPTED` (200): Pipeline paused
- `NOOP` (200): Already paused (`reasonCode: "ALREADY_PAUSED"`)
- `REJECTED` (409): Invalid state (`reasonCode: "INVALID_STATE:{currentState}"`)
- `REJECTED` (404): Instrument not found (`reasonCode: "INSTRUMENT_NOT_FOUND"`)

Pause allowed from: WARMUP, OBSERVING, POSITIONED.
Pause rejected from: PENDING_FILL, DAY_STOPPED, INITIALIZING, HALTED.

---

#### `POST /api/v1/controls/resume/{instrumentId}` — Resume Pipeline

**REQ:** REQ-LIVE-003

**Path Parameters:** `instrumentId` (String)

**Response:** `ControlResponse`

- `ACCEPTED` (200): Pipeline resumed
- `REJECTED` (409): Not in HALTED state (`reasonCode: "NOT_HALTED:{currentState}"`)
- `REJECTED` (409): Kill switch active (`reasonCode: "KILL_SWITCH_ACTIVE"`)
- `REJECTED` (404): Instrument not found (`reasonCode: "INSTRUMENT_NOT_FOUND"`)

---

### 1.5 SSE Streams

#### `GET /api/v1/stream/instruments/{instrumentId}` — Instrument SSE Stream

**REQ:** REQ-LIVE-001

**Path Parameters:** `instrumentId` (String)

**Headers:** `Last-Event-ID` (optional, for reconnect replay)

**Content-Type:** `text/event-stream`

**Event Types:**

| Event | Payload Description |
|-------|---------------------|
| `snapshot` | Price, Bid/Ask, Spread, Volume, VWAP, RSI, ATR. Contains `runId` and `instrumentId` |
| `state-change` | Pipeline state (INITIALIZING, WARMUP, OBSERVING, POSITIONED, PENDING_FILL, HALTED, DAY_STOPPED) |
| `trade-update` | Open position, unrealized P&L, tranche status, tranchenVariant |
| `order-update` | Order status (CREATED, SUBMITTED, ACCEPTED, FILLED, PARTIALLY_FILLED, CANCELLED, REJECTED) |
| `llm-update` | Regime, regimeConfidence, requestMarketTime |
| `alert` | Alert level (INFO, WARNING, CRITICAL), message, emitter |
| `pnl` | Tages-P&L (realized + unrealized), grossPnl, netPnl, tradeCount |
| `heartbeat` | Keep-alive (periodic) |

**Reconnect:** Events replayed from in-memory ring buffer (max 60s). eventId is monoton steigend per stream.

---

#### `GET /api/v1/stream/global` — Global SSE Stream

**REQ:** REQ-LIVE-002

**Headers:** `Last-Event-ID` (optional, for reconnect replay)

**Content-Type:** `text/event-stream`

**Event Types:**

| Event | Payload Description |
|-------|---------------------|
| `risk-update` | AccountRiskState: remainingBudget, exposure, roundTrips, coolingOff |
| `alert` | Global alerts |
| `pnl` | Aggregated P&L across all pipelines |
| `heartbeat` | Keep-alive (periodic) |

---

### 1.6 Health & System

#### `GET /api/v1/health` — Health Check

**Legacy alias:** `GET /api/health`

**Response:**

```typescript
type HealthResponse = {
  status: 'UP';
  mode: 'LIVE' | 'SIMULATION';
};
```

---

#### `GET /api/v1/instruments` — Configured Instruments

**Legacy alias:** `GET /api/instruments`

**Response:** `string[]` (list of ticker symbols, e.g. `["AAPL", "MSFT"]`)

---

### 1.7 Optimization

#### `POST /api/v1/backtests/{backtestId}/optimize` — Start LLM Optimization

**REQ:** REQ-OPT-001

**Path Parameters:** `backtestId` (UUID)

**Request Body:** (optional)

```typescript
type OptimizationRequest = {
  parentOptimizationId?: string;  // UUID, for iterative chains
};
```

**Response:** HTTP 202 Accepted

```typescript
type OptimizationStartResponse = {
  optimizationId: string;      // UUID
};
```

**Errors:** 404 if backtest not found, 409 if backtest not COMPLETED

---

#### `GET /api/v1/optimizations/{optimizationId}` — Optimization Status

**REQ:** REQ-OPT-002

**Path Parameters:** `optimizationId` (UUID)

**Response:**

```typescript
type OptimizationStatusDto = {
  optimizationId: string;      // UUID
  backtestId: string;          // UUID
  status: 'QUEUED' | 'RUNNING' | 'COMPLETED' | 'FAILED';
  suggestedParamsJson: string | null;
  reasoning: string | null;
  expectedImpact: string | null;
  parentOptimizationId: string | null;  // UUID
  resultBacktestId: string | null;      // UUID
  createdAt: string;           // ISO-8601 instant
  completedAt: string | null;
  errorMessage: string | null;
};
```

**Errors:** 404 if optimization not found

---

### 1.8 Parameter Sets

#### `GET /api/v1/parameter-sets` — List Parameter Sets (Paginated)

**REQ:** REQ-OPT-004

**Request Parameters:**

| Parameter | Type | Required | Default |
|-----------|------|----------|---------|
| `sourceType` | `ParameterSetSourceType` (`MANUAL`, `LLM_OPTIMIZED`) | No | all |
| `page` | `int` | No | 0 |
| `size` | `int` | No | 20 |

**Response:** `Page<ParameterSetDto>`

```typescript
type ParameterSetDto = {
  id: string;                  // UUID
  name: string;
  description: string | null;
  sourceType: 'MANUAL' | 'LLM_OPTIMIZED';
  sourceOptimizationId: string | null;  // UUID
  paramsJson: string;          // JSONB string
  performanceJson: string | null;
  createdAt: string;           // ISO-8601 instant
  updatedAt: string | null;
};
```

---

#### `GET /api/v1/parameter-sets/{id}` — Get Parameter Set

**Path Parameters:** `id` (UUID)

**Response:** `ParameterSetDto`

**Errors:** 404 if not found

---

#### `POST /api/v1/parameter-sets` — Create Parameter Set

**Request Body:**

```typescript
type CreateParameterSetRequest = {
  name: string;                // @NotBlank, max 100 chars
  description?: string;
  paramsJson: string;          // @NotNull
};
```

**Response:** HTTP 201 Created, body: `ParameterSetDto`

---

#### `PUT /api/v1/parameter-sets/{id}` — Update Parameter Set

**Path Parameters:** `id` (UUID)

**Request Body:**

```typescript
type UpdateParameterSetRequest = {
  name: string;                // @NotBlank, max 100 chars
  description?: string;
  paramsJson: string;          // @NotNull
};
```

**Response:** `ParameterSetDto`

**Errors:** 404 if not found

---

#### `DELETE /api/v1/parameter-sets/{id}` — Delete Parameter Set

**Path Parameters:** `id` (UUID)

**Response:** HTTP 204 No Content

**Errors:** 404 if not found

---

### 1.9 Simulation

#### `POST /api/v1/runs/simulation` — Trigger Single-Day Simulation

**Legacy alias:** `POST /api/runs/simulation`

**Condition:** Only active when `odin.agent.mode=SIMULATION`

**Request Parameters:**

| Parameter | Type | Required | Default |
|-----------|------|----------|---------|
| `tradingDate` | `LocalDate` | No | Yesterday in exchange TZ |

**Response:** `SimulationResult` (aggregated results from `SimulationRunner`)

---

### 1.10 Error Handling

**Controller:** `GlobalExceptionHandler` (`@RestControllerAdvice`)

All errors return standardized JSON:

```typescript
// Standard error
type ApiError = {
  timestamp: string;           // ISO-8601 instant
  status: number;              // HTTP status code
  error: string;               // HTTP status reason phrase
  message: string;             // Human-readable description
  path: string;                // Request URI
};

// Validation error (extends ApiError)
type ValidationError = ApiError & {
  violations: FieldViolation[];
};

type FieldViolation = {
  field: string;
  message: string;
};
```

**Exception Mapping:**

| Exception | HTTP Status | Error Type |
|-----------|-------------|-----------|
| `MethodArgumentNotValidException` | 400 | `ValidationError` |
| `ConstraintViolationException` | 400 | `ValidationError` |
| `HttpMessageNotReadableException` | 400 | `ApiError` |
| `TypeMismatchException` | 400 | `ApiError` |
| `MissingServletRequestParameterException` | 400 | `ApiError` |
| `IllegalArgumentException` | 400 | `ApiError` |
| `EntityNotFoundException` | 404 | `ApiError` |
| `IllegalStateException` | 409 | `ApiError` |
| Unhandled `Exception` | 500 | `ApiError` (generic message, no stack trace) |

---

## 2. UI Requirements by View

### 2.1 Dashboard

The Dashboard provides an at-a-glance overview of system state.

| Feature | Data Source | Endpoint |
|---------|------------|----------|
| Active Trading Runs (today) | REST | `GET /api/v1/runs?mode=LIVE&status=ACTIVE&tradingDate={today}` |
| Active Backtests (running) | REST | `GET /api/v1/backtests?status=RUNNING` |
| Tages-P&L Summary (all pipelines) | SSE | Global stream `pnl` events |
| Global Risk State | SSE | Global stream `risk-update` events |
| Kill-Switch State | SSE | Global stream `risk-update` events (coolingOff, etc.) |
| Alert Feed | SSE | Global stream `alert` events |
| Active Instrument List | REST | `GET /api/v1/instruments` |
| System Health | REST | `GET /api/v1/health` |

### 2.2 Backtesting

| Feature | Data Source | Endpoint |
|---------|------------|----------|
| Backtest History (list, filterable) | REST | `GET /api/v1/backtests?status=...&startFrom=...&endTo=...` |
| Start New Backtest | REST | `POST /api/v1/backtests` |
| Backtest Status / Progress | REST (polling) | `GET /api/v1/backtests/{backtestId}` |
| Backtest Full Report | REST | `GET /api/v1/backtests/{backtestId}/report` |
| Daily Results (paginated) | REST | `GET /api/v1/backtests/{backtestId}/days` |
| Cancel Running Backtest | REST | `POST /api/v1/backtests/{backtestId}/cancel` |
| Chart Data for Day | REST | `GET /api/v1/charts/{symbol}/bars`, `GET /api/v1/charts/{symbol}/indicators` |
| Trades per Run (chart markers) | REST | `GET /api/v1/runs/{runId}/trades` |
| Decision Log per Run | REST | `GET /api/v1/runs/{runId}/decisions` |
| Runs by Backtest Batch | REST | `GET /api/v1/runs?batchId={batchId}` |
| Data Availability Check | REST | `GET /api/v1/data/availability` |
| Data Download | REST | `POST /api/v1/data/download` |
| Start Optimization | REST | `POST /api/v1/backtests/{backtestId}/optimize` |
| Optimization Status | REST | `GET /api/v1/optimizations/{optimizationId}` |
| Parameter Set CRUD | REST | `GET/POST/PUT/DELETE /api/v1/parameter-sets` |

### 2.3 Trading Operations (Live)

| Feature | Data Source | Endpoint |
|---------|------------|----------|
| Live Pipeline Telemetry | SSE | Instrument stream: `snapshot`, `state-change`, `trade-update`, `order-update`, `llm-update`, `pnl` |
| Pipeline Detail (State, OMS, Tranchen) | REST + SSE | `GET /api/v1/runs/{runId}` + instrument stream |
| Decision Log (forensics) | REST | `GET /api/v1/runs/{runId}/decisions` |
| LLM History (forensics) | REST | `GET /api/v1/runs/{runId}/llm-history` |
| Kill-Switch Control | REST | `POST /api/v1/controls/kill` |
| Pause Pipeline | REST | `POST /api/v1/controls/pause/{instrumentId}` |
| Resume Pipeline | REST | `POST /api/v1/controls/resume/{instrumentId}` |
| Live Chart (Candlestick + Indicators) | REST + SSE | `GET /api/v1/charts/{symbol}/bars`, SSE `snapshot` for live updates |
| Alert Feed | SSE | Instrument stream `alert` events |

---

## 3. Gap Analysis

### 3.1 Missing Endpoints

| Gap | Description | Required By | Priority |
|-----|-------------|-------------|----------|
| **`GET /api/v1/runs/current`** | Shorthand for current live runs (today, LIVE, ACTIVE). Defined in Kap 9 but not implemented as a dedicated endpoint. Currently achievable via filters on `GET /api/v1/runs?mode=LIVE&status=ACTIVE&tradingDate={today}`. | Dashboard | P1 |
| **`GET /api/v1/exchanges/{exchangeId}/session`** | Session times (Pre-Market, RTH, After-Hours) for chart session shading. Required by Chart-Komponente (chart-component-concept.md section 12). | Chart | P2 |
| **SSE Backtest Progress Stream** | Backtest progress as SSE events instead of REST polling. Would eliminate polling overhead for long-running backtests. Currently: clients must poll `GET /api/v1/backtests/{backtestId}`. | Backtesting | P2 |
| **`GET /api/v1/symbols/search`** | Symbol search/autocomplete for instrument selection in backtest form. REQ-DATA-006. | Backtesting UI | P2 |

### 3.2 Endpoints Needing Changes

| Endpoint | Change | Reason | Priority |
|----------|--------|--------|----------|
| **SSE Instrument Stream** (`/api/v1/stream/instruments/{instrumentId}`) | Add `?timeframe=5m` query parameter | Chart-Komponente requires timeframe-specific bar data in `snapshot` events (chart-component-concept.md section 7). Backend must aggregate ticks into the subscribed timeframe. | P1 |
| **SSE `snapshot` event payload** | Add bar-related fields: `barOpenTime`, `barOpen`, `barHigh`, `barLow`, `lastPrice`, `barVolume` | Chart live-update requires timeframe-conforming bar data (chart-component-concept.md section 9). Currently snapshot payload is not specified in code. | P1 |
| **REST Bar/Indicator responses** | Add `asOfEventId` field to responses | Chart-Komponente Handover-Protokoll requires correlation between REST snapshot and SSE stream position (chart-component-concept.md section 9). | P1 |
| **SSE `trade-update` payload** | Ensure stable `tradeId` field | Needed as idempotency key for SSE overlap dedup during timeframe switch (chart-component-concept.md section 11). | P1 |
| **SSE `order-update` payload** | Ensure stable `orderId` field | Needed as idempotency key for SSE overlap dedup during timeframe switch (chart-component-concept.md section 11). | P1 |
| **`POST /api/v1/data/download`** | Make truly asynchronous (return immediately, track progress) | REQ-DATA-005 specifies async. Current implementation is synchronous. For large downloads, the frontend could time out. | P1 |
| **`GET /api/v1/backtests/{backtestId}/report`** | Return the actual report DTO, not `BacktestStatusDto` | Currently returns the same `BacktestStatusDto`. Should return a dedicated report type with parsed summary, or the controller should be renamed to make it clear this is the same shape. | P0 |
| **`BacktestDayResultDto`** | Add more fields from daily results: `winRate`, `sharpeApprox`, per-instrument breakdown | REQ-BT-003 specifies "daily results with all summary fields". Current DTO only has `tradingDate`, `totalTrades`, `totalPnl`, `maxDrawdown`, `instruments`. | P1 |

### 3.3 Already Complete

| Endpoint / Feature | Status | Notes |
|---------------------|--------|-------|
| `GET /api/v1/runs` (paginated, filtered) | **Complete** | All filters (mode, tradingDate, instrumentId, status, batchId) implemented. REQ-DATA-001 fulfilled. |
| `GET /api/v1/runs/{runId}` | **Complete** | REQ-DATA-002 fulfilled. |
| `GET /api/v1/runs/{runId}/trades` | **Complete** | REQ-DATA-003, REQ-CHART-003 fulfilled. |
| `GET /api/v1/runs/{runId}/decisions` | **Complete** | REQ-CHART-005 fulfilled. Filter by intentType, paginated. |
| `GET /api/v1/runs/{runId}/llm-history` | **Complete** | REQ-DATA-004 fulfilled. Without payload fields. |
| `POST /api/v1/backtests` | **Complete** | REQ-BT-001 fulfilled. Async execution. ParameterSet support. |
| `GET /api/v1/backtests/{backtestId}` | **Complete** | REQ-BT-002 fulfilled. Status + progress. |
| `GET /api/v1/backtests` (history) | **Complete** | REQ-BT-004 fulfilled. Filterable, paginated. |
| `GET /api/v1/backtests/{backtestId}/days` | **Complete** | REQ-BT-003 detail fulfilled. |
| `POST /api/v1/backtests/{backtestId}/cancel` | **Complete** | REQ-BT-007 fulfilled. |
| `GET /api/v1/data/availability` | **Complete** | REQ-BT-006 fulfilled. |
| `POST /api/v1/data/download` | **Partial** | REQ-DATA-005: Endpoint exists but is synchronous. |
| `GET /api/v1/charts/{symbol}/bars` | **Complete** | REQ-CHART-001, REQ-CHART-004 fulfilled. 1m, 5m, 15m. |
| `GET /api/v1/charts/{symbol}/indicators` | **Complete** | REQ-CHART-002, REQ-KPI-004 fulfilled. Persisted + on-the-fly. |
| `POST /api/v1/controls/kill` | **Complete** | REQ-LIVE-003 fulfilled. |
| `POST /api/v1/controls/pause/{instrumentId}` | **Complete** | REQ-LIVE-003 fulfilled. FSM validation. |
| `POST /api/v1/controls/resume/{instrumentId}` | **Complete** | REQ-LIVE-003 fulfilled. Kill-switch check. |
| SSE Instrument Stream | **Complete** | REQ-LIVE-001 fulfilled. Ring buffer, replay, heartbeat. |
| SSE Global Stream | **Complete** | REQ-LIVE-002 fulfilled. |
| `POST /api/v1/backtests/{backtestId}/optimize` | **Complete** | REQ-OPT-001 fulfilled. |
| `GET /api/v1/optimizations/{optimizationId}` | **Complete** | REQ-OPT-002 fulfilled. |
| `GET/POST/PUT/DELETE /api/v1/parameter-sets` | **Complete** | REQ-OPT-004 fulfilled. Full CRUD. |
| Global Exception Handler | **Complete** | REQ-WEB-001 fulfilled. |
| CORS (assumed configured) | **Complete** | REQ-WEB-002 fulfilled. |
| JSON serialization (Jackson) | **Complete** | REQ-WEB-003 fulfilled. |
| Pagination | **Complete** | REQ-WEB-004 fulfilled. |
| API Versioning (`/api/v1/`) | **Complete** | REQ-WEB-005 fulfilled. Legacy aliases retained. |
| Request Validation | **Complete** | REQ-WEB-006 fulfilled. |

---

## 4. New Endpoint Specifications

### 4.1 `GET /api/v1/runs/current` — Current Live Runs

A convenience endpoint returning today's active live runs.

| Aspect | Detail |
|--------|--------|
| **Method** | GET |
| **Path** | `/api/v1/runs/current` |
| **Controller** | `TradingRunController` |
| **Module** | odin-app |
| **Priority** | P1 |

**Request Parameters:** None

**Response:** `List<TradingRunDto>` (not paginated — max 2-3 runs)

**Implementation:** Delegates to `tradingRunRepository.findByFilters(LIVE, today, null, ACTIVE, null, unpaged)`.

**TypeScript:**

```typescript
// Response is TradingRunDto[] (same shape as paginated version, but unwrapped list)
```

---

### 4.2 `GET /api/v1/exchanges/{exchangeId}/session` — Exchange Session Times

Returns trading session times for chart session shading (chart-component-concept.md section 12).

| Aspect | Detail |
|--------|--------|
| **Method** | GET |
| **Path** | `/api/v1/exchanges/{exchangeId}/session` |
| **Controller** | New `ExchangeController` or extend `HealthController` |
| **Module** | odin-app (uses data from odin-core `CoreProperties` or TradingCalendar) |
| **Priority** | P2 |

**Path Parameters:** `exchangeId` (String, e.g. `"NYSE"`, `"NASDAQ"`)

**Response:**

```typescript
type ExchangeSessionResponse = {
  exchangeId: string;
  timezone: string;             // e.g. "America/New_York"
  preMarketOpen: string;        // ISO time, e.g. "04:00:00"
  regularOpen: string;          // e.g. "09:30:00"
  regularClose: string;         // e.g. "16:00:00"
  afterHoursClose: string;      // e.g. "20:00:00"
};
```

**Notes:** V1 can return a static configuration. The session times are needed for the chart session zone shading (Pre-Market, RTH, After-Hours).

---

### 4.3 `GET /api/v1/symbols/search` — Symbol Search

Provides symbol lookup for instrument selection in the backtest form.

| Aspect | Detail |
|--------|--------|
| **Method** | GET |
| **Path** | `/api/v1/symbols/search` |
| **Controller** | New `SymbolController` |
| **Module** | odin-app |
| **Priority** | P2 |

**Request Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | `String` | Yes | Search query (min 1 char) |

**Response:** `List<SymbolDto>` (max 20 results)

```typescript
type SymbolDto = {
  symbol: string;              // e.g. "AAPL"
  name: string;                // e.g. "Apple Inc."
  exchange: string;            // e.g. "NASDAQ"
};
```

**Notes:** V1 implementation can use a static list of common US tickers or Yahoo Finance API lookup.

---

### 4.4 SSE Backtest Progress Stream (Alternative to Polling)

| Aspect | Detail |
|--------|--------|
| **Method** | GET |
| **Path** | `/api/v1/stream/backtests/{backtestId}` |
| **Controller** | `StreamController` (extend) |
| **Module** | odin-app |
| **Priority** | P2 |

**Path Parameters:** `backtestId` (UUID)

**Content-Type:** `text/event-stream`

**Event Types:**

| Event | Payload |
|-------|---------|
| `progress` | `{ completedDays: number, totalDays: number, currentInstrument: string, currentDate: string }` |
| `completed` | `{ backtestId: string, summaryJson: string }` |
| `failed` | `{ backtestId: string, errorMessage: string }` |
| `cancelled` | `{ backtestId: string, completedDays: number }` |

**Notes:** This is a nice-to-have. REST polling on `GET /api/v1/backtests/{backtestId}` works for V1. This SSE stream would eliminate polling for long-running backtests and provide smoother progress bars.

---

## 5. Endpoint Change Specifications

### 5.1 SSE Instrument Stream: Add Timeframe Parameter

**Endpoint:** `GET /api/v1/stream/instruments/{instrumentId}`

**Change:** Add query parameter `?timeframe=5m` (values: `1m`, `5m`, `15m`, default: `5m`).

**Reason:** The Chart-Komponente requires bar data in the subscribed timeframe within `snapshot` events (chart-component-concept.md section 7). The backend must aggregate raw ticks into the subscribed timeframe and deliver timeframe-conforming bar fields.

**Implementation:** `StreamController` receives the timeframe parameter and passes it to `SseEmitterManager`. The pipeline's bar builder must produce timeframe-specific snapshots.

**Priority:** P1

---

### 5.2 SSE `snapshot` Event: Add Bar-Level Fields

**Change:** The `snapshot` SSE event payload must include these bar-related fields:

```typescript
type SnapshotEventPayload = {
  // Existing fields
  runId: string;
  instrumentId: string;
  lastPrice: number;
  bidPrice: number;
  askPrice: number;
  spread: number;
  volume: number;
  vwap: number | null;
  rsi14: number | null;
  atr14: number | null;
  mode: 'LIVE' | 'SIMULATION';

  // NEW: Bar-level fields (timeframe-conforming)
  barOpenTime: number;         // Unix timestamp (seconds)
  barOpen: number;
  barHigh: number;
  barLow: number;
  barVolume: number;
};
```

**Reason:** Chart live-update (chart-component-concept.md section 9) requires `barOpenTime` to distinguish "update current bar" from "new bar begins". The values must correspond to the subscribed timeframe.

**Priority:** P1

---

### 5.3 REST Responses: Add `asOfEventId` Field

**Affected Endpoints:**
- `GET /api/v1/charts/{symbol}/bars`
- `GET /api/v1/charts/{symbol}/indicators`

**Change:** Add `asOfEventId: number` field to the response wrapper (or as HTTP response header `X-As-Of-Event-Id`).

```typescript
// Option A: Response wrapper
type BarsResponse = {
  bars: BarDto[];
  asOfEventId: number;         // Highest SSE eventId at time of query
};

// Option B: HTTP response header
// X-As-Of-Event-Id: 42
```

**Reason:** The Chart-Komponente Handover-Protokoll (chart-component-concept.md section 9) requires correlation between the REST snapshot and the SSE event stream to deduplicate events during initial load.

**Recommendation:** Use HTTP response header (`X-As-Of-Event-Id`) to avoid changing the response body shape. This keeps the DTO clean and the header is easy to read from the fetch response.

**Priority:** P1

---

### 5.4 SSE Event Payloads: Ensure Stable IDs

**`trade-update`:** Payload must include a stable `tradeId` field (UUID).
**`order-update`:** Payload must include a stable `orderId` field (UUID or order reference number).

**Reason:** Used as idempotency keys for cross-stream deduplication during timeframe switches (chart-component-concept.md section 11).

**Priority:** P1

---

### 5.5 `POST /api/v1/data/download`: Make Truly Asynchronous

**Current:** Synchronous — blocks until download completes.
**Desired:** Return HTTP 202 immediately with a `downloadId`, poll progress via GET.

**Option A (minimal):** Run the download in a background thread, return 202. No progress tracking — just "started". Frontend polls `GET /api/v1/data/availability` to check completion.

**Option B (full):** New entity `DataDownloadRun` with status tracking. `GET /api/v1/data/downloads/{downloadId}` for progress.

**Recommendation:** Option A for V1 — the download is typically fast (seconds), and `availability` check is already available.

**Priority:** P1

---

### 5.6 `BacktestDayResultDto`: Enrich with More Fields

**Current fields:** `tradingDate`, `totalTrades`, `totalPnl`, `maxDrawdown`, `instruments`

**Add fields:**

```typescript
type BacktestDayResultDto = {
  tradingDate: string;
  totalTrades: number;
  totalPnl: number;
  maxDrawdown: number;
  instruments: string[];
  // NEW
  winRate: number | null;              // Win rate for the day (0.0-1.0)
  winners: number;                     // Number of winning trades
  losers: number;                      // Number of losing trades
  grossPnl: number;                    // P&L before costs
  totalCost: number;                   // Transaction costs
  avgWin: number | null;               // Average winning trade P&L
  avgLoss: number | null;              // Average losing trade P&L
};
```

**Reason:** REQ-BT-003 specifies "daily results with all summary fields". The backtest results view needs these fields for meaningful per-day analysis.

**Priority:** P1

---

## 6. SSE Event Types

### 6.1 Existing Events

Defined in `SseEventType.java`, implemented in `SseEmitterManager`:

| Event Type | Stream | Description | Defined |
|-----------|--------|-------------|---------|
| `snapshot` | Instrument | Market data snapshot | Yes |
| `state-change` | Instrument | Pipeline state transition | Yes |
| `trade-update` | Instrument | Trade/position update | Yes |
| `order-update` | Instrument | Order lifecycle update | Yes |
| `llm-update` | Instrument | LLM analysis result | Yes |
| `alert` | Both | Alert notification | Yes |
| `pnl` | Both | P&L update | Yes |
| `risk-update` | Global | Account risk state | Yes |
| `heartbeat` | Both | Keep-alive | Yes |

### 6.2 New Events Needed

| Event Type | Stream | Description | Priority |
|-----------|--------|-------------|----------|
| `backtest-progress` | Dedicated backtest stream | Backtest day completion progress | P2 |
| `backtest-completed` | Dedicated backtest stream | Backtest finished (success/fail) | P2 |

**Note:** Backtest progress events are a nice-to-have enhancement. For V1, REST polling is sufficient.

### 6.3 SSE Event Payload TypeScript Definitions

```typescript
// -- Instrument Stream Events --

type SnapshotEvent = {
  type: 'snapshot';
  runId: string;
  instrumentId: string;
  lastPrice: number;
  bidPrice: number;
  askPrice: number;
  spread: number;
  volume: number;
  vwap: number | null;
  rsi14: number | null;
  atr14: number | null;
  mode: 'LIVE' | 'SIMULATION';
  barOpenTime: number;         // Unix timestamp (seconds)
  barOpen: number;
  barHigh: number;
  barLow: number;
  barVolume: number;
};

type StateChangeEvent = {
  type: 'state-change';
  runId: string;
  instrumentId: string;
  previousState: PipelineState;
  newState: PipelineState;
  marketClockTimestamp: string;
};

type TradeUpdateEvent = {
  type: 'trade-update';
  runId: string;
  instrumentId: string;
  tradeId: string;             // Stable UUID for dedup
  direction: 'LONG';
  entryPrice: number;
  quantity: number;
  unrealizedPnl: number;
  tranchenVariant: string;
  tranchesFilled: number;
  tranchesTotal: number;
  stopLevel: number;
  currentPrice: number;
};

type OrderUpdateEvent = {
  type: 'order-update';
  runId: string;
  instrumentId: string;
  orderId: string;             // Stable ID for dedup
  orderStatus: 'CREATED' | 'SUBMITTED' | 'ACCEPTED' | 'FILLED' | 'PARTIALLY_FILLED' | 'CANCELLED' | 'REJECTED';
  orderType: string;
  quantity: number;
  filledQuantity: number;
  limitPrice: number | null;
  stopPrice: number | null;
};

type LlmUpdateEvent = {
  type: 'llm-update';
  runId: string;
  instrumentId: string;
  regime: string;
  regimeConfidence: number;
  requestMarketTime: string;
  provider: string;
  model: string;
};

type AlertEvent = {
  type: 'alert';
  level: 'INFO' | 'WARNING' | 'CRITICAL';
  message: string;
  emitter: string;
  instrumentId: string | null;  // null for global alerts
  marketClockTimestamp: string;
};

type PnlEvent = {
  type: 'pnl';
  runId: string | null;        // null for global aggregate
  instrumentId: string | null;
  realizedPnl: number;
  unrealizedPnl: number;
  grossPnl: number;
  netPnl: number;
  tradeCount: number;
};

// -- Global Stream Events --

type RiskUpdateEvent = {
  type: 'risk-update';
  remainingBudget: number;
  exposure: number;
  roundTrips: number;
  coolingOff: boolean;
  killSwitchActive: boolean;
  killSwitchReason: string | null;
  marketClockTimestamp: string;
};

type HeartbeatEvent = {
  type: 'heartbeat';
};

// -- Discriminated Union --

type InstrumentStreamEvent =
  | SnapshotEvent
  | StateChangeEvent
  | TradeUpdateEvent
  | OrderUpdateEvent
  | LlmUpdateEvent
  | AlertEvent
  | PnlEvent
  | HeartbeatEvent;

type GlobalStreamEvent =
  | RiskUpdateEvent
  | AlertEvent
  | PnlEvent
  | HeartbeatEvent;
```

---

## 7. TypeScript Type Definitions

Complete shared type definitions for frontend consumption. These types correspond 1:1 to the Java DTOs and enums defined in the backend.

### 7.1 Enums

```typescript
// Runtime mode
type RuntimeMode = 'LIVE' | 'SIMULATION';

// Trading run status
type TradingRunStatus = 'ACTIVE' | 'COMPLETED' | 'DAY_STOPPED' | 'ERROR';

// Backtest run status
type BacktestRunStatus = 'QUEUED' | 'RUNNING' | 'COMPLETED' | 'FAILED' | 'CANCELLED';

// Optimization run status
type OptimizationRunStatus = 'QUEUED' | 'RUNNING' | 'COMPLETED' | 'FAILED';

// Pipeline state (FSM)
type PipelineState =
  | 'INITIALIZING' | 'WARMUP' | 'OBSERVING' | 'POSITIONED'
  | 'PENDING_FILL' | 'HALTED' | 'DAY_STOPPED';

// Market regime
type Regime =
  | 'TRENDING' | 'MEAN_REVERTING' | 'VOLATILE'
  | 'LOW_VOLATILITY' | 'UNKNOWN';

// Intent type
type IntentType = 'ENTRY' | 'EXIT' | 'NO_ACTION';

// Final decision
type FinalDecision = 'EXECUTE' | 'REJECT';

// Trade direction
type TradeDirection = 'LONG';

// Tranchen variant
type TranchenVariant = 'COMPACT_3T' | 'STANDARD_4T' | 'FULL_5T';

// Exit reason
type ExitReason =
  | 'STOP_LOSS' | 'TARGET_EXIT' | 'TRAILING_STOP' | 'EOD_FLAT'
  | 'KILL_SWITCH' | 'MANUAL' | 'REGIME_CHANGE' | 'TIME_STOP';

// LLM provider
type LlmProvider = 'CLAUDE' | 'OPENAI' | 'CACHED';

// LLM validation result
type ValidationResult =
  | 'VALID' | 'SCHEMA_ERROR' | 'PLAUSIBILITY_ERROR' | 'CONSISTENCY_ERROR';

// Control status
type ControlStatus = 'ACCEPTED' | 'REJECTED' | 'NOOP';

// Bar interval (for backtests)
type BarInterval = 'ONE_MINUTE' | 'FIVE_MINUTES';

// Chart interval (includes aggregated)
type ChartInterval = 'ONE_MINUTE' | 'FIVE_MINUTES' | 'FIFTEEN_MINUTES';

// Timeframe shorthand (for chart component)
type Timeframe = '1m' | '5m' | '15m';

// Parameter set source type
type ParameterSetSourceType = 'MANUAL' | 'LLM_OPTIMIZED';

// Alert level
type AlertLevel = 'INFO' | 'WARNING' | 'CRITICAL';

// Order status
type OrderStatus =
  | 'CREATED' | 'SUBMITTED' | 'ACCEPTED' | 'FILLED'
  | 'PARTIALLY_FILLED' | 'CANCELLED' | 'REJECTED';
```

### 7.2 Pagination

```typescript
// Spring Data Page wrapper
type Page<T> = {
  content: T[];
  totalElements: number;
  totalPages: number;
  number: number;              // Current page (0-based)
  size: number;                // Page size
  first: boolean;
  last: boolean;
  empty: boolean;
  numberOfElements: number;    // Elements in current page
};

// Pageable request parameters
type PageRequest = {
  page?: number;               // Default: 0
  size?: number;               // Default: 20, max: 100
  sort?: string;               // e.g. "tradingDate,desc"
};
```

### 7.3 Error Types

```typescript
type ApiError = {
  timestamp: string;           // ISO-8601
  status: number;
  error: string;
  message: string;
  path: string;
};

type ValidationError = ApiError & {
  violations: FieldViolation[];
};

type FieldViolation = {
  field: string;
  message: string;
};
```

### 7.4 REST DTO Types

All REST DTO types are defined inline in their respective endpoint sections in Chapters 1 and 4 of this document. For quick reference, the complete list of DTOs:

| DTO | Section | Direction |
|-----|---------|-----------|
| `TradingRunDto` | 1.1 | Response |
| `TradeDto` | 1.1 | Response |
| `DecisionLogDto` | 1.1 | Response |
| `LlmCallSummaryDto` | 1.1 | Response |
| `BacktestRequest` | 1.2 | Request |
| `BacktestStartResponse` | 1.2 | Response |
| `BacktestStatusDto` | 1.2 | Response |
| `BacktestHistoryDto` | 1.2 | Response |
| `BacktestDayResultDto` | 1.2 | Response |
| `CancelBacktestResponse` | 1.2 | Response |
| `DataAvailabilityResponse` | 1.3 | Response |
| `DataDownloadRequest` | 1.3 | Request |
| `DataDownloadResponse` | 1.3 | Response |
| `BarDto` | 1.3 | Response |
| `IndicatorDto` | 1.3 | Response |
| `ControlResponse` | 1.4 | Response |
| `HealthResponse` | 1.6 | Response |
| `OptimizationRequest` | 1.7 | Request |
| `OptimizationStartResponse` | 1.7 | Response |
| `OptimizationStatusDto` | 1.7 | Response |
| `ParameterSetDto` | 1.8 | Response |
| `CreateParameterSetRequest` | 1.8 | Request |
| `UpdateParameterSetRequest` | 1.8 | Request |
| `ExchangeSessionResponse` | 4.2 | Response (new) |
| `SymbolDto` | 4.3 | Response (new) |

---

## 8. Summary

### Endpoint Count

| Category | Existing | New | Total |
|----------|----------|-----|-------|
| Trading Run Management | 5 | 1 (`/runs/current`) | 6 |
| Backtesting | 6 | 0 | 6 |
| Data & Charts | 4 | 0 | 4 |
| Controls | 3 | 0 | 3 |
| SSE Streams | 2 | 1 (backtest progress, P2) | 3 |
| Health & System | 2 | 0 | 2 |
| Optimization | 2 | 0 | 2 |
| Parameter Sets | 5 | 0 | 5 |
| Exchange/Symbol (new) | 0 | 2 | 2 |
| **Total** | **29** | **4** | **33** |

### Change Summary

| Change Type | Count | Priority |
|------------|-------|----------|
| New endpoints | 4 | 1x P1, 3x P2 |
| SSE payload enrichment | 3 | All P1 |
| REST response enrichment | 2 | All P1 |
| Behavior change (async download) | 1 | P1 |
| DTO enrichment (BacktestDayResult) | 1 | P1 |

### Implementation Priority

**P0 (must fix):**
- `GET /api/v1/backtests/{backtestId}/report` returns same DTO as status — clarify or differentiate

**P1 (before frontend integration):**
- SSE `snapshot` payload: add bar-level fields
- SSE instrument stream: add `?timeframe` parameter
- REST bar/indicator responses: add `asOfEventId` (header or wrapper)
- SSE `trade-update`/`order-update`: ensure stable tradeId/orderId
- `GET /api/v1/runs/current` convenience endpoint
- `BacktestDayResultDto` field enrichment
- `POST /api/v1/data/download` async fix

**P2 (can follow):**
- `GET /api/v1/exchanges/{exchangeId}/session`
- `GET /api/v1/symbols/search`
- SSE backtest progress stream
