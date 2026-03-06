# Protokoll: ODIN-118 — Decision Feed — Virtualisierter Activity-Log

**Datum:** 2026-03-06
**Status:** Abgeschlossen (nach QA-Runde R1 Nacharbeit)

---

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (35 Tests, alle gruen)
- [x] ChatGPT-Sparring fuer Edge-Cases und Scroll-Detection
- [x] Gemini-Review Dimension 1 (Code) — via ChatGPT als Fallback (Gemini Pool defekt)
- [x] Gemini-Review Dimension 2 (Konzepttreue) — via ChatGPT als Fallback
- [x] Gemini-Review Dimension 3 (Praxis) — via ChatGPT als Fallback
- [x] Review-Findings eingearbeitet (F-01, F-02, F-03 aus QA-Report R1)
- [x] Commit & Push

---

## Erstellte / Geaenderte Dateien

### Neue Dateien

| Datei | Beschreibung |
|-------|-------------|
| `src/shared/components/DecisionFeed/DecisionFeed.tsx` | Hauptkomponente: Virtualisierter Feed mit Auto-Scroll, Filter, Badge |
| `src/shared/components/DecisionFeed/DecisionFeedItem.tsx` | Zeilenkomponente: Kategorie-Badge, Expand/Collapse, JSON-Detail |
| `src/shared/components/DecisionFeed/DecisionFeedFilter.tsx` | Tab-Bar fuer Kategorie-Filter |
| `src/shared/components/DecisionFeed/types.ts` | TypeScript-Interfaces: `DecisionFeedEvent`, `DecisionFeedProps`, `DecisionCategory` |
| `src/shared/components/DecisionFeed/DecisionFeed.module.css` | CSS Module: Container, ListWrapper, Badge, EmptyState |
| `src/shared/components/DecisionFeed/DecisionFeedItem.module.css` | CSS Module: Item-Layout, Kategorie-Farbcodierung |
| `src/shared/components/DecisionFeed/DecisionFeedFilter.module.css` | CSS Module: Tab-Bar-Styling |
| `src/shared/components/DecisionFeed/DecisionFeed.test.tsx` | 35 Unit/Integration-Tests |
| `src/shared/components/DecisionFeed/index.ts` | Barrel Export |
| `src/shared/config/memoryLimits.ts` | `MEMORY_LIMITS.DECISION_EVENTS_MAX` Konstante |

### Geaenderte Dateien

| Datei | Aenderung |
|-------|----------|
| `src/shared/components/index.ts` | Barrel-Export fuer DecisionFeed hinzugefuegt |

---

## Design-Entscheidungen

### 1. react-window v2 statt v1

Die Story nennt `FixedSizeList` aus react-window v1. Installiert wurde react-window v2.2.7, das eine neue API mit `List` und `rowComponent`-Pattern verwendet. Die v2-API wurde gewaehlt (statt Downgrade auf v1), da sie moderner ist und dieselbe Virtualisierungsleistung bietet.

### 2. FixedSizeList-Aequivalent: Einheitliche Zeilenhoehe

Anstatt `VariableSizeList` (komplex, benoetigt Hoehenmessung per Item) wird eine einheitliche Item-Hoehe verwendet: 48px collapsed, 220px expanded. Wenn mindestens ein Item expanded ist, wechselt die gesamte Liste auf 220px. Unexpandierte Items haben dann extra Whitespace — akzeptabler Tradeoff fuer einfachere Implementierung. (Bekanntes Verhalten, in QA-F-04 dokumentiert.)

### 3. Auto-scroll-Detection via onScroll am react-window List-Container (Fix F-01)

**Urspruenglicher Ansatz (defekt):** `onScroll` wurde am aeusseren Wrapper-Div (overflow: hidden) registriert. Da react-window v2 `List` intern ein eigenes Scroll-Container-Div mit `overflow: auto` rendert, erreichte kein Scroll-Event das Wrapper-Div.

**Korrigierter Ansatz:** `onScroll={handleListScroll}` wird direkt als Prop an die `List`-Komponente uebergeben. Da `ListProps` `Omit<HTMLAttributes<HTMLDivElement>, 'onResize'>` erweitert, werden alle Standard-DOM-Attribute inklusive `onScroll` via `...rest` an das aeussere Div der List weitergeleitet. Das aeussere Div der List ist identisch mit dem tatsaechlichen Scroll-Container.

**Verbesserung durch Sparring:** Die Scroll-Abstandsberechnung wurde durch Clamping abgesichert:
```typescript
const maxScrollTop = Math.max(0, target.scrollHeight - target.clientHeight);
const clampedScrollTop = Math.min(Math.max(0, target.scrollTop), maxScrollTop);
const distanceFromBottom = maxScrollTop - clampedScrollTop;
```
Dies schuetzt vor subpixel-Werten und Browser-Rubber-Band-Effekten (Safari Overscroll).

