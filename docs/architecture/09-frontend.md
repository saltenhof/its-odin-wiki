# ODIN — Kapitel 9: React-Frontend

Version: 0.3 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2 — P0/P1 Fixes (SimClock→MarketClock, AccountRiskState-Felder, Kill-Switch-Semantik, Pause/Resume-FSM, Endpoint-Liste, SSE-Replay, Bar-Aggregation)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt das React-Frontend von ODIN: Echtzeit-Dashboard fuer Trade-Monitoring, Pipeline-State-Visualisierung, Chart-Darstellung und manuelle Controls (Kill-Switch, Pause/Resume).

**Modulzuordnung:** `odin-frontend` (separater npm-Build, kein Maven-Dependency-Graph)

**Abgrenzung:**
- Das Frontend hat **keine Geschaeftslogik** — es visualisiert Daten und leitet Benutzer-Aktionen weiter
- Datenquelle: Backend-API (odin-app, Spring Boot) — das Frontend greift nie direkt auf Persistence oder EventLog zu
- Das Frontend ist ein **Monitoring- und Control-Interface**, kein Analyse-Tool. Langzeit-Analyse erfolgt ueber direkte DB-Queries (Kap 8)

---

## 2. Architektur

```
┌──────────────────────────────────────────────────┐
│                  React Frontend                   │
│                                                  │
│  ┌────────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Chart      │  │ Pipeline │  │ Controls     │ │
│  │ (Trading-  │  │ Status   │  │ (Kill-Switch │ │
│  │  View LW   │  │ Panel    │  │  Pause/Resume│ │
│  │  Charts)   │  │          │  │  Manual Ops) │ │
│  └──────┬─────┘  └────┬─────┘  └──────┬───────┘ │
│         │             │               │          │
│         v             v               v          │
│  ┌─────────────────────────────────────────────┐ │
│  │           State Management Layer            │ │
│  └──────────────┬──────────────────────────────┘ │
│                 │                                 │
└─────────────────┼─────────────────────────────────┘
                  │
         ┌────────┴────────┐
         │                 │
    SSE (read)        REST (r/w)
         │                 │
         v                 v
┌─────────────────────────────────────┐
│          odin-app (Backend)         │
│                                     │
│  SSE Endpoints:                     │
│  /api/stream/instruments/{id}       │
│  /api/stream/global                 │
│                                     │
│  REST Endpoints:                    │
│  POST /api/controls/kill            │
│  POST /api/controls/pause/{id}      │
│  POST /api/controls/resume/{id}     │
│  GET  /api/health                   │
│  GET  /api/runs/{runId}             │
│  GET  /api/runs/{runId}/decisions   │
│  GET  /api/runs/{runId}/llm-history │
│  GET  /api/instruments              │
└─────────────────────────────────────┘
```

> **Kein WebSocket.** v0.1 hatte ein WebSocket-Endpoint fuer Controls. **Vereinfacht** gemaess Lean-Prinzip: Controls sind seltene, nicht-zeitkritische Aktionen (Kill-Switch, Pause/Resume). REST POST ist ausreichend und vermeidet die Komplexitaet von WS-Connection-Management. SSE bleibt fuer den Monitoring-Stream (unidirektional, Server→Client).

---

## 3. Kommunikationsprotokolle

### SSE (Server-Sent Events) — Monitoring-Streams

Fuer unidirektionalen Echtzeit-Datenfluss vom Backend zum Frontend:

| Endpoint | Daten | Update-Frequenz |
|----------|-------|-----------------|
| `/api/stream/instruments/{instrumentId}` | Pipeline-State, Kurs, Indikatoren, P&L, Orders, LLM-Status | Alle 1-5s (konfigurierbar) |
| `/api/stream/global` | Globaler Risk-Status (AccountRiskState), aggregierte P&L, Kill-Switch-State, Alerts | Alle 5s |

**instrumentId** als Path-Parameter (nicht pipelineId): entspricht der Pipeline-Identifier-Konvention (1:1 in v1, Kap 8 §4).

**Reconnect/Resync:**
- Jedes SSE-Event traegt eine `eventId` (monoton steigend, **pro Stream** — instrument-stream und global-stream haben unabhaengige Sequenzen). Frontend sendet `Last-Event-ID` beim Reconnect. Backend replayed verpasste Events (aus In-Memory-Ringbuffer, max. 60s)
- Bei Verbindungsabbruch: Automatischer Reconnect mit Exponential Backoff (1s/2s/4s/8s, max 30s). Nach Reconnect: Backend sendet aktuellen State-Snapshot als erstes Event
- **Nach Backend-Restart:** Ringbuffer ist leer (In-Memory). Frontend erkennt eventId-Reset (Sequenz startet bei 1) → erzwingt Full-Snapshot-Refresh. Kein Replay moeglich ueber Backend-Neustart hinweg (akzeptiertes v1-Verhalten)

