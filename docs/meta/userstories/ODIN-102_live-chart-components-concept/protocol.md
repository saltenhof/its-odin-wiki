# ODIN-102 — Implementierungsprotokoll

## Konzept-Erstellung

### Vorgehen
- Frontend-Analyse (Sub-Agent): 45 Chart-Dateien, SSE-Client, Zustand-Stores, Controls, Pages
- Konzept-Dokument geschrieben: `docs/concept/12-live-chart-components.md` (997 Zeilen)

### Bestandsaufnahme (Key Finding)
Die Frontend-Infrastruktur war bereits deutlich weiter als in der Story angenommen:
- IntraDayChart mit 16 Indicator-Layers, Multi-Panel-Layout, Live/History-Modus
- TradingEventLayer mit addTradeEvent() fuer Live-Updates
- SSE-Client mit Reconnect + Last-Event-ID
- Zustand State Management (global + domain stores)
- LiveChartPage mit mode="live" — aber Live-Bar-Streaming NICHT verdrahtet
- **Hauptluecke**: useChartData mode="live" ist gestubbt, kein Bar-Streaming, kein DecisionFeed

## Multi-LLM-Review

### ChatGPT-Review
**Fokus:** Komponentenarchitektur, State-Konsistenz, UX

Kritische Findings (eingearbeitet):
1. Single SSE ownership — kein Duplikat-Client in useChartLiveUpdates (§3.1, §3.5)
2. chartEpoch/dataVersion fuer Reconnect-Gating (§9.2)
3. rAF Batch-Cap + per-Frame Coalescing (§5.5)
4. Chart-Feed Linking (click-to-jump) — hoechstes UX-Leverage (§6.5)
5. DecisionFeed Virtualisierung + auto-scroll Freeze (§6.2)
6. Memory-Policy vereinheitlichen — 1.000 Events einheitlich (§4.2, §4.5)
7. Filter-Model geloest: Single-Select-Tabs (§6.3)

### Gemini-Review
**Fokus:** Performance, Race Conditions, UX-Konsistenz

Kritische Findings (eingearbeitet):
1. IntraDayChart als Monolith — useChartLiveUpdates SSE-agnostisch machen (§3.1, §3.5)
2. Reconnect Race Condition — isRestoring Flag + Event-Buffer via chartEpoch (§9.2)
3. rAF Queue Overflow — Batch-Cap pro Frame + Coalescing (§5.5)
4. DecisionFeed auto-scroll Freeze + "N new events" Indicator (§6.2)
5. Day-Change Reset UX — Rolling Window (letzte 30 Bars) statt hard Reset (§10.4)
6. Indicator Lag auf Live-Tick — bewusste Entscheidung, dokumentiert (§5.3)
7. Timezone-Alignment — Exchange-TZ im Chart konfigurieren (§11.2)

### Bewertung
Beide Reviews bestaetigen die Zwei-Pfad-Strategie (Store + imperativ).
Die Kritikpunkte betreffen Konsistenz-Risiken und UX-Feinheiten, nicht das Grunddesign.
Alle substanziellen Findings wurden ins Konzept eingearbeitet.

## Eingearbeitete Aenderungen (Detail)

| # | Finding | Betroffene Abschnitte | Art der Aenderung |
|---|---------|----------------------|-------------------|
| 1 | Single SSE Ownership | §3.1 (Mermaid), §3.5 | Kante UCLU→SSE entfernt, Designregel-Block eingefuegt, SSE-Agnostik dokumentiert |
| 2 | chartEpoch / Reconnect Gating | §9.2 | Komplett ueberarbeitet: ChartEpochState Interface, Ablaufdiagramm, Epoch-basierte Event-Filterung |
| 3 | rAF Batch-Cap + Coalescing | §5.5 | Code-Beispiel erweitert: MAX_UPDATES_PER_FRAME=50, coalesceBySeriesAndTime(), Yield-to-Browser |
| 4 | Chart-Feed Linking | §6.5 (NEU) | Neuer Abschnitt: onEventClick Callback, scrollToTime(), visuelles Feedback |
| 5 | DecisionFeed Virtualisierung | §6.2, §6.3 | Virtualisierung (react-window), Auto-Scroll-Freeze, Lazy-Format, Single-Select-Tabs |
| 6 | Day-Change UX | §10.4, Q2 | Rolling Context (30 Bars ausgegraut) als Default, Animated Reset als Fallback |
| 7 | Indicator Lag Dokumentation | §5.3 | Designentscheidungs-Block: Indikatoren nur auf completed Bars, kein Intra-Bar-Update |
| 8 | Timezone Handling | §11.2 (NEU) | Neuer Abschnitt: Exchange-TZ Pflicht, Referenz timeUtils.ts, Dual-TZ BarTooltip |
| 9 | Memory Policy Unification | §4.2, §4.5 | Zentrale MEMORY_LIMITS Konstante, tradingStore von 500 auf 1.000 vereinheitlicht |

## Ergebnis
- Konzept-Dokument: `docs/concept/12-live-chart-components.md`
- Umfang: 1.149 Zeilen (nach Review-Einarbeitung, vorher 997)
- Neue Abschnitte: §6.5 (Chart-Feed-Linking), §9.2 (chartEpoch, komplett ueberarbeitet), §10.4 (Day-Change-UX), §11.2 (Timezone)
- Geaenderte Abschnitte: §3.1, §3.5, §4.2, §4.5, §5.3, §5.5, §6.2, §6.3, Q2
- Status: Abgeschlossen