### 4. filterEvents und buildEventCounts als exportierte Funktionen

Diese Pure-Functions werden aus der Komponente exportiert, um direkte Unit-Tests ohne React-Rendering zu ermoeglichen (bessere Testbarkeit).

### 5. Timestamps in lokaler Zeit

`formatTimestamp` nutzt `getHours()` (lokale Zeit), nicht `getUTCHours()`. Dies entspricht dem erwarteten Verhalten einer Trading-UI (Operator sieht lokale Marktzeit). Tests sind timezone-agnostisch formuliert (Regex `\d{2}:\d{2}:45`).

### 6. ResizeObserver fuer Hoehenmessung

Der Container nutzt `ResizeObserver`, um die Listenhoehe dynamisch zu messen. In Tests wird ResizeObserver als no-op gemockt.

### 7. Gemini-Pool-Fallback auf ChatGPT (analog ODIN-116)

Der Gemini Pool Service lieferte auch fuer diese Story `[unknown, 0ms]` fuer alle Nachrichten. Alle drei Review-Dimensionen wurden mit separaten ChatGPT-Sessions durchgefuehrt (owner: `ODIN-118-review-d1` als kombinierte Session). Die inhaltliche Qualitaet des Reviews war hoch.

---

## Offene Punkte

### A. FIFO-Eviction-Detection (ChatGPT-Finding, NIEDRIG priorisiert)

ChatGPT identifizierte: Bei bounded feeds mit FIFO-Eviction (drop 1, add 1) bleibt die Array-Laenge konstant — der `pendingNewEvents`-Zaehler wuerde nicht inkrementieren. Da ODIN derzeit keine FIFO-Eviction im Frontend implementiert (Events werden nur angehaeuft bis zum Session-Reset), ist dies kein akutes Problem. Fuer eine zukuenftige Eviction-Implementierung: Tracking via letzter Event-ID statt Array-Laenge.

### B. Burst-Throttling fuer Auto-Scroll (ChatGPT-Finding, NIEDRIG priorisiert)

Bei sehr schnellen Event-Bursts koennte `scrollToRow` pro Event aufgerufen werden. Ein rAF-basiertes Throttling (analog ODIN-116) wuerde den Main-Thread entlasten. Akzeptabel fuer den aktuellen Intraday-Scope; relevanter fuer Backtest-Replay-Bursts.

### C. Badge-Count-Cap (ChatGPT-Finding, NIEDRIG priorisiert)

Bei sehr langen Pausen koennte der Badge-Count beliebig gross werden. Ein Display-Cap (`99+`) waere eine UX-Verbesserung. Akzeptabel fuer den aktuellen Scope.

---

## ChatGPT-Sparring

**Session:** 1x ChatGPT (owner=ODIN-118-rework), ~169 Sekunden

**Kontext:** Sparring zu Auto-scroll-Freeze-Detection, Scroll-Threshold-Berechnung und Test-Strategie fuer F-03.

### Vorgeschlagene Edge-Cases und Bewertung

| Vorschlag | Umgesetzt? | Begruendung |
|-----------|-----------|------------|
| Clamped `scrollTop`-Berechnung (`maxScrollTop - clampedScrollTop`) | JA | Schuetzt vor Safari-Rubber-Band und Subpixel-Werten |
| Hysterese-Threshold (freeze bei >24px, resume bei <=8px) | NEIN | Overkill fuer aktuellen Scope; einfacher einheitlicher 20px-Threshold genuegt |
| `pendingNewEvents` auf 0 beim Resume (scroll zurueck zu Bottom) | JA (bereits umgesetzt) | Cleanste UX: Badge = Events waehrend Pause, nicht historischer Counter |
| Test via echtem Scroll-Element (data-testid + fireEvent.scroll) | JA | Mock-List gibt onScroll und data-testid weiter; Test prueft jetzt echtes Badge-Verhalten |
| FIFO-Eviction: Event-ID statt Laenge tracken | NEIN (dokumentiert) | Kein FIFO in aktuellem Frontend; als offener Punkt dokumentiert |
| Programmatische Scrolls feuern auch onScroll (Badge-Click-Loop-Risk) | BESTAETIGT KEIN PROBLEM | `handleListScroll` prueft `isAutoScrollActive` vor State-Aenderung; Loop nicht moeglich |
| Background-Tab-Handling | NEIN (dokumentiert) | Out-of-Scope; akzeptabler State nach Tab-Wechsel |