**Event-Typen (SSE):**

| Event | Payload |
|-------|---------|
| `snapshot` | Preis, Bid/Ask, Spread, Volume, VWAP, RSI, ATR. Enthaelt `runId` und `instrumentId` |
| `state-change` | Pipeline-State (STARTUP, WARMUP, OBSERVING, POSITIONED, PENDING_FILL, HALTED, DAY_STOPPED — Kap 6 FSM) |
| `trade-update` | Offene Position, unrealisierte P&L, Tranchen-Status, tranchenVariant |
| `order-update` | Order-Status (CREATED, SUBMITTED, ACCEPTED, FILLED, PARTIALLY_FILLED, CANCELLED, REJECTED — Kap 7 §6 OMS Order-Lifecycle) |
| `llm-update` | Regime, regimeConfidence, requestMarketTime (kompakt, kein Freitext) |
| `alert` | Alert-Level (INFO, WARNING, CRITICAL), Message (operator-orientierter Freitext — nicht LLM-basiert), emitter |
| `pnl` | Tages-P&L (realisiert + unrealisiert), grossPnl, netPnl, Trade-Count |
| `risk-update` | remainingBudget, exposure, roundTrips, coolingOff (identisch zu AccountRiskState DTO in odin-core, Kap 0). SSE-Payload = 1:1 Abbildung des odin-core DTO, kein Frontend-Mapping |

> **MarketClock-Timestamps:** Alle Timestamps in SSE-Events sind **MarketClock-basiert** (Kap 0). Das Frontend zeigt diese direkt an — keine Konvertierung noetig, da MarketClock in Exchange-TZ arbeitet.

### REST-Endpoints

| Endpoint | Methode | Zweck |
|----------|---------|-------|
| `/api/controls/kill` | POST | Kill-Switch **Request** (odin-core entscheidet, Kap 0). Response: `{ status: "ACCEPTED" | "REJECTED" | "NOOP", reasonCode?, marketClockTimestamp }`. ACCEPTED = Request weitergeleitet an odin-core. REJECTED = z.B. bereits aktiv. NOOP = keine offenen Positionen. UI zeigt Kill-Switch-State (read-only) aus Global-Stream; Button sendet nur den Request |
| `/api/controls/pause/{instrumentId}` | POST | Pipeline pausieren → HALTED (Kap 6). Erlaubt aus: WARMUP, OBSERVING, POSITIONED. **Nicht erlaubt aus:** PENDING_FILL (Orders in-flight — erst Fill/Abandon abwarten), DAY_STOPPED (bereits beendet), STARTUP (noch nicht initialisiert). Bei unzulaessigem State: HTTP 409 mit `{ currentState, allowedStates }` |
| `/api/controls/resume/{instrumentId}` | POST | Pipeline fortsetzen → State vor HALTED. **Nicht erlaubt aus:** DAY_STOPPED (kein Resume moeglich). Bei globalem Kill-Switch aktiv: HTTP 409 (Controls deaktiviert). Bei unzulaessigem State: HTTP 409 mit `{ currentState, reason }` |
| `/api/health` | GET | Health-Check (Spring Actuator) |
| `/api/instruments` | GET | Liste aktiver Instrumente mit aktuellem Pipeline-State |
| `/api/runs` | GET | TradingRun-Liste mit Filtern: `?mode=LIVE|SIMULATION`, `?tradingDate=2026-01-15`, `?instrumentId=AAPL`, `?batchId=backtest-2026Q1`, `?status=ACTIVE|COMPLETED`. Paginiert |
| `/api/runs/current` | GET | Kurzform fuer aktuelle TradingRuns (heute, mode=LIVE, status=ACTIVE) |
| `/api/runs/{runId}` | GET | TradingRun-Details (Kap 8 §4) |
| `/api/runs/{runId}/trades` | GET | Trades des Runs |
| `/api/runs/{runId}/decisions` | GET | DecisionLog-Eintraege (paginiert) |
| `/api/runs/{runId}/llm-history` | GET | LlmCall-Historie (paginiert, ohne inputPayload/outputPayload — zu gross) |

> **Streaming-Payload-Policy:** SSE-Events enthalten nur kompakte, entscheidungsrelevante Delta-Updates. Groessere Daten (LLM-Historie, DecisionLog, Chart-History) werden per REST on-demand abgerufen.

