# ODIN-123: Reconnect und Chart-Recovery nach SSE-Unterbrechung — Protokoll

**Status**: Done
**Commit**: `2f02e93` (`feat(ODIN-123): reconnect and chart recovery after SSE interruption`)
**Repo**: `its-odin-ui`
**Datum**: 2026-03-06

---

## Umsetzung

### Geaenderte Dateien (7)

| Datei | Aenderung |
|---|---|
| `src/shared/components/IntraDayChart/hooks/useChartLiveUpdates.ts` | `triggerRecovery()`, `isRestoringRef`, `reconnectBufferRef`, AbortController, epoch-sync, bounded buffer (1000 cap), rAF-Cancellation bei Recovery-Start, Unmount-Status-Restore |
| `src/shared/components/IntraDayChart/hooks/useChartRecovery.test.ts` | **NEU** — 11 Tests: Buffer, Flush, Epoch-Gating, Double-Reconnect, Failure-Handling, ConnectionStatus-Transitions, Unmount-during-Recovery |
| `src/domains/trading-operations/hooks/usePipelineStream.ts` | `stream-gap`-Handling, `onReconnect`-Callback, SSE-Reconnect-Erkennung (reconnecting->connected) |
| `src/domains/trading-operations/hooks/usePipelineStream.streamGap.test.ts` | **NEU** — 6 Tests: stream-gap, SSE-Reconnect, Initial-Connection, Non-Gap-Events |
| `src/shared/components/ConnectionBanner.tsx` | `restoring`-Status in Banner-Logik |
| `src/shared/components/StatusBadge.tsx` | `restoring` in STATUS_CLASS_MAP und ACTIVE_STATUSES |
| `src/shared/types/common.ts` | `'restoring'` zu `ConnectionStatus` Union-Type |

### Recovery-Flow (Konzept 12 SS9.2)

```
triggerRecovery()
  1. Abort in-flight recovery (double-reconnect)
  2. Cancel pending rAF + clear pendingMap
  3. isRestoring=true, buffer=[]
  4. setConnectionStatus('restoring')
  5. await refreshFromRest(signal)
  6. if aborted -> return (new recovery takes over)
  7. if null -> resume normal flow
  8. isRestoring=false
  9. epochRef.current++ (sync, vor React-Render)
  10. Flush buffer through enqueueUpdate (epoch-gating)
  11. setConnectionStatus('connected')
```

### Kritische Fixes (aus ChatGPT-Review)

1. **rAF-Cancellation bei Recovery-Start**: Pending `requestAnimationFrame` und `pendingMap` werden zu Beginn von `triggerRecovery()` gecancelled/gecleared. Verhindert dass pre-recovery Updates waehrend der Recovery angewendet werden.

2. **Epoch-Race-Fix**: Nach `refreshFromRest` (das `setChartEpoch(prev+1)` aufruft) wird `epochRef.current` synchron inkrementiert, bevor der Buffer geflusht wird. React hat zu diesem Zeitpunkt noch nicht re-gerendert, daher waere `epochRef` sonst stale.

3. **Unmount-Status-Restore**: Cleanup-Effect prueft `isRestoringRef.current` und setzt `connectionStatus` zurueck auf `'connected'`, falls Recovery waehrend Unmount lief.

4. **Bounded Reconnect Buffer**: Buffer-Cap bei 1000 Events mit FIFO-Eviction. Schuetzt gegen pathologische Netzwerk-Stalls.

---

## Qualitaetssicherung

### Build & Tests

- `tsc --noEmit`: PASS (0 Errors)
- `npm run build`: PASS (2.42s)
- `npx vitest run`: **521 Tests PASS** (17 neue Tests)

### LLM-Sparring

#### ChatGPT (1 Aufruf)

| Finding | Schwere | Status |
|---|---|---|
| Epoch-Race: epochRef.current stale nach async setChartEpoch | Kritisch | Behoben (sync epochRef++) |
| rAF nicht gecancellt bei Recovery-Start | Kritisch | Behoben (cancelAnimationFrame + clear) |
| Unmount laesst 'restoring' Status | Kritisch | Behoben (cleanup restores connected) |
| connectionStatus dual responsibility | Mittel | Akzeptiert (kein Produktions-Impact) |
| Reconnect buffer unbounded | Niedrig | Behoben (1000-Event-Cap) |
| Double stream-gap thrash | Mittel | Akzeptiert (natuerlich durch AbortController gehandhabt) |

#### Gemini Dimension 1 — Code Quality

| Aspekt | Rating |
|---|---|
| TypeScript Strictness | 5/5 |
| Ref Usage Patterns | 5/5 |
| Cleanup Correctness | 5/5 |
| rAF Batching | 5/5 |
| Callback Stability | 4.5/5 |
| React Best Practices | 5/5 |

Einzige Anmerkung: `refreshFromRest` in `triggerRecovery` useCallback-Deps -- wenn Caller unmemoized inline-Funktion uebergibt, wird `triggerRecovery` bei jedem Render neu erstellt. Verantwortung liegt beim Caller.

#### Gemini Dimension 2 — Konzepttreue

| Konzept-Sektion | Rating |
|---|---|
| SS9.1 chartEpoch | 5/5 |
| SS9.2 Recovery Flow | 5/5 |
| SS9.3 stream-gap | 5/5 |
| SS9.4 SSE Reconnect | 5/5 |

Keine Abweichungen vom Konzept. Erweiterungen (manuelle Epoch-Advancement, StrictMode-Handling) verbessern das Konzept ohne es zu verletzen.

#### Gemini Dimension 3 — Praxis (Edge Cases)

| Edge Case | Rating |
|---|---|
| Double-Reconnect | 5/5 |
| Stale Epochs | 5/5 |
| Buffer Overflow | 3.5/5 -> 5/5 (nach Fix) |
| Hidden Tab | 4.5/5 |
| Race Conditions | 5/5 |
| Strict Mode | 5/5 |

Buffer-Overflow-Finding (3.5/5) wurde direkt behoben durch 1000-Event-Cap mit FIFO-Eviction.
