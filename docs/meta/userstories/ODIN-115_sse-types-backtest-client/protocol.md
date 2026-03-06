# ODIN-115 — Implementation Protocol

**Story:** SSE Event Types und Backtest Client Factory
**Date:** 2026-03-06
**Agent:** Worker (Claude Sonnet 4.6)
**Status:** COMPLETE

---

## 1. Scope

Frontend-only story. Extends existing files in `its-odin-ui`:

- `src/shared/types/sseEvents.ts` — 5 new discriminated union members + 6 new exported interfaces
- `src/shared/types/index.ts` — barrel export update
- `src/shared/api/sseClient.ts` — new factory function + SSE_EVENT_TYPES update
- `src/shared/api/index.ts` — barrel export update

New test files:
- `src/shared/api/sseClientFactory.test.ts` — factory input validation and URL construction tests

Existing test file extended:
- `src/shared/types/sseEvents.test.ts` — 10 new test cases for backtest event types

---

## 2. Implementation

### 2.1 New Payload Interfaces (sseEvents.ts)

| Interface | Fields | Notes |
|-----------|--------|-------|
| `BacktestBarProgressPayload` | backtestId, completedDays, totalDays, completedBars, totalBars, currentDate, estimatedRemainingMs | estimatedRemainingMs nullable |
| `BacktestDaySummaryPayload` | backtestId, tradingDate, dayPnl, cumulativePnl, totalTrades, winRate, maxDrawdown | |
| `BacktestQuantScorePayload` | backtestId, instrumentId, timestamp, overallScore, intent, activeSetup, signals | signals: ReadonlyArray<string> |
| `BacktestGateEntry` | gate, passed, reason | Named sub-type (extracted from anonymous inline) |
| `BacktestGateResultPayload` | backtestId, instrumentId, timestamp, passed, gateResults, riskRewardRatio, positionSize | riskRewardRatio/positionSize nullable; gateResults uses BacktestGateEntry |
| `StreamGapPayload` | reason, missedEventCount, lastKnownEventId | General stream control event |

### 2.2 SseEvent Union Extension

5 new members added to the discriminated union:
```
| { readonly type: 'backtest-bar-progress'; readonly payload: BacktestBarProgressPayload }
| { readonly type: 'backtest-day-summary'; readonly payload: BacktestDaySummaryPayload }
| { readonly type: 'backtest-quant-score'; readonly payload: BacktestQuantScorePayload }
| { readonly type: 'backtest-gate-result'; readonly payload: BacktestGateResultPayload }
| { readonly type: 'stream-gap'; readonly payload: StreamGapPayload }
```

### 2.3 Factory Function (sseClient.ts)

```typescript
export type BacktestDetailLevel = 'progress' | 'trading' | 'full';

export function createBacktestSseClient(
  backtestId: string,
  detailLevel: BacktestDetailLevel,
  onEvent: (event: SseEvent) => void,
): SseClient
```

URL: `/api/v1/stream/backtests/{backtestId}?detailLevel={detailLevel}`

Reuses `SseClient` class — inherits exponential backoff (10s/20s/40s max 60s) and Last-Event-ID tracking.

### 2.4 SSE_EVENT_TYPES Update

5 new strings added. The array now uses `satisfies ReadonlyArray<SseEvent['type'] | 'heartbeat' | ...>` for compile-time drift detection between the string list and the discriminated union.

---

## 3. Review Findings

### 3.1 ChatGPT Sparring (1 call)

**Key findings:**
1. HIGH: `SSE_EVENT_TYPES` and `SseEvent` union can drift silently — use `satisfies` constraint
2. HIGH: Some string fields may be nullable from backend (`activeSetup`, `gateResults[].reason`, `lastKnownEventId`)
3. MEDIUM: Anonymous inline type for `gateResults` items — extract to named interface
4. MEDIUM: URL trailing slash if `VITE_API_BASE_URL` ends with `/`
5. MEDIUM: Blank backtestId produces valid-but-wrong endpoint
6. LOW: `readonly` is compile-time only (accepted — no action needed)

