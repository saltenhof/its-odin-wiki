# Working State — ODIN

Stand: 2026-02-19

---

## Aktueller Stand

### Architekturkonzept: FERTIG (v0.1–v0.7)

Alle 11 Architekturkapitel sind geschrieben, reviewed (ChatGPT R1–R4), und durch zwei Cross-Kapitel-Gesamtreviews validiert. FINAL GO.

| Kapitel | Version | Status |
|---------|---------|--------|
| 0 — Gesamtuebersicht | v0.7 | FERTIG (3 Iter. + Cross-Review + DDD-Modulschnitt) |
| 1 — Modularchitektur | v0.5 | FERTIG (3 Iter. + DDD-Modulschnitt) |
| 2 — Echtzeit-Datenpipeline | v0.4 | FERTIG (3 Iter.) |
| 3 — Broker-Anbindung | v0.3 | FERTIG (3 Iter.) |
| 4 — KPI-Engine | v0.3 | FERTIG (3 Iter.) |
| 5 — LLM-Integration | v0.3 | FERTIG (3 Iter.) |
| 6 — Rules Engine | v0.3 | FERTIG (2 Iter.) |
| 7 — OMS | v0.3 | FERTIG (4 Iter.) |
| 8 — Datenmodell | v0.6 | FERTIG (3 Iter. + Cross-Review R1+R2 + DDD-Modulschnitt) |
| 9 — Frontend | v0.3 | FERTIG (2 Iter.) |
| 10 — Deployment | v0.6 | FERTIG (3 Iter. + Cross-Review R1+R2 + DDD-Modulschnitt) |
| Guardrail Module Structure | v1.2 | FERTIG |
| Guardrail Development Process | v1.2 | FERTIG |
| Guardrail Frontend | v1.0 | FERTIG |

### Web Application Transformation: FERTIG (WP-01 bis WP-10)

Alle 10 Work Packages aus dem Anforderungskatalog (`frontend/requirements/web-application-requirements.md`) sind implementiert, ChatGPT-reviewed (wo moeglich), getestet und committed.

**Kennzahlen:**
- ~9.500 Zeilen neuer Code
- ~170 neue Tests
- 4 Flyway-Migrationen (V015–V018)
- 25+ REST-Endpoints
- 7 Commits auf `main`

#### Work Package Status

| WP | Titel | Groesse | Status | Commit | ChatGPT Review |
|----|-------|---------|--------|--------|---------------|
| WP-01 | Web Foundation | S | FERTIG | `06eeb64` | R1 — 4 Findings umgesetzt |
| WP-03 | TradingRun & Trade API | M | FERTIG | `f317e47` | R1 (Phase 2 combined) |
| WP-04 | Chart Data API Bars | M | FERTIG | `f317e47` | R1 — Validation Bug gefunden + behoben |
| WP-07 | DecisionLog & LLM-History | S | FERTIG | `f317e47` | R1 (Phase 2 combined) |
| WP-02 | Backtest Integration | L | FERTIG | `8116128` | Pool-Fehler, ohne Review |
| WP-05 | Backtest-Historie | M | FERTIG | `f15cfdd` | — |
| WP-06 | KPI Erweiterung | L | FERTIG | `d0626f7` | — |
| WP-10 | API-Versionierung | S | FERTIG | `d0626f7` | — |
| WP-08 | SSE Live Monitoring | XL | FERTIG | `0ada628` | R1 — 9 Findings umgesetzt |
| WP-09 | LLM Optimization Loop | XL | FERTIG | `0ada628` | R1 — 7 Findings umgesetzt |

#### Review-Fixes Commit

| Commit | Beschreibung |
|--------|-------------|
| `65481e6` | ChatGPT Phase-2-Review: ChartController Validation Bug, Pagination Tiebreaker JavaDoc |

#### Implementierte REST-Endpoints

