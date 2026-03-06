# Protokoll: ODIN-126 — Simulationsuhr

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### 1. Prop-basierte Komponente (nicht direkt store-gekoppelt)

Gemaess Implementation Note 3 in der Story: `SimulationClock` akzeptiert `simTime: string | null` als Prop.
Der Store-Selector wird in `BacktestLiveView` aufgerufen (in der isolierten Sub-Komponente `SimClockStrip`),
analog zum `ProgressStrip`-Pattern. So bleibt die Komponente testbar ohne Store-Mock.

### 2. `Intl.DateTimeFormat` mit `hourCycle: 'h23'`

Verwendet `hourCycle: 'h23'` (nicht `hour12: false`), um sicherzustellen, dass Mitternacht
konsistent als "00:00" formatiert wird — nicht als "24:00" — was in einigen Browser/OS-Kombinationen
(insb. Safari) mit `hour12: false` auftreten kann. Gemini-Review D1 hat diesen Edge Case identifiziert.

### 3. Formatter-Instanzen als Modul-Konstanten

`Intl.DateTimeFormat`-Instanzen werden als Modul-Konstanten ausserhalb der Komponente definiert
(kein `new` bei jedem Render), um unnoetige Objekterstellung zu vermeiden.

### 4. CET vs. CEST — Modulo-Offset-Berechnung

Die DST-Erkennung (CET vs. CEST) wird via Offset-Vergleich geloest: Berlin-Lokalzeit (per
`BERLIN_HOUR_FORMATTER`) minus UTC-Zeit, modulo 1440 Minuten. >= 90 min Offset = CEST (UTC+2),
< 90 min = CET (UTC+1).

Hintergrund: Der native Ansatz `Intl.DateTimeFormat` mit `timeZoneName: 'short'` gibt in der
Node.js-Laufzeit "GMT+2" statt "CEST" zurueck — kein CET/CEST-Support. Der Modulo-Ansatz
funktioniert plattformunabhaengig (Node.js + Browser). Gemini (D3) schlug den nativen Ansatz vor,
dieser wurde bewusst wegen des Node.js-Kompatibilitaetsproblems verworfen.

Der Modulo-Ansatz ist explizit auf UTC+-Zeitzonen beschraenkt (Kommentar im Code).

### 5. ET statt EST/EDT

Laut Akzeptanzkriterium: immer "ET" (nicht EST/EDT dynamisch). Dies vereinfacht die Anzeige
und ist fuer Trading-UIs ueblich — Trader sprechen von "ET" unabhaengig von DST.

### 6. Isolierter Re-Render via `SimClockStrip`

Analog zum `ProgressStrip`-Pattern: Eine isolierte Sub-Komponente `SimClockStrip` in
`BacktestLiveView.tsx` liest `simTime` aus dem Store. Der Parent re-rendert nicht bei jeder
simTime-Aenderung.

### 7. `string | null` vs. Store-Typ `string`

Der Store definiert `simTime: string` und initialisiert es als `''`. Die Komponente akzeptiert
`string | null`, um auch explizite null-Werte zu handhaben (Robustheit). Der Leerstring-Check
greift fuer den initialen Store-Zustand.

## ChatGPT-Sparring (DoD 5.5)

**Session:** ChatGPT-Sparring zu Test-Edge-Cases fuer `SimulationClock`.

**Vorgeschlagene Edge Cases:**

| # | Kategorie | Szenario | Umgesetzt? | Begruendung |
|---|-----------|----------|------------|-------------|
| 1 | US DST | Spring-forward boundary (2025-03-09) | Ja | Kritisch: 02:xx wird uebersprungen in ET |
| 2 | EU DST Fall-back | 02:30 CEST vs. 02:30 CET (repeated hour) | Ja | Kritisch: gleiche Uhrzeit, zwei Abkuerzungen |
| 3 | US DST Fall-back | 01:30 ET repeated (2025-11-02) | Ja | Gut fuer ET-Label-Konsistenz |
| 4 | ISO mit Offset | `+01:00`, `-05:00` | Ja | Backend-Kontraktvalidierung |
| 5 | Millisekunden | `.123Z`, `.999Z`, kein Sekundenfeld | Ja | Minutenrundung pruefen |
| 6 | Offsetloser ISO | `2025-01-15T14:30:00` (keine Z) | Nein | Verworfen: Backend-Kontrakt garantiert Z-Suffix; Test waere environment-sensitiv (lokale TZ) |
| 7 | Semantisch invalid | `2025-02-30` (JavaScript normalisiert) | Nein | Verworfen: Kein Fehler aus Trading-Sicht; JavaScript-Normalisierung ist kein Bug |
| 8 | Berlin Mitternacht | 23:30 UTC = 00:30 CET next day | Ja | Day-rollover fuer Berlin |
| 9 | `h23` Mitternacht | `24:00` vs `00:00` Browser edge | Implizit | Adressiert durch `hourCycle: 'h23'` im Code |

