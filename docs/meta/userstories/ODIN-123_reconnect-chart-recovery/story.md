## Kontext

Robuste Chart-Wiederherstellung nach SSE-Reconnect oder Stream-Gap. Wenn die SSE-Verbindung abbricht und sich neu verbindet, oder wenn das Backend ein `stream-gap` Event sendet (Ring-Buffer-Overflow), muss der Chart seinen Zustand per REST neu laden und dann nahtlos auf den SSE-Stream umschalten. Ohne diese Recovery-Logik zeigt der Chart nach einem Reconnect veraltete oder lueckenhafte Daten.

## Modul

its-odin-ui — `src/shared/components/IntraDayChart/`

## Abhaengigkeiten

#9 (ODIN-116: chartEpoch-Mechanismus)

## Scope

### In Scope
- `isRestoring` State-Flag: waehrend Reconnect werden eingehende SSE-Events in einen Buffer geschrieben statt sofort verarbeitet
- REST-Refresh: Bars + Indicators + Trades per REST neu laden (bestehende REST-Endpunkte)
- `chartEpoch` Increment bei Reconnect → `series.setData()` mit frischen REST-Daten + anschliessendes Buffer-Flush
- `stream-gap` Event Handling: wenn `stream-gap` empfangen wird → REST-Refresh triggern (gleicher Pfad wie Reconnect)
- DecisionFeed Reconnect: alte Events per REST laden, neue aus SSE appenden
- Connection-Status-Banner Integration (nutzt bestehendes `ConnectionBanner.tsx`)

### Out of Scope
- SSE-Client-Reconnect-Logik (existiert bereits in `sseClient.ts` mit Exponential Backoff)
- Neue SSE-Event-Types (ODIN-115)
- Chart-Modi oder neue Komponenten

## Akzeptanzkriterien
- [ ] `isRestoring` Flag verhindert die Verarbeitung von SSE-Events waehrend der REST-Recovery
- [ ] SSE-Events die waehrend `isRestoring=true` eintreffen werden gebuffert
- [ ] REST-Refresh laedt aktuelle Bars, Indikatoren und Trades
- [ ] Nach REST-Load: `series.setData()` mit frischen Daten, dann Buffer-Flush (gebufferte SSE-Events verarbeiten)
- [ ] `chartEpoch` wird bei Recovery inkrementiert
- [ ] Updates mit veraltetem `chartEpoch` werden verworfen
- [ ] `stream-gap` Event loest denselben Recovery-Pfad aus wie ein SSE-Reconnect
- [ ] DecisionFeed-Events werden bei Recovery per REST nachgeladen
- [ ] ConnectionBanner zeigt Reconnect-Status an ("Reconnecting...", "Restoring chart data...")
- [ ] `tsc --noEmit` kompiliert fehlerfrei
- [ ] Unit-Tests fuer Recovery-Flow, Buffer-Mechanismus und Epoch-Gating existieren

## Technische Details

### Geaenderte Dateien
- `src/shared/components/IntraDayChart/hooks/useChartLiveUpdates.ts`:
  - `isRestoring` State hinzufuegen (Ref, nicht State — kein Re-Render noetig)
  - `reconnectBuffer: ChartUpdate[]` Ref fuer gebufferte Events waehrend Recovery
  - `triggerRecovery()` Methode: setzt `isRestoring=true`, laedt REST-Daten, ruft `setData()`, flusht Buffer, setzt `isRestoring=false`
  - `enqueueUpdate()` prueft `isRestoring` — wenn true, pushed in Buffer statt rAF-Queue
  - `chartEpoch` Increment in `triggerRecovery()`
  - Neues Return-Feld: `triggerRecovery: () => Promise<void>`

- `src/shared/components/IntraDayChart/hooks/useChartData.ts`:
  - Neue Methode `refreshFromRest()`: laedt aktuelle Bars + Indikatoren + Trades per REST (wiederverwendet bestehende REST-Logik)
  - Return-Feld: `refreshFromRest: () => Promise<ChartDayData>`

- `src/domains/trading-operations/hooks/usePipelineStream.ts`:
  - `stream-gap` Event Handling: ruft `onReconnect` Callback auf
  - SSE-Reconnect-Event (`onReconnect` vom SseClient): ruft `onReconnect` Callback auf
  - Neuer optionaler `onReconnect` Parameter

