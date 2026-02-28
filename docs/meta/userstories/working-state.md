# ODIN User Story Working State

Letzte Aktualisierung: 2026-02-24 (Session 4)

## Statistik
- Phase 1 (ODIN-001–045): 45/45 Done ✅
- ODIN-051 (Agent E2E Infra): Done ✅
- ODIN-FIX-001 (Backend Error Cleanup): Done ✅
- ODIN-052 (Backtest E2E Pipeline Fix): Done ✅
- ODIN-053 (Backtest Event Log UI): Done ✅
- ODIN-054 (Chart Trade Annotations): Done ✅
- Offen: 0
- In Arbeit: 0

---

## Phase 1 Done (45 Stories)

| ID | Titel |
|----|-------|
| ODIN-001 | Subregime-Modell im API-Modul |
| ODIN-002 | LLM Tactical Parameter Schema |
| ODIN-003 | Gate-Cascade-Modell im API-Modul |
| ODIN-004 | Degradation-Mode-Enum und System-Health-Model |
| ODIN-005 | Monitor-Event-Types im API-Modul |
| ODIN-006 | Cycle-Tracking-Modell im API-Modul |
| ODIN-007 | 1-Minute Monitor Event Detection |
| ODIN-008 | Session Boundary Detection und RTH-Erkennung |
| ODIN-009 | Pattern Feature Recognition |
| ODIN-010 | Subregime-Resolver im Brain-Modul |
| ODIN-011 | Gate-Kaskade im Brain-Modul |
| ODIN-012 | Setup A: Opening Consolidation FSM |
| ODIN-013 | Setup D: ReEntryCorrectionFsm |
| ODIN-014 | R-basierte Profit Protection |
| ODIN-015 | 3-Pillar Exhaustion Detection |
| ODIN-016 | TACTICAL_EXIT Mechanismus |
| ODIN-017 | LLM als Tactical Parameter Controller |
| ODIN-018 | Pre-Market Instrument Profiling |
| ODIN-019 | Brain Dual-Key Arbiter |
| ODIN-020 | Repricing Policy für Limit-Orders |
| ODIN-021 | Stop-Trailing auf 1-Minute-Bar-Close |
| ODIN-022 | Multi-Cycle OMS Handling |
| ODIN-023 | FX-Conversion in Position Sizing |
| ODIN-024 | TRAIL_ONLY Override im OMS |
| ODIN-025 | Multi-Cycle-Day Orchestration |
| ODIN-026 | DegradationManager FSM |
| ODIN-027 | EOD Forced Close Timing |
| ODIN-028 | Flash-Crash Kill-Switch |
| ODIN-029 | EOD Daily Performance Recording |
| ODIN-030 | Session Boundary Lifecycle Integration |
| ODIN-031 | Audit Hash-Chain Integrity |
| ODIN-032 | Audit Trade Intent HMAC Signing |
| ODIN-033 | Walk-Forward Validation |
| ODIN-034 | Backtest Challenger Suite |
| ODIN-035 | Backtest Governance Variants A-E |
| ODIN-036 | Enhanced Cost Model für Backtests |
| ODIN-037 | Structured SSE Event DTOs |
| ODIN-038 | Control Endpoints Enhancement |
| ODIN-039 | Daily Report Endpoint |
| ODIN-040 | Frontend Live Trading Dashboard |
| ODIN-041 | Frontend Pipeline Detail Realtime |
| ODIN-042 | Frontend Kill-Switch und Controls Integration |
| ODIN-043 | Frontend Alert Routing |
| ODIN-044 | Cycle-Tabelle (Persistence) |
| ODIN-045 | Audit Hash-Chain Schema |

---

## Infrastruktur & Fixes

| ID | Titel | Commit | Status |
|----|-------|--------|--------|
| ODIN-051 | Agent-Optimized E2E Diagnostic Infrastructure | `885a061` (its-odin-ui) | ✅ Done |
| ODIN-FIX-001 | Backend Error Cleanup (3 Endpoints) | `40998c4` (its-odin-backend) | ✅ Done |