| Methode | Endpoint | Modul | WP |
|---------|----------|-------|-----|
| GET | `/api/v1/health` | odin-app | WP-10 |
| GET | `/api/v1/instruments` | odin-app | WP-10 |
| POST | `/api/v1/runs/simulation` | odin-app | WP-10 |
| GET | `/api/v1/runs` | odin-app | WP-03 |
| GET | `/api/v1/runs/{runId}` | odin-app | WP-03 |
| GET | `/api/v1/runs/{runId}/trades` | odin-app | WP-03 |
| GET | `/api/v1/runs/{runId}/decisions` | odin-app | WP-03 |
| GET | `/api/v1/runs/{runId}/llm-history` | odin-app | WP-07 |
| GET | `/api/v1/charts/{symbol}/bars` | odin-app | WP-04 |
| GET | `/api/v1/charts/{symbol}/indicators` | odin-app | WP-06 |
| POST | `/api/v1/backtests` | odin-app | WP-02 |
| GET | `/api/v1/backtests/{id}` | odin-app | WP-02 |
| GET | `/api/v1/backtests/{id}/report` | odin-app | WP-02 |
| GET | `/api/v1/backtests` | odin-app | WP-05 |
| POST | `/api/v1/backtests/{id}/cancel` | odin-app | WP-05 |
| GET | `/api/v1/backtests/{id}/days` | odin-app | WP-05 |
| GET | `/api/v1/data/availability` | odin-app | WP-05 |
| POST | `/api/v1/data/download` | odin-app | WP-05 |
| POST | `/api/v1/backtests/{id}/optimize` | odin-app | WP-09 |
| GET | `/api/v1/optimizations/{id}` | odin-app | WP-09 |
| GET | `/api/v1/parameter-sets` | odin-app | WP-09 |
| POST | `/api/v1/parameter-sets` | odin-app | WP-09 |
| PUT | `/api/v1/parameter-sets/{id}` | odin-app | WP-09 |
| DELETE | `/api/v1/parameter-sets/{id}` | odin-app | WP-09 |
| GET | `/api/v1/stream/instruments/{id}` | odin-app | WP-08 |
| GET | `/api/v1/stream/global` | odin-app | WP-08 |
| POST | `/api/v1/controls/kill` | odin-app | WP-08 |
| POST | `/api/v1/controls/pause/{id}` | odin-app | WP-08 |
| POST | `/api/v1/controls/resume/{id}` | odin-app | WP-08 |

#### Flyway-Migrationen

| Migration | Inhalt | WP |
|-----------|--------|-----|
| V015 | `backtest_run` Tabelle | WP-02 |
| V015_1 | `daily_results_json` Spalte in backtest_run | WP-05 |
| V016 | `indicator_snapshot` Tabelle | WP-06 |
| V017 | `optimization_run` Tabelle | WP-09 |
| V018 | `parameter_set` Tabelle | WP-09 |

#### Wichtige Architekturentscheidungen (WP-Phase)

- **odin-backtest als Library:** `@Profile("cli")` auf BacktestApplication, spring-boot-maven-plugin entfernt. odin-app integriert den BacktestRunner direkt
- **Async Backtest Execution:** Dedizierter ThreadPoolTaskExecutor, `@Async`, 202 Accepted Pattern
- **Dual-Mode Indicators:** Persisted (IndicatorSnapshotEntity) vs. On-the-fly (IndicatorCalculationService)
- **SSE mit Ring Buffer:** SseEmitterManager + SseRingBuffer fuer Reconnect-Replay (60s, konfigurierbar)
- **PipelineStateMachine HALTED:** Neuer State fuer Pause/Resume, nur ueber `pause()`/`resume()` erreichbar
- **ParameterOverrideApplier:** Flat JSON Keys → immutable Record-Konstruktoren (kein Reflection)
- **JPQL Constructor Projections:** LLM-Payload-Felder in Listenabfragen ausgeschlossen (LlmCallSummaryProjection)
- **Multi-Path API Versioning:** `@GetMapping({"/api/v1/...", "/api/..."})` fuer Backward-Kompatibilitaet
- **Jackson2ObjectMapperBuilderCustomizer:** Statt eigenem ObjectMapper-Bean (ChatGPT Review Finding)
- **rejectedValue entfernt:** Sicherheitsrisiko bei Validation-Error-Responses (ChatGPT Review Finding)

### Bekannte offene Punkte

- **8 pre-existing Test-Failures** in `LlmAnalystOrchestratorTest` (odin-brain) — nicht WP-bezogen, vorher bereits vorhanden
- **Unstaged Dateien** aus frueheren Sessions: `MarketOrderJustification.java`, `V014__fix_exit_reason_check_constraint.sql`, Aenderungen in `LlmAnalystOrchestrator.java`, `PromptBuilder.java`, `OdinEWrapper.java`, `OrderManagementService.java`
- **REQ-CHART-004** (Multi-Timeframe 15m): FERTIG — BarAggregationService unterstuetzt 5m→15m, 1m→5m, 1m→15m. ChartController mit Fallback-Kaskade
- **REQ-KPI-003** (Anchored VWAP): FERTIG — AnchoredVwapCalculator (odin-data), IndicatorSnapshotEntity + V020 Migration (anchored_vwap Spalte), On-the-fly AVWAP-Berechnung (VwapAccumulator in IndicatorCalculationService), IndicatorDto-Mapping. ChatGPT-reviewed
- **REQ-OPT-006** (Iterative Optimierung): Grundstruktur da (parentOptimizationId), Auto-Chain nicht implementiert
- **REQ-LIVE-004** (Live-Bars in DB): P2, nicht implementiert