- `src/domains/backtesting/hooks/useBacktestStream.ts`:
  - Analoges `stream-gap` und Reconnect-Handling wie usePipelineStream

### Recovery-Ablauf
```
SSE-Reconnect oder stream-gap empfangen
  │
  ├── isRestoring = true
  ├── Eingehende SSE-Events → reconnectBuffer
  │
  ├── REST-Refresh starten (parallel):
  │   ├── GET /api/v1/charts/{symbol}/bars → aktuelle Bars
  │   ├── GET /api/v1/charts/{symbol}/indicators → aktuelle Indikatoren
  │   └── GET /api/v1/charts/{symbol}/trades → aktuelle Trades
  │
  ├── chartEpoch++
  ├── series.setData(freshBars)
  ├── Indikator-Serien.setData(freshIndicators)
  ├── TradingEventLayer aktualisieren
  │
  ├── Buffer-Flush: reconnectBuffer.forEach(enqueueUpdate)
  │   (nur Events mit aktuellem chartEpoch)
  │
  └── isRestoring = false
```

### Patterns
- **Epoch-Gating:** Jeder `ChartUpdate` traegt (implizit oder explizit) eine Epoch-Information. Updates die vor dem Recovery-Epoch erzeugt wurden, werden beim Flush verworfen.
- **Buffer-then-Flush:** Waehrend `isRestoring=true` gehen SSE-Events in den Buffer. Nach `setData()` wird der Buffer geflusht. Events im Buffer die aelter sind als die REST-Daten werden via Timestamp-Vergleich verworfen.
- **ConnectionBanner:** Das bestehende `ConnectionBanner.tsx` zeigt den Connection-Status an. Es liest aus `globalStore.connectionStatus`. Waehrend Recovery kann ein zusaetzlicher Status "restoring" angezeigt werden.

## Konzept-Referenzen
- `docs/concept/12-live-chart-components.md` — §9: Reconnect-Strategie, §9.2: chartEpoch-Mechanismus, §4.6: Konsistenz bei Mid-Stream-Connect
- `docs/concept/11-live-data-streaming.md` — §7.5: Stream-Gap-Signalisierung (stream-gap Event)

## Guardrail-Referenzen
- `docs/frontend/guardrails/frontend.md` — §6: Error-Handling (SSE-Verbindungsabbruch → Banner, nicht Error-Boundary), §5: Reconnect-Logik in shared/api/ gekapselt
- `CLAUDE.md` — Frontend Architecture: SSE Reconnect mit Exponential Backoff

## Notizen fuer den Implementierer
- Die SSE-Reconnect-Logik in `sseClient.ts` existiert bereits (Exponential Backoff). Diese Story behandelt nur die CHART-seitige Recovery — was passiert nachdem der SSE-Client sich erfolgreich reconnected hat.
- `stream-gap` kann auch ohne Reconnect auftreten — wenn das Backend einen Ring-Buffer-Overflow hatte. In dem Fall ist der SSE-Client weiter verbunden, aber Events fehlen. Der Recovery-Pfad ist identisch.
- Der REST-Refresh sollte die gleichen Endpoints verwenden die `useChartData` beim initialen Load nutzt. Code-Wiederverwendung ist hier wichtig.
- Die `isRestoring` Flag muss ein `useRef` sein, kein `useState` — ein State-Update wuerde einen Re-Render ausloesen, der waehrend der Recovery stoerend ist. Die Logik ist rein imperativ.
- Buffer-Groesse: In der Praxis werden waehrend eines REST-Calls (200-500ms) wenige Events ankommen. Ein einfaches Array reicht als Buffer.
- Edge-Case: Wenn waehrend der Recovery ein ZWEITER Reconnect passiert, muss der laufende Recovery abgebrochen und neu gestartet werden. AbortController fuer den REST-Call verwenden.

## Definition of Done
- [ ] Code kompiliert fehlerfrei (`tsc --noEmit`)
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Vitest: `*.test.ts`) fuer Recovery-Flow (isRestoring, Buffer, Flush, Epoch-Gating)
- [ ] Unit-Tests fuer stream-gap Handling
- [ ] Integrationstests fuer Reconnect → REST-Refresh → Buffer-Flush Sequenz
- [ ] Kein `any` ohne Begruendung
- [ ] Alle Interfaces mit `readonly`-Modifiern
- [ ] TSDoc auf allen geaenderten/neuen exportierten Funktionen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + TSDoc)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