## Session 6 — Stories + Fixes (2026-02-25)

| ID | Titel | Commits | Status |
|----|-------|---------|--------|
| ODIN-057 | Agent-Native Diagnostic Layer | `a64fef6`, `f3ee013` (backend); `2c337ec` (frontend) | ✅ Done |

### Was ODIN-057 geliefert hat
- `DiagnosticSink` Port-Interface (odin-api) + `NoOpDiagnosticSink` — Fachmodule programmieren gegen Interface
- `DiagnosticCollector` (Pro-Pipeline-POJO) + `DiagnosticRegistry` (Singleton, odin-core) — Ringbuffer, thread-safe
- Instrumentierung: `RulesEngine`, `DecisionArbiter`, `KpiEngine`, `OrderManagementService`, `TradingPipeline`
- REST-Endpoints: `GET/POST /api/v1/diagnostics/config`, `GET /api/v1/diagnostics/snapshot`, `GET /api/v1/diagnostics/trace`, `GET /api/v1/diagnostics/stream` (SSE)
- Frontend: `DiagnosticsPanel` (collapsible), VerbositySelector (OFF/MINIMAL/STANDARD/FULL), FeatureToggleCheckboxes (6), Live-SSE-TraceFeed (max 100 Einträge)
- Zero-Overhead im OFF-Modus (Supplier-Pattern)
- 296 Backend-Tests + 277 Vitest-Tests grün

## Session 6 — Fixes (2026-02-25)

| ID | Titel | Commits | Status |
|----|-------|---------|--------|
| FIX: Claude Agent SDK | ClaudeProviderClient → Subprocess zu claude.exe, kein API-Key | `349ce00` (backend) | ✅ Done |
| FIX: Null-LLM Bridge | TradingPipeline neutral pass-through + RulesEngine LLM Gate entfernt | `0283750` (backend) | ✅ Done |
| FIX: OMS 0-Trades | SimulationRunner liest zeroed PositionState, notifyCycleEnd skip-Guard | `35539e1` (backend) | ✅ Done |
| FIX: API-Key Cleanup | Alle ANTHROPIC_API_KEY-Referenzen entfernt, OptimizationAnalystClient gelöscht | `04d7824` (backend) | ✅ Done |

### Session 6 Ergebnis
- **ODIN tradet**: `totalTrades=1, totalPnl=-304.15` für IREN/2026-02-23 — erster echter Backtest mit Trades
- **Claude Agent SDK**: `claude.exe` Subprocess via `ProcessBuilder`, SSO-Authentifizierung, kein API-Key
- Tests: odin-brain 726 grün, odin-core 261 grün, odin-execution 196 grün

## Session 5 — Neue Stories (2026-02-24)

| ID | Titel | Commits | Status |
|----|-------|---------|--------|
| ODIN-055 | LLM Sampling Policy + Real-LLM Backtest | `c41ba9e` (backend) | ✅ QS PASS |
| ODIN-056 | Latency-Aware CachedAnalyst + Fast Replay | `57bd1f8` (backend) | ✅ QS PASS |

### Was ODIN-056 geliefert hat
- `CachedAnalyst` Modus B: Time-based Replay — `preloadFromRecords()` lädt LlmCallRecords, gibt Analysis nur frei wenn `responseMarketTime <= clock.now()` (korrekte Latenz-Simulation)
- `LlmCallRecordDeserializer`: defensives JSON-Deserialisieren mit Schema-Toleranz
- `BacktestRunner`: Auto-Detection REAL_LLM vs. REPLAY (all-or-nothing: alle Instrumente müssen Records haben)
- `LlmAnalyst.mayReturnNullWhenNotReady()`: verhindert DegradationManager-Eskalation bei "noch nicht bereit"
- `TradingPipeline`: null-Semantik sauber getrennt (nicht verfügbar ≠ Fehler)
- 983+ Tests grün