### 3.2 Gemini Review — Dimension 1: Code Quality

**Key findings:**
1. Drift risk between `SSE_EVENT_TYPES` and union (confirmed ChatGPT finding)
2. String-based dates have no runtime validation
3. URL construction — trailing slash issue
4. Anonymous type for gateResults — extract to named type
5. Numeric precision: financial values as `number` (accepted as-is — standard for TS frontend)

### 3.3 Gemini Review — Dimension 2: Concept Adherence

**Key findings:**
1. All 5 interfaces match Concept 11 §3 exactly
2. stream-gap correctly placed in general union (not backtest-sub-union)
3. `createBacktestSseClient` correctly reuses `SseClient` with existing backoff/Last-Event-ID
4. detailLevel as query param is correct
5. Last-Event-ID "gap": Gemini suggested adding it as factory param — REJECTED. SseClient manages lastEventId internally on reconnect; no external initialization needed.

### 3.4 Gemini Review — Dimension 3: Praxis-Gaps (ODIN-115-rework, 2026-03-06)

Dedicated Gemini pool call (owner: ODIN-115-rework). Files reviewed: sseClient.ts, sseEvents.ts, sseEvents.test.ts, sseClientFactory.test.ts.

**Performance unter Last:**
- CRITICAL: JSON.parse + Store-Updates laufen synchron auf dem Main-Thread. Bei 100+ Events/s (Backtest-Bar-Progress) führt dies zu UI-Jank → DEFERRED: Throttling/Batching in hooks/store layer, out of scope for this story.
- MEDIUM: console.debug String-Interpolation und data.length-Berechnung bei allen Events haben Logging-Overhead → ACCEPTED as-is: Logging ist im Production-Build durch Tree-Shaking/Build-Pipeline adressierbar.

**Reconnect-Handling & EventSource-Lifecycle:**
- CRITICAL: EventListener Accumulation — beim Reconnect werden neue Listener per addEventListener hinzugefügt, aber alte nicht entfernt. Die anonymen Closures halten Referenz auf `this` → ASSESSED: Da die alte EventSource per .close() geschlossen und anschliessend null-gesetzt wird, kann der GC sie einsammeln. Kein echter Leak in der Praxis (closures referenzieren SseClient, nicht die EventSource). Kein Fix nötig.
- MEDIUM: Status Race Condition in disconnect() — connectionStatus wurde erst nach eventSource.close() gesetzt. → FIXED: Status wird jetzt VOR close() gesetzt (Zeile 133-134).

**Memory Leaks:**
- HIGH: Closure Leak in openConnection() — Loop-Closures halten `this` (SseClient) und `eventType` (String). → ASSESSED: Kein echter Leak, da EventSource korrekt null-gesetzt und Closures nur SseClient referenzieren (den Owner). SseClient lebt ohnehin so lang wie die Connection gebraucht wird.

**SSE Edge Cases:**
- MEDIUM: JSON parse errors werden geloggt aber kein Reconnect erzwungen. → ACCEPTED as-is: Partial messages führen zu JSON-Fehler → Error-Log → Stream läuft weiter. stream-gap Mechanismus ist Server-seitig zuständig. Kein erzwungener Reconnect nötig.
- LOW: lastEventId wird manuell als Query-Param angehängt (Browser schickt es auch nativ als Header beim Reconnect). → ACCEPTED as-is: Manueller Param für page-reload Recovery ist intentional und ergänzt das native Header-Verhalten.

**Browser-Kompatibilität:**
- HIGH: HTTP/1.1 Connection Limit (max 6 SSE per Domain). Bei mehreren offenen Instrument-Clients hängt das System. → ACCEPTED as-is für ODIN (privates System, Desktop-only, kontrollierte Deployment-Umgebung). HTTP/2 als Infrastructure-Voraussetzung dokumentiert unter "Offene Punkte".

