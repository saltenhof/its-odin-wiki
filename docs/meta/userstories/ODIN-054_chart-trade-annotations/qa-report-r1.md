# QS-Bericht ODIN-054 — Runde 1

**Datum:** 2026-02-24
**QS-Agent:** Sub-Agent (QS-Runde 1)
**Commit geprüft:** d7f58aa (its-odin-ui)

---

## Binäres Ergebnis: FAIL

---

## Findings

### FAIL-1: Failing Unit Test — Negative PnL Formatierung
**Schwere:** Bug
**Beschreibung:** Test `TradeMarkerTooltip.test.tsx:187` schlägt fehl. Die Funktion `formatPnl()` in `TradeMarkerTooltip.tsx` erzeugt für negative Werte die falsche Reihenfolge von Vorzeichen und Dollar-Symbol.

**Root Cause:** In `formatPnl`:
```typescript
function formatPnl(pnl: number): string {
  const sign = pnl >= 0 ? '+' : '';
  return `${sign}$${pnl.toFixed(2)}`;
}
```
Für `pnl = -45.00`: `pnl.toFixed(2)` gibt `-45.00` zurück, weshalb das Template `$${pnl.toFixed(2)}` zu `$-45.00` wird statt `-$45.00`.

**Fundstelle:** `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/components/TradeMarkerTooltip.tsx:59-62`

**Test-Assertion:** `TradeMarkerTooltip.test.tsx:187` — `expect(screen.getByText('-$45.00')).toBeDefined()`

**Erwartetes Verhalten:** Für negative PnL muss das Minuszeichen VOR dem Dollar-Symbol stehen: `-$45.00`. Die Funktion muss umgeschrieben werden:
```typescript
function formatPnl(pnl: number): string {
  const abs = Math.abs(pnl).toFixed(2);
  return pnl >= 0 ? `+$${abs}` : `-$${abs}`;
}
```

---