**Ergebnis:** 6 neue Test-Kategorien umgesetzt (US Spring, EU Fall, US Fall, ISO offsets, Millisekunden, Berlin-Midnight-Crossover).
Gesamtzahl Tests: 28 (von 16 initial auf 28 erweitert).

## Gemini-Review (DoD 5.6)

### Dimension 1: Code-Review

**Findings:**

| # | Schwere | Finding | Massnahme |
|---|---------|---------|-----------|
| 1 | MAJOR | `hour12: false` kann in Safari/Chrome `24:00` statt `00:00` produzieren | Behoben: `hourCycle: 'h23'` in allen drei Formattern |
| 2 | MINOR | Modulo-Formel ohne Kommentar ueber UTC+-Einschraenkung | Behoben: Kommentar hinzugefuegt |
| 3 | INFO | `getBerlinTzAbbreviation` silent fallback — keine Fehlerloggung | Bewusst akzeptiert: UI darf nicht crashen; kein `console.log` (ESLint: no-console) |
| 4 | INFO | `React.memo` fehlt | Bewusst verworfen: `SimulationClock` wird bereits durch `SimClockStrip` isoliert; `React.memo` erst nach nachgewiesenem Re-Render-Problem (CLAUDE.md Guardrail R11.2) |

### Dimension 2: Konzepttreue

**Findings:**

| # | Bereich | Finding | Massnahme |
|---|---------|---------|-----------|
| 1 | Header-Integration | `SimClockStrip` hat kein Label-Span (anders als SimDate) | Bewusst so: Story-AC sagt "neben ProgressStrip oder currentDate", kein Label vorgeschrieben. Akzeptiert. |
| 2 | Store-Typ | `simTime: string` (Store) vs. `string \| null` (Prop) — korrekt gehandhabt | Bestaetigt: `''`-Check greift fuer initialen Store-Zustand. |
| 3 | Locale fuer NY | Story sagt `de-DE`, Implementation nutzt `en-US` fuer NY-Formatter | Bewusst: `en-US` fuer New York logisch; bei `hourCycle: 'h23'` identische Ausgabe. Akzeptiert. |

**Gesamtbewertung:** Konzepttreu. Alle Akzeptanzkriterien erfuellt.

### Dimension 3: Praxis-Review

**Findings (evaluiert):**

| # | Szenario | Bewertung | Massnahme |
|---|----------|-----------|-----------|
| 1 | "Slot Machine" Effect bei 1000x Speed | INFO | In Scope als Datenquelle (SSE), nicht als Anzeige-Problem; UI zeigt was gesendet wird. Kein Handlungsbedarf im Rahmen dieser Story. |
| 2 | Asynchrone DST (US 3 Wochen vor EU) | INFO | `Intl.DateTimeFormat` handhabt das korrekt; unsere Tests beweisen das. Kein Handlungsbedarf. |
| 3 | Time Skips (Nights/Weekends) | INFO | Die Simulation zeigt die SimTime korrekt; Day-Skip-Visualisierung ist Out-of-Scope fuer ODIN-126. |
| 4 | RTH vs. ETH Farbindikator | IDEA | Interessant fuer zukuenftige Stories; Out-of-Scope ODIN-126. In Offene Punkte. |
| 5 | Stale Data / Connection Drop | INFO | Wird durch `connectionStatus` im Store und `ConnectionBanner` gehandhabt (bereits implementiert). |
| 6 | Native `timeZoneName: 'short'` | REJECTED | Gibt in Node.js "GMT+2" statt "CEST" — inkompatibel mit Test-Environment. Modulo-Ansatz behalten. |

## Offene Punkte

- **IDEA (niedrige Prioritaet):** RTH/ETH-Indikator als Farbpunkt neben der Uhrzeit, um
  Market-State-Kontext direkt sichtbar zu machen (Gemini D3 Suggestion). Potentielle zukuenftige Story.
- **HINWEIS:** Backend muss `simTime` immer mit `Z`-Suffix senden (ISO 8601 UTC). Offsetlose
  Timestamps wuerden in der Lokalen Zeitzone des Browsers interpretiert. Konzept-Kontrakt ist klar,
  aber ein defensiver Backend-Validierungsschritt waere sinnvoll.