---

## Gemini-Review

**Fallback auf ChatGPT** (Gemini Pool defekt — identisches Verhalten wie in ODIN-116)
**Session:** ChatGPT (owner=ODIN-118-review-d1), ~345 Sekunden, kombinierte Dimension-1/2/3-Review

### Dimension 1: Code Bugs & Qualitaet

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | Auto-scroll bricht bei FIFO-Eviction (konstante Array-Laenge) | OFFEN: Dokumentiert als offener Punkt A; kein FIFO in aktuellem Design |
| 2 | HIGH | `listRef`-Timing: `prevFilteredCountRef` wird aktualisiert bevor `listRef` verfuegbar ist | AKZEPTIERT: Timing-Problem minimal; `useListCallbackRef` gibt ref fruehzeitig zurueck |
| 3 | HIGH | Fehlende stabile `itemKey`-Funktion (`event.id`) fuer react-window | AKZEPTIERT: react-window v2 `List` nutzt automatisch index-based keys; ID-basierter Key wuerde `itemKey`-Prop erfordern das v2-API-Doc nicht explizit nennt |
| 4 | MEDIUM/HIGH | Expanded-Row + Fixed-Row-Height nicht kompatibel (globale Hoehenumschaltung) | DOKUMENTIERT als Design-Entscheidung #2 und QA-Finding F-04 (Known Behavior) |
| 5 | MEDIUM | `hasExpandedItem` ignoriert Filterung (expanded IDs ausserhalb des sichtbaren Filters) | AKZEPTIERT als bekannter Tradeoff; Fix wuerde Set-Intersection erfordern |
| 6 | MEDIUM | `expandedIds` Leakage bei Event-Eviction | Nicht relevant (kein FIFO) |
| 7 | MEDIUM | ResizeObserver guard fuer SSR | JA: Test-Mock vorhanden; SSR ist fuer ODIN (SPA) nicht relevant |
| 8 | MEDIUM | Accessibility-Gaps (ARIA-Roles, keyboard interaction) | OFFEN: Nicht im Story-Scope; Trading-Dashboard ist Desktop-only fuer Operator |

### Dimension 2: Konzepttreue

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | MEDIUM | Events-Sortierung wird nicht erzwungen | DOKUMENTIERT: Invariante "events kommen chronologisch sortiert" ist Store-Verantwortung |
| 2 | MEDIUM | Auto-resume-Threshold (20px) kann bei Momentum-Scroll ungewollt triggern | AKZEPTIERT: Konservative Implementierung; Hysterese als offener Punkt |
| 3 | MEDIUM | Expansion-State wird bei Filter-Wechsel nicht geloescht | AKZEPTIERT: Expansion ist global, passt zu "gleichhoehige Zeilen" Design-Entscheidung |
| 4 | LOW/MEDIUM | filterEvents O(n) bei jedem Events-Update | AKZEPTIERT: Mit `useMemo` und Max-Cap (MEMORY_LIMITS) akzeptabel |

### Dimension 3: Praxis-Gaps

| # | Severity | Finding | Eingearbeitet |
|---|---------|---------|--------------|
| 1 | HIGH | Burst-Events koennen `scrollToRow` pro Event ausfuehren (Frame-Drops) | OFFEN: Dokumentiert als offener Punkt B; rAF-Throttling empfohlen fuer spaetere Iteration |
| 2 | HIGH | Background-Tab: grossen Event-Batch beim Tab-Wechsel | OFFEN: Akzeptabel fuer aktuellen Scope |
| 3 | MEDIUM | Badge-Count unbounded bei langer Pause | OFFEN: Dokumentiert als offener Punkt C; Display-Cap empfohlen |
| 4 | MEDIUM | ResizeObserver kann rapid feuern bei Panel-Resize | AKZEPTIERT: `setListHeight` ist guenstig, State-Update nur bei echten Aenderungen |
| 5 | MEDIUM | StrictMode / ref-callback-Churn | AKZEPTIERT: `useListCallbackRef` von react-window v2 ist darauf ausgelegt |
| 6 | LOW/MEDIUM | Fehlende Observability (Telemetrie fuer Feed-Lag) | OFFEN: Out-of-Scope fuer ODIN-118; kein Bedarf im aktuellen Operator-Kontext |

---

## Build-Status

```
tsc -b && vite build: GREEN (245 Module transformiert)
vitest run: 519 Tests, 33 Dateien — alle PASS
  davon DecisionFeed.test.tsx: 35 Tests (inkl. Badge-Behavior-Verifikation nach F-03-Fix)
```

## Erstellte Commits

Siehe Git-Log des `its-odin-ui` Repos.