### FAIL-2: ChatGPT-Sparring nicht durchgeführt
**Schwere:** Prozess
**Beschreibung:** DoD-Punkt 2.5 verlangt zwingend eine ChatGPT-Sparring-Session für Test-Edge-Cases. Laut `protocol.md` war der ChatGPT-Pool während der Implementierung nicht erreichbar (`[unknown, 0ms]` für alle Sends). Der Schritt wurde übersprungen statt abgebrochen und eskaliert (Verletzung von Regel 7.2 der User-Story-Spezifikation: „Fail-Fast bei blockierten DoD-Schritten").

**Fundstelle:** `T:/codebase/its_odin/temp/userstories/ODIN-054_chart-trade-annotations/protocol.md` — Abschnitt „Review Status"

**Erwartetes Verhalten:** Laut Spezifikation §7.2: Sofort abbrechen und Fehlermeldung zurückgeben, wenn ein Pflichtschritt der DoD nicht ausführbar ist. Keine Partial-Completion.

---

### FAIL-3: Gemini-Review nicht durchgeführt
**Schwere:** Prozess
**Beschreibung:** DoD-Punkt 2.6 verlangt zwingend ein Gemini-Review in drei Dimensionen (Code, Konzepttreue, Praxis). Laut `protocol.md` war auch der Gemini-Pool nicht erreichbar. Gleiches Problem wie FAIL-2.

**Fundstelle:** `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/components/TradeMarkerTooltip.tsx` (nicht reviewed)

**Erwartetes Verhalten:** Alle drei Review-Dimensionen müssen dokumentiert und Findings eingearbeitet worden sein.

---

### FAIL-4: Protocol.md fehlen Pflichtabschnitte
**Schwere:** Prozess
**Beschreibung:** Die `protocol.md` verwendet ein abweichendes Format und enthält NICHT die in der User-Story-Spezifikation §2.7 vorgeschriebenen Pflichtabschnitte. Folgende Pflichtabschnitte fehlen vollständig:
- `## Working State` (Checkliste mit Meilensteinen)
- `## Design-Entscheidungen` (stattdessen: `## Implementation Decisions` — falscher Name)
- `## Offene Punkte`
- `## ChatGPT-Sparring`
- `## Gemini-Review`

Die vorhandenen Abschnitte (Summary, Files Changed, Implementation Decisions, Test Coverage, Build Status, Review Status, Notes for Future Work) weichen von der vorgeschriebenen Struktur ab.

**Fundstelle:** `T:/codebase/its_odin/temp/userstories/ODIN-054_chart-trade-annotations/protocol.md`

**Erwartetes Verhalten:** Protocol.md muss exakt die fünf Pflichtabschnitte aus der Spezifikation §2.7 enthalten.

---

### FAIL-5: Fehlender Vitest Komponenten-Integrationstest (DoD 2.3)
**Schwere:** Prozess
**Beschreibung:** DoD-Punkt 2.3 verlangt explizit: „Vitest-Komponenten-Test: `IntraDayChart` mit Mock-Trades zeigt die erwartete Anzahl Marker". Es gibt keinen Test der Art `IntraDayChart.test.tsx` oder ähnliches. Die vorhandenen Tests (`TradingEventLayer.test.ts`, `TradeMarkerTooltip.test.tsx`, `useChartData.test.ts`) sind alle Unit-Tests auf Klassenebene (DoD 2.2). Kein Integrationstest existiert, der mehrere reale Implementierungen zusammenschaltet.

**Fundstelle:** Kein `IntraDayChart*.test.*`-File im Repo vorhanden.

**Erwartetes Verhalten:** Mindestens 1 Integrationstest der `IntraDayChart`-Komponente mit Mock-Trades, der die korrekte Anzahl Marker verifiziert.

---

### FAIL-6: AC-4 nur teilweise erfüllt — STOP_TRAIL als Square-Marker statt horizontaler Linie
**Schwere:** Konzepttreue / Minor
**Beschreibung:** AC-4 spezifiziert: „Stop-Anpassungen (STOP_TRAIL) erscheinen als horizontale gestrichelte Linie oder Marker auf der Stop-Preisebene, an der Zeitposition der Anpassung." Die Implementierung setzt STOP_TRAIL als `square`-Marker (`aboveBar`), NICHT auf der tatsächlichen Stop-Preislinie. Die Story-Spec und das technische Detail benennen explizit eine horizontale Linie auf der Stop-Preisebene (z.B. via `PriceLine` in lightweight-charts) als bevorzugte Variante.

**Fundstelle:**
- `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/layers/TradingEventLayer.ts:114-124` (STOP_TRAIL als `square aboveBar`)
- Story, Technische Details: „Stop-Trail-Marker als horizontale `PriceLine` in lightweight-charts oder als spezieller Marker"

**Bewertung:** Die Story erlaubt explizit beide Optionen („oder als spezieller Marker"), daher ist dies akzeptabel als Design-Entscheidung — ABER diese Entscheidung ist im protocol.md nicht begründet. Da AC-4 beide Varianten nennt und der Marker-Ansatz implementiert wurde, ist dies ein MINOR-Finding, kein harter Blocker. Der QS-Bericht registriert es als Abweichung ohne eigenständigen FAIL-Status, aber die fehlende Begründung in protocol.md verstärkt FAIL-4.

---

### Minor-1: TRANCHE_ADD verwendet circle statt Raute/Diamond
**Schwere:** Minor
**Beschreibung:** Die Story-Spezifikation (Technische Details, Marker-Typen) gibt für TRANCHE_ADD „◆ (Raute/Diamond)" als Symbol vor. Die lightweight-charts API bietet keinen `diamond`-Shape (`'circle' | 'square' | 'arrowUp' | 'arrowDown'`). Die Implementierung nutzt `circle`, was die einzig sinnvolle Alternative ist. Dies ist eine korrekte Pragmatik-Entscheidung, aber sie ist nicht im protocol.md begründet.

**Fundstelle:** `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/layers/TradingEventLayer.ts:74`

**Erwartetes Verhalten:** Akzeptable Lösung — fehlende Begründung verstärkt FAIL-4.

---

### Minor-2: STOP_TRIGGER-Typ in AC-4 nicht erwähnt
**Schwere:** Minor
**Beschreibung:** Die Implementierung fügt einen sechsten Event-Typ `STOP_TRIGGER` hinzu, der in der Story nicht spezifiziert ist. Die fünf spezifizierten Typen aus Story §Technische Details sind: ENTRY, TRANCHE_ADD, PARTIAL_EXIT, FULL_EXIT, STOP_TRAIL. `STOP_TRIGGER` ist eine Eigenentscheidung des Implementierungs-Agents. Inhaltlich sinnvoll, aber nicht beauftragt.

**Fundstelle:** `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/types/chart.ts:46`, `TradingEventLayer.ts:103-112`, `TradeMarkerTooltip.tsx:33,43`

**Erwartetes Verhalten:** Nicht-spezifizierte Additions sollten im protocol.md unter Design-Entscheidungen begründet werden.

---

### Minor-3: Inline-Stil statt CSS Modules in TradeMarkerTooltip
**Schwere:** Minor
**Beschreibung:** `TradeMarkerTooltip.tsx` verwendet ausschließlich Inline-Styles (`style={{...}}`). Das Frontend-Guardrail (`docs/frontend/guardrails/frontend.md`) schreibt CSS Modules vor. Für ein dynamisch positioniertes Overlay mit wenigen Styles ist Inline-Style pragmatisch, aber es widerspricht der Guardrail-Vorgabe.

**Fundstelle:** `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/components/TradeMarkerTooltip.tsx:88-125`

**Erwartetes Verhalten:** Positionale Styles (left, top) als Inline akzeptabel, aber statische Styles (backgroundColor, border, etc.) sollten in CSS Modules ausgelagert sein.

---

### Minor-4: non-null assertion (`!`) ohne explizite Konstante
**Schwere:** Minor
**Beschreibung:** In `PricePanel.tsx:176` wird `currentChart!.timeScale()` mit einem `@typescript-eslint/no-non-null-assertion` eslint-disable-comment verwendet. Das protocol.md dokumentiert dies als „known TypeScript limitation for closures", was korrekt ist. Allerdings verletzt dies streng genommen die TypeScript-strict-Vorgabe ohne garantierten strukturellen Schutz. Pragmatisch akzeptabel, aber nicht ideal.

**Fundstelle:** `T:/codebase/its_odin/its-odin-ui/src/shared/components/IntraDayChart/panels/PricePanel.tsx:175-176`

---

## Vollständige DoD-Checkliste

### 2.1 Code-Qualität

| Punkt | Status | Begründung |
|-------|--------|------------|
| Implementierung vollständig gemäß Akzeptanzkriterien | ✅ Weitgehend | AC-1 bis AC-8 funktional implementiert. AC-4 mit vertretbarer Variante (square marker statt PriceLine). |
| npm run build fehlerfrei | ✅ | Build clean laut Orchestrator-Vorinformation |
| TypeScript strict, kein `any` | ✅ | Kein `any` oder `as any` in neuen Dateien gefunden |
| Keine Magic Numbers — Konstanten für Farben/Größen | ✅ | `MARKER_SIZE_LARGE`, `MARKER_SIZE_SMALL`, `MARKER_PROXIMITY_THRESHOLD_PX`, `TOOLTIP_OFFSET_X/Y`, `EVENT_TYPE_LABELS`, `EVENT_TYPE_COLORS` vorhanden. CSS-Tokens für alle Farben. |
| Keine TODO/FIXME-Kommentare | ✅ | Keine TODO/FIXME in neuen Dateien gefunden |
| Code-Sprache Englisch | ✅ | Alle neuen Dateien in Englisch |

### 2.2 Tests — Klassenebene (Unit-Tests)

| Punkt | Status | Begründung |
|-------|--------|------------|
| Unit-Tests für TradingEventLayer | ✅ | 9 Tests in `TradingEventLayer.test.ts` — alle Event-Typen, Sortierung, Visibility, addEvent, destroy |
| Unit-Tests für Tooltip-Rendering | ❌ | 12 Tests in `TradeMarkerTooltip.test.tsx` — aber Test #10 (negative PnL) SCHLÄGT FEHL |
| Unit-Tests für useChartData Mapping | ✅ | 7 Tests in `useChartData.test.ts` |
| Neue Geschäftslogik → Unit-Test PFLICHT | ✅ | Vorhanden, aber 1 Test rot |

### 2.3 Tests — Komponentenebene (Integrationstests)

| Punkt | Status | Begründung |
|-------|--------|------------|
| Vitest-Komponenten-Test IntraDayChart mit Mock-Trades | ❌ | FEHLT VOLLSTÄNDIG. Kein `IntraDayChart.test.tsx` oder ähnlicher Integrationstest vorhanden |
| Mindestens 1 Integrationstest pro Story | ❌ | Kein Integrationstest der Hauptfunktionalität End-to-End |

### 2.4 Tests — Datenbank (nicht zutreffend)

| Punkt | Status | Begründung |
|-------|--------|------------|
| N/A — kein Datenbankzugriff | ✅ N/A | Reine Frontend-Story |

### 2.5 Test-Sparring mit ChatGPT

| Punkt | Status | Begründung |
|-------|--------|------------|
| ChatGPT-Session mit Klassen, ACs, Tests | ❌ | NICHT DURCHGEFÜHRT — Pool-Problem. Fail-Fast-Regel (§7.2) wurde verletzt statt zu eskalieren |
| Ergebnis in protocol.md dokumentiert | ❌ | Kein `## ChatGPT-Sparring`-Abschnitt |

### 2.6 Review durch Gemini

| Punkt | Status | Begründung |
|-------|--------|------------|
| Dimension 1: Code-Review | ❌ | NICHT DURCHGEFÜHRT |
| Dimension 2: Konzepttreue | ❌ | NICHT DURCHGEFÜHRT |
| Dimension 3: Praxis-Review | ❌ | NICHT DURCHGEFÜHRT |
| Findings bewertet und eingearbeitet | ❌ | NICHT DURCHGEFÜHRT |
| Ergebnis in protocol.md | ❌ | Kein `## Gemini-Review`-Abschnitt |

### 2.7 Protokolldatei

| Punkt | Status | Begründung |
|-------|--------|------------|
| protocol.md vorhanden | ✅ | Datei existiert |
| `## Working State` (Checkliste) | ❌ | FEHLT — falsches Format verwendet |
| `## Design-Entscheidungen` | ❌ | Heißt `## Implementation Decisions` — falscher Abschnittname laut Spezifikation |
| `## Offene Punkte` | ❌ | FEHLT — stattdessen `## Notes for Future Work` |
| `## ChatGPT-Sparring` | ❌ | FEHLT |
| `## Gemini-Review` | ❌ | FEHLT |
| Live-Aktualisierung während Arbeit | Unklar | Format-Abweichung macht das nicht prüfbar |

### 2.8 Abschluss

| Punkt | Status | Begründung |
|-------|--------|------------|
| Commit vorhanden | ✅ | Commit d7f58aa |
| Push auf Remote | ✅ | Laut Orchestrator bestätigt |
| Story-Verzeichnis enthält story.md + protocol.md | ✅ | Beide vorhanden |

---

## Zusammenfassung der FAIL-Gründe

| Nr. | Kategorie | Schwere | Beschreibung |
|-----|-----------|---------|--------------|
| FAIL-1 | Bug | Bug | Negativer PnL formatiert als `$-45.00` statt `-$45.00` → 1 Test rot |
| FAIL-2 | Prozess | Prozess | ChatGPT-Sparring nicht durchgeführt, nicht eskaliert |
| FAIL-3 | Prozess | Prozess | Gemini-Review (alle 3 Dimensionen) nicht durchgeführt |
| FAIL-4 | Prozess | Prozess | protocol.md fehlen alle 5 Pflichtabschnitte der Spezifikation |
| FAIL-5 | Test | Prozess | Kein Vitest-Integrationstest für IntraDayChart |
| FAIL-6 | Konzept | Minor | AC-4 STOP_TRAIL als square-Marker, nicht als PriceLine — Design-Entscheidung unzureichend begründet |

**Harte Blocker (Nacharbeit zwingend):** FAIL-1, FAIL-2, FAIL-3, FAIL-4, FAIL-5