### Was ODIN-055 geliefert hat
- `LlmSamplingPolicy` Interface + `TimeBasedSamplingPolicy`, `EventDrivenSamplingPolicy`, `CompositeSamplingPolicy`
- `LlmCallRecord` DTO + `LlmCallRecordEntity` + `LlmCallRecordRepository` + Flyway V028
- `CachedAnalyst` Fix: Cache-Miss gibt `null` zurück (nicht mehr `UNCERTAIN/0.0`)
- `TradingPipeline.resolveLatestLlmAnalysis()` komplett überarbeitet: Sampling-Policy statt TTL, LlmCallRecord-Persistenz, null = kein Fehler
- `PipelineFactory` + `BacktestRunner`: backtestId-Propagation, LlmCallRecordRepository-Injection
- 705 Tests grün, ChatGPT-Review P0-Findings umgesetzt (SLF4J-Bug, Threshold-Validation)

---

## Session 4 — Neue Stories (2026-02-24)

| ID | Titel | Commits | Status |
|----|-------|---------|--------|
| ODIN-052 | Backtest E2E Pipeline Fix | `0b1cd49`, `afa7f03`, `739226f`, `3c7184d`, `46b0b18` (backend); `a1e8a75` (ui) | ✅ QS PASS |
| ODIN-053 | Backtest Event Log UI | `d5b8a19`, `582fea7` (backend); `f51eab0`, `baf026c`, `28fb415` (ui) | ✅ QS PASS |
| ODIN-054 | Chart Trade Annotations | `d7f58aa`, `4dc4c55` (ui) | ✅ QS PASS |

### Was ODIN-052 geliefert hat
- Root Cause 1: `BacktestController.startBacktest()` rief `executeBacktest(id, false)` → `DataDownloadService` nie aufgerufen → keine Bars → keine Ergebnisse
- Fix 1: `autoDownload=false → true` (jetzt als `DownloadMode.ENSURE_AVAILABLE` Enum)
- Root Cause 2 (Session 4, Nachtrag): `TradingPipeline.onSnapshot()` rief deprecated `DecisionArbiter.decide(LlmAnalysis)` → Überladung bridged mit `QUANT_ONLY` → alle ENTRY-Entscheidungen systemweit blockiert → 0 Trades trotz Kursdaten
- Fix 2: `@Deprecated`-Überladung vollständig entfernt; Bridge-Logik in `TradingPipeline.bridgeLegacyToTactical()` verschoben; `DecisionArbiter` hat nur noch eine `decide()`-Signatur; 9 neue `TradingPipelineTest`-Bridge-Cases
- Frontend: 0-Trade-Meldung sauber kommuniziert; Polling-Flicker-Fix
- 5 neue `BacktestPipelineIntegrationTest` (Zonky) + 9 neue `DecisionArbiterTest`-Legacy-Bridge-Cases

### Was ODIN-053 geliefert hat
- Backend: `GET /api/v1/events` (Cross-Run, 5 optionale Filter, Paginierung), `EventRecordDto` + `backtestId`/`backtestName`/`wallClockTimestamp`
- Frontend: Events-Tab in `BacktestDetailPage` (Tabelle, Event-Typ-Filter, Freitextsuche, Sort-Toggle, Payload-Viewer)
- Frontend: Globale Event-Suche in `BacktestListPage` (Cross-Run, Link zum Backtest-Detail)
- 268 Vitest-Tests grün

### Was ODIN-054 geliefert hat
- `TradingEventLayer` erweitert: ENTRY, TRANCHE_ADD, PARTIAL_EXIT, FULL_EXIT, STOP_TRAIL, STOP_TRIGGER
- `TradeMarkerTooltip`: Hover-Tooltip mit Event-Typ, Zeit, Preis, Shares, PnL
- CSS Modules, eindeutige Marker-IDs, `IntraDayChart.integration.test.ts`
- `data-testid="trade-marker-tooltip"` auf Tooltip