**URL nicht erreichbar / Backoff:**
- MEDIUM: Backoff wurde in onopen() sofort zurückgesetzt. Bei "Flapping"-Connections (connect → sofort fail) hämmert der Client ohne Rücksicht auf den Server. → FIXED: Backoff wird nur zurückgesetzt wenn die Verbindung >= 30s stabil war (MIN_STABLE_CONNECTION_MS Konstante). Implementiert in onerror() Handler.

---

## 4. Findings Adopted

| Finding | Source | Action |
|---------|--------|--------|
| Use `satisfies` for SSE_EVENT_TYPES | ChatGPT + Gemini D1 | Implemented: `satisfies ReadonlyArray<SseEvent['type'] | 'heartbeat' | ...>` |
| Extract anonymous gate entry type | ChatGPT + Gemini D1 | Implemented: `BacktestGateEntry` interface |
| Guard against blank backtestId | ChatGPT | Implemented: throws `Error` if `backtestId.trim().length === 0` |
| Normalize trailing slash in baseUrl | ChatGPT + Gemini D1 | Implemented: `.replace(/\/$/, '')` on baseUrl |
| Disconnect status race condition | Gemini D3 | Fixed: status set BEFORE eventSource.close() in disconnect() |
| Flapping connection backoff reset | Gemini D3 | Fixed: backoff only resets after MIN_STABLE_CONNECTION_MS (30s) stable connection |

## 5. Findings Rejected / Deferred

| Finding | Reason |
|---------|--------|
| Make `activeSetup`/`reason`/`lastKnownEventId` nullable | Backend contract defines these as required strings per Concept 11 spec |
| Add `lastEventId` param to factory | SseClient manages this internally; adding it would change SseClient API (out of scope) |
| Terminal event auto-disconnect | Explicitly out of scope — ODIN-123 |
| Throttle progress events at 100+/s | Batching in hooks/store layer, out of scope for this story |
| Runtime type validation (Zod) | Out of scope; adds dependency not in project |
| Branded date/timestamp types | Low value for this story; can be considered globally if needed |
| EventListener named-function refactor | No real memory leak: EventSource is closed + nulled, GC collects closures |
| Force reconnect on JSON parse error | Stream continues; server-side stream-gap mechanism handles recovery |

---

## 6. Offene Punkte

| Punkt | Priorität | Story |
|-------|-----------|-------|
| Throttling/Batching für High-Frequency Backtest-Events (100+/s) | MEDIUM | ODIN-123 oder dedizierter Store-Story |
| Terminal-Event (completed/failed/cancelled) löst kein auto-disconnect aus — Reconnect-Loop nach Backtest-Ende | HIGH | ODIN-123 |
| HTTP/2 als Deployment-Voraussetzung für Multi-Instrument SSE (HTTP/1.1 Limit: 6 Connections/Domain) | MEDIUM | Infra/Deployment |

---

## 7. Test Results

**Build:** `tsc --noEmit` — GREEN (no errors)
**Tests:** 326 passed / 0 failed (23 test files)

New/modified tests:
- `sseEvents.test.ts`: 10 new test cases (total: 20 tests in file)
- `sseClientFactory.test.ts`: 6 new test cases

Code changes from D3 Rework: sseClient.ts (disconnect status ordering, MIN_STABLE_CONNECTION_MS backoff guard)

---

## 8. Files Changed

### its-odin-ui
- `src/shared/types/sseEvents.ts` — Added 6 interfaces, 5 union members
- `src/shared/types/index.ts` — Added 6 new type exports
- `src/shared/api/sseClient.ts` — Added `BacktestDetailLevel`, `createBacktestSseClient`, updated `SSE_EVENT_TYPES` with `satisfies`; D3 rework: disconnect status ordering, MIN_STABLE_CONNECTION_MS backoff guard
- `src/shared/api/index.ts` — Added `createBacktestSseClient`, `BacktestDetailLevel` exports
- `src/shared/types/sseEvents.test.ts` — 10 new test cases
- `src/shared/api/sseClientFactory.test.ts` — New file, 6 test cases

### its-odin-wiki
- `docs/meta/userstories/ODIN-115_sse-types-backtest-client/protocol.md` — This file