### Authentifizierung und Autorisierung

| Aspekt | Regelung |
|--------|---------|
| AuthN | HTTP Basic Auth (v1) ueber HTTPS. Credentials konfigurierbar (ENV-Variable). Kein offener Zugang |
| AuthZ | Rolle `OPERATOR` fuer Controls (Kill-Switch, Pause/Resume). Rolle `VIEWER` fuer Monitoring-Streams und Reports |
| CSRF | REST-Mutations-Endpoints (POST /api/controls/*) geschuetzt via Spring Security CSRF-Token |
| Localhost-Scope | v1: Binding auf localhost (nicht extern erreichbar). Produktionsbetrieb nur lokal |

---

## 4. Frontend-Komponenten

### 4.1 Chart-Ansicht

| Feature | Bibliothek | Details |
|---------|-----------|---------|
| Candlestick-Chart | TradingView LW Charts | OHLC + Timeframe-Wechsel (1m, 5m, 15m). **Bar-Aggregation erfolgt backendseitig** (odin-data, Kap 2); Frontend rendert nur fertige Bars. Initiales Laden via REST (`/api/runs/{runId}/bars?tf=5m`), Live-Updates via SSE `snapshot` (last-price + aktuelle Bar-Werte). Scrollback per REST-Paging |
| Volumen-Overlay | TradingView LW Charts | Volumen-Balken unter dem Chart |
| Indikator-Overlay | TradingView LW Charts | VWAP, EMA(9), EMA(21), Bollinger Bands (Kap 4 KPI-Engine) |
| Entry/Exit-Marker | TradingView LW Charts | Marker auf dem Chart bei Trades (aus `trade-update` Events) |
| P&L-Verlauf | Recharts | Intraday-P&L-Kurve (netto nach Kosten, Kap 7 §10) |
| Quant-Score-History | Recharts | Score-Verlauf ueber den Tag |

### 4.2 Pipeline-Status-Panel

Pro Instrument/Pipeline eine Statusanzeige:

| Element | Inhalt |
|---------|--------|
| Pipeline-State | Farbcodiert: STARTUP=grau, WARMUP=gelb, OBSERVING=blau, POSITIONED=gruen, PENDING_FILL=orange, HALTED=rot, DAY_STOPPED=grau dunkel. Alle FSM-States (Kap 6) dargestellt |
| Instrument | Symbol + aktueller Preis + Tagesaenderung |
| Position | Stueckzahl, Entry-Preis, unrealisierte P&L, Stop-Level, tranchenVariant |
| LLM-Status | Regime, regimeConfidence, letzter requestMarketTime, Provider+Model |
| Tranchen-Uebersicht | Welche Tranchen offen/gefuellt, Target-Level je Tranche |
| OMS-State | NO_POSITION, ENTRY_PENDING, POSITIONED, EXIT_PENDING (Kap 7 §6) |

### 4.3 Global-Dashboard

| Element | Inhalt |
|---------|--------|
| Aggregierte P&L | Summe ueber alle Pipelines (netPnl = grossPnl - totalCost) |
| Exposure | Gesamtes offenes Exposure in % des Tageskapitals (aus AccountRiskState) |
| Trade-Count | Heute ausgefuehrte Trades (aller Pipelines) |
| Risk-Status | remainingBudget, Abstand zum Hard-Stop, Cooling-Off-Status |
| Kill-Switch-Button | Gut sichtbar, mit Bestaetigungs-Dialog. POST `/api/controls/kill` |
| System-Alerts | Letzte N Alerts, farbcodiert nach Level |

### 4.4 Alert-Darstellung (Frontend)

| Alert-Level | UI-Darstellung |
|-------------|---------------|
| INFO | Grauer Banner, automatisch ausblendend (5s) |
| WARNING | Gelber Banner, bleibt sichtbar bis Acknowledge |
| CRITICAL | Roter Banner + Sound-Alert, manuelles Acknowledge |

> **Kein EMERGENCY-Level.** v0.1 hatte ein "Fullscreen-Overlay, Sound, blinkend" EMERGENCY-Level. **Vereinfacht:** Kritische Situationen (Kill-Switch) werden ueber CRITICAL-Alerts plus die Kill-Switch-UI abgedeckt. Ein separates Fullscreen-Overlay ist Over-Engineering fuer v1.

### 4.5 LLM-Analyse-Historie

Abruf per REST (`GET /api/runs/{runId}/llm-history?limit=20`), nicht ueber SSE-Stream (zu gross).

Scrollbare Liste der letzten LLM-Analysen mit:
- requestMarketTime, Regime, regimeConfidence
- validationResult (VALID / SCHEMA_ERROR / PLAUSIBILITY_ERROR / CONSISTENCY_ERROR — Kap 5)
- Provider + Model

> **Kein Freitext im Frontend.** LLM-Reasoning wird nicht im UI angezeigt (Analyst-Prinzip, Kap 5 §2: LLM liefert strukturierte Daten, kein Freitext fuer Entscheidungen). Die rohen Payloads sind per DB-Query abrufbar (Kap 8, LlmCall.outputPayload).

---

## 5. Simulation-Semantik

### Frontend im Sim-Modus

Das Frontend kann sowohl Live- als auch Sim-Runs anzeigen. Unterschiede:

| Aspekt | Live | Simulation |
|--------|------|------------|
| SSE-Streams | Echtzeit-Updates | Beschleunigte Updates (MarketClock simuliert, vom Runner gesteuert). SSE-Events enthalten `mode: SIMULATION` |
| Controls | Kill-Switch, Pause/Resume aktiv | Kill-Switch deaktiviert (Sim hat keinen Broker). Pause/Resume steuern den SimulationRunner |
| runId | Aktueller Live-Run | Selektierbarer Sim-Run (via REST `/api/runs?mode=SIMULATION&batchId=...`) |
| Timestamps | MarketClock (Echtzeit) | MarketClock (simulierte Zeit, vom Runner gesteuert — es gibt keine separate „SimClock", MarketClock bleibt einzige Zeitquelle, Kap 0) |
| Anzeige-Indikator | Kein spezieller Indikator | Banner "SIMULATION MODE" im Header, farblich abgesetzt |

### Sim-Run-Auswahl

Fuer Sim-Replay und -Analyse:
- `GET /api/runs?mode=SIMULATION&tradingDate=2026-01-15&instrumentId=AAPL` → Liste von Sim-Runs
- `GET /api/runs?batchId=backtest-2026Q1` → Alle Runs einer Backtest-Serie
- Frontend kann zwischen Runs wechseln und historische Daten anzeigen (Charts, Decisions, Trades)

---

## 6. Build und Deployment

### Build-Prozess

- **Framework:** React 18+ mit TypeScript
- **Build-Tool:** Vite (schneller Build, HMR fuer Entwicklung)
- **Ergebnis:** Statische Dateien (HTML, JS, CSS)
- **Deployment:** Statische Dateien werden von odin-app (Spring Boot) ausgeliefert (`src/main/resources/static/`)

### Entwicklungsworkflow

| Modus | Setup |
|-------|-------|
| Entwicklung | `npm run dev` (Vite Dev Server, Port 3000) mit Proxy auf odin-app (Port 8080) |
| Produktion | `npm run build` → Output in `odin-app/src/main/resources/static/` |

> **Maven-Integration:** Der Frontend-Build wird ueber `frontend-maven-plugin` in den Maven-Build integriert. `mvn package` baut automatisch das Frontend mit.

---

## 7. Konfiguration

```properties
# odin-frontend.properties (Backend-seitige Konfiguration)

# SSE
odin.frontend.sse.instrument-update-interval-ms=1000
odin.frontend.sse.global-update-interval-ms=5000
odin.frontend.sse.replay-buffer-seconds=60

# Auth
odin.frontend.auth.username=${ODIN_UI_USER}
odin.frontend.auth.password=${ODIN_UI_PASSWORD}

# Chart
odin.frontend.chart.default-timeframe=5m
odin.frontend.chart.max-bars-visible=200
```

---

## 8. Abhaengigkeiten und Schnittstellen

### Konsumiert (Backend-API von odin-app)

- SSE-Streams: Pipeline-/Global-Updates (aufbereitet aus EventLog + in-memory State)
- REST-Endpoints: TradingRun, Trade, DecisionLog, LlmCall Queries (odin-persistence, Kap 8)
- AccountRiskState (odin-core): fuer Global-Dashboard Risk-Status
- MarketClock (odin-api): Timestamps in allen Events

### Bereitgestellt (an den Benutzer)

- Echtzeit-Monitoring (SSE-basiert)
- Manuelle Controls (REST-basiert): Kill-Switch, Pause/Resume
- Historische Analyse (REST + Sim-Run-Auswahl)

### Nicht in Scope

- Geschaeftslogik (Rules, OMS, Risk Gate) → Kap 6, 7
- Datenmodell/Persistence → Kap 8
- LLM-Integration → Kap 5
- Broker-Anbindung → Kap 3