---

## Was ODIN-051 gebaut hat

- `src/shared/diagnostics/agentDiagnostics.ts` — DEV-only Singleton: interceptet `console.error/warn/log` + `window.fetch`, exposes `window.__odinDiagnostics` für Playwright `page.evaluate()`. HMR-safe, MAX_LOG_ENTRIES=500.
- `src/shared/diagnostics/DiagnosticsProvider.tsx` — React-Provider, injiziert Zustand-Store-Getter. In Production transparent.
- `src/shared/components/ErrorBoundary.tsx` — erweitert: `componentDidCatch` pusht in `reactErrors`-Log.
- `data-testid`-Attribute auf 12+ Komponenten: `kill-switch-button`, `confirm-dialog`, `system-mode-badge`, `alert-feed`, `alert-filter-{level}`, `sse-connection-status`, `pipeline-card`, `position-panel`, `system-health-card`, `toaster-notification`, etc.
- `e2e/helpers/diagnostics.ts` — Playwright-Helper: `collectDiagnostics()`, `resetDiagnostics()`, `expectNoErrors()`, `takeAnnotatedScreenshot()`.
- `e2e/live-dashboard.spec.ts`, `e2e/controls.spec.ts`, `e2e/alerts.spec.ts` — 11 E2E-Specs gegen live Server.

## Was ODIN-FIX-001 gefixt hat

1. `GET /api/v1/controls/mode` → 404: Endpoint existierte nicht. Fix: `SystemModeDto`-Record + `@GetMapping("/mode")` in `ControlController`.
2. `GET /api/v1/runs/current` → 500: Hibernate 6.6.8 Type-Inference-Bug bei `LocalDate`-Parameter in JPQL IS-NULL-Pattern. Fix: JPQL → Native-SQL mit `CAST(:tradingDate AS date)`, Parameter-Typen auf `String`.
3. `GET /api/v1/runs?...` → 500: Gleiche Ursache, gleicher Fix.
- Wichtig: Nach Repository-Signatur-Änderung `mvn clean install -pl odin-execution,odin-app --also-make` nötig (sonst `NoSuchMethodError` wegen altem JAR im Maven-Repo).

---

## Aktueller System-Status (Stand 2026-02-24, Session 4)

- Backend: Port 3300 (SIMULATION-Modus)
- Frontend: Port 3000 (Vite dev server)
- Vitest: **268 Tests grün** (nach ODIN-052/053/054)
- Backend Unit-Tests: **296+ Tests grün** (odin-app, +9 neue DecisionArbiter Legacy-Bridge-Tests nach ODIN-052 Nachtrag)
- Pre-Existing-Bug: `TrailingStopManagerIntegrationTest.trailingStopEventIsLoggedOnUpdate` (odin-execution) — unverändert offen
- **Gemini-Pool dauerhaft ausgefallen** (Google-Block): DoD 2.6 = ChatGPT-Review deckt alle 3 Dimensionen ab

---

## Phase 2 Stories (nicht beauftragt)

| ID | Titel | Größe |
|----|-------|-------|
| ODIN-046 | EU Market Session Boundaries | L |
| ODIN-047 | L2-Depth Data Tier | XL |
| ODIN-048 | FusionLab Evaluation | XL |
| ODIN-049 | Universe Screening | L |
| ODIN-050 | Advanced Crash Recovery | L |

---

## Bekannte offene Punkte

- `TrailingStopManagerIntegrationTest.trailingStopEventIsLoggedOnUpdate` (odin-execution) — pre-existing Fehler, Ursache unklar, nicht beauftragt
- Stufe 2 Fehler (fachliche Fehler beim Durchklicken der UI, noch nicht spezifiziert) — vom User angekündigt, noch nicht erfasst