### Uebergreifende Architekturentscheidungen (aus Konzeptphase)

- Kill-Switch-Ownership: odin-data eskaliert, odin-core entscheidet
- Regime-Bestimmung: KPI deterministisch (primaer), LLM sekundaer
- Risk-Limits: Global ueber alle Pipelines (AccountRiskState in odin-core)
- VWAP: Source-of-Truth ausschliesslich in odin-data
- Crash-Recovery: GTC-Stops beim Broker + Startup-Reconciliation + Safe-Mode
- Decision Loop: Single-threaded pro Pipeline, KPI vor Rules
- DDD-Modulschnitt: Entities in Domaenen-Modulen, odin-persistence = reine Infrastruktur
- Drain-Mechanismus: Zwei Schreibpfade — direkte Domain-Persistierung + AuditEventDrainer fuer EventRecord
- Transaktionsgrenzen: Pro-Pipeline-POJOs nutzen TransactionTemplate statt @Transactional
- Frontend-Kommunikation: SSE (Monitoring) + REST POST (Controls) — kein WebSocket

---

## Frontend-Durchentwicklung

React-Frontend in `its-odin-ui/`: Dashboard, Backtesting, Trading Operations, Chart-Komponente.

| Phase | Status | Agent | Ergebnis |
|-------|--------|-------|----------|
| Phase 0: Bootstrap | FERTIG | Sub-Agent | `441104e` — 49 Dateien, Vite+React+TS, Design System, Routing, Stores, SSE/REST Clients |
| Phase 1: API-Design | FERTIG | Sub-Agent | `80d3c40` — api-interface-spec.md: 29 Endpoints inventarisiert, 4 neue, 7 Aenderungen |
| Phase 2A: Shared + Dashboard | FERTIG | Sub-Agent | `b9bed08` — 10 Shared Components, Dashboard Domain, SSE Integration |
| Phase 2B: Backtesting-Domaene | FERTIG | Sub-Agent | `1d6e9fc` — 41 Dateien, 4 Pages, 11 Components, Store, Mocks |
| Phase 2C: Trading-Operations | FERTIG | Sub-Agent | `c38819d` — 46 Dateien, 5 Pages, 12 Components, SSE Streams |
| Phase 2D: Backend-API-Anpassungen | FERTIG | Sub-Agent | `7d6d737` — 4 neue + 4 geaenderte Endpoints, 193 Tests pass |
| Phase 3: Chart-Komponente | FERTIG | Sub-Agent | `b9bed08` — 30 Dateien, Multi-Panel, 15 Layers, TradingView LWC v4 |
| Phase 4: Integration + Smoke-Test | FERTIG | Sub-Agent | `5d68983` — Chart-Integration, Token-Fix, 196 Module, 0 Fehler |
| Phase 5: Qualitaetssicherung | FERTIG | 2 QA-Agents | `11b1ba9` — 7 Bugs, 10 Arch-Violations gefixt, 197 Module, 0 Fehler |

### End-to-End-Betrieb: FERTIG

Frontend und Backend laufen End-to-End gegen die echte Datenbank.

| Komponente | URL | Details |
|-----------|-----|---------|
| Frontend (Vite Dev) | `http://localhost:3000` | React 18, HMR, Dark Theme |
| Backend (Spring Boot) | `http://localhost:3300` | SIMULATION-Modus, PostgreSQL, Flyway V001–V018 |

**Infrastruktur:**
- `tools/start-backend.sh` — Startet Backend (Port 3300), wartet auf Health-Check, speichert PID
- `tools/stop-backend.sh` — Graceful Shutdown via `POST /actuator/shutdown`, Fallback: PID-Kill
- Spring Boot Actuator: `health` + `shutdown` Endpoints exponiert
- CORS: `localhost:3000` erlaubt (GET, POST, PUT, DELETE, OPTIONS + Last-Event-ID Header)
- DB-Credentials: `ODIN_DB_USER` / `ODIN_DB_PASSWORD` (Defaults in start-backend.sh)
- Mock-Daten-Fallback via `VITE_USE_MOCKS=false` (default) deaktiviert
- SSE-URL-Bug gefixt (instrument → instruments)

**Commits (noch nicht gepusht):**
- Backend: Actuator-Dependency, Port 3300, CORS-Erweiterung, Start/Stop-Skripte
- Frontend: `.env.development` (API-URL 3300, Mocks off), Mock-Gating in allen Hooks/Stores

### Offene Punkte

- **8 pre-existing Test-Failures** in `LlmAnalystOrchestratorTest` (odin-brain)
- **REQ-OPT-006** (Iterative Optimierung): Auto-Chain nicht implementiert
- **REQ-LIVE-004** (Live-Bars in DB): P2, nicht implementiert
