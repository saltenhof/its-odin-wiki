# Protokoll: ODIN-097 — ATR-normalisierte Event-Schwellen fuer LLM-Sampling

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### ATR-Quelle: Berechnung aus fiveMinBars im Snapshot

**Problem:** `MarketSnapshot` enthaelt kein ATR-Feld. Die ATR liegt in `IndicatorResult`, das nur im `TradingPipeline` nach `kpiEngine.onSnapshot()` verfuegbar ist. Das `LlmSamplingPolicy`-Interface erhaelt nur den `MarketSnapshot`.

**Optionen:**
1. ATR aus `fiveMinBars` im Snapshot selbst berechnen (einfacher True Range Average)
2. `LlmSamplingPolicy`-Interface erweitern (bricht Interface-Stabilitaet)
3. ATR als Parameter zum Konstruktor (statisch, skaliert nicht dynamisch)

**Entscheidung:** Option 1 — ATR direkt aus `snapshot.fiveMinBars()` berechnen (True Range Average ueber die letzten 14 Bars). Das entspricht exakt dem ATR(14) auf 5m-Bars, das auch der `KpiEngine` verwendet. Der Snapshot enthaelt diese Bars bereits. NaN-safe: falls weniger als 2 Bars vorhanden oder TR nicht berechenbar → NaN → Fallback (kein Trigger).

**Warum nicht IndicatorResult weiterreichen:** Das Interface bleibt stabil (`shouldSample(MarketSnapshot, long, long)`). Die `CompositeSamplingPolicy`-Integration bleibt unveraendert. Keine Cascade-Aenderungen noetig.

### Fallback bei NaN-ATR

Wenn ATR nicht berechnet werden kann (weniger als 2 Bars, NaN-Preise), gibt die Policy `false` zurueck. Kein Exception-Throw. Dies ist das sichere Default-Verhalten: "keine unbekannte Volatilitaet → kein Trigger".

### Default-Wert 0.6

Default 0.6 entspricht laut Konzept bei ATR=0.40 einem Trigger bei $0.24-Move. Das entspricht dem bisherigen 1.5%-Verhalten bei einem $16-Instrument (1.5% × $16 = $0.24). Rueckwaertskompatibel.

## Offene Punkte

Keine.

## ChatGPT-Sparring

**Datum:** 2026-03-03
**Prompt-Thema:** Edge Cases und fehlende Tests fuer `EventDrivenSamplingPolicy`

### Identifizierte Gaps (eingearbeitet)

ChatGPT identifizierte folgende fehlende Test-Faelle, die alle eingearbeitet wurden:

**1. `thresholdAtrFactor=0.0` mit `move=0` triggert**
- `0.0 * ATR = 0.0`, `absoluteMove >= 0.0` ist immer true fuer valide Preise
- Test hinzugefuegt: `shouldSample_zeroThresholdAndZeroMove_triggers`
- Test hinzugefuegt: `shouldSample_zeroThresholdAndPositiveMove_triggers`
- Semantik dokumentiert: factor=0 bedeutet "trigger on any bar"

**2. Multi-bar decisionBars: nur letzter Bar entscheidet**
- Nicht getestet: mehrere decisionBars, davon nur der letzte relevant
- Tests hinzugefuegt: `shouldSample_multipleDecisionBars_onlyLastBarUsed_lastTriggering`
- Tests hinzugefuegt: `shouldSample_multipleDecisionBars_onlyLastBarUsed_lastNotTriggering`

**3. `infiniteClosePrice` fehlte**
- Nur `infiniteOpenPrice` war getestet; `infiniteClosePrice` fehlte
- Test hinzugefuegt: `shouldSample_infiniteClosePrice_returnsFalse`

**4. Gap-Up / Gap-Down True Range Korrektheit**
- Keine expliziten Tests fuer den Fall, dass TR durch `|high-prevClose|` oder `|low-prevClose|` dominiert wird (nicht durch `high-low`)
- Tests hinzugefuegt: `computeAtr_gapUpBar_trDrivenByHighMinusPrevClose`
- Tests hinzugefuegt: `computeAtr_gapDownBar_trDrivenByLowMinusPrevClose`

**5. Gemischte NaN/valide Bars im ATR-Fenster**
- Nur "alle NaN" getestet; nicht: teilweise NaN mit skip-and-continue
- Test hinzugefuegt: `computeAtr_mixedNanAndValidBars_usesOnlyValidTrs`

**6. `barCount=14` liefert 13 TRs (best-effort Verhalten)**
- startIndex = max(1, 14-14) = 1, Schleife i=1..13 → 13 TR-Werte, nicht 14
- Test hinzugefuegt: `computeAtr_exactlyFourteenBars_returnsThirteenTrAverage`
- Verhalten ist korrekt (best-effort); ATR_PERIOD=14 benennt das Ziel-Fenster

**7. Korrektheitsproblem bei nullFiveMinBars-Test (behoben)**
- `createSnapshot(16.00, 16.50, null)` konvertierte `null` intern zu `List.of()`
- Daher testete `shouldSample_nullFiveMinBars_atrIsNan_returnsFalse` nicht wirklich `null`
- Behoben: neuer Helper `createSnapshotWithNullFiveMinBars()` uebergibt `null` direkt

### Verworfene Findings

**`thresholdAtrFactor=0.0` + `ATR=+Infinity` → `0.0 * Inf = NaN` → false:**
- Technisch korrekt (NaN-Vergleich false = sicherer Fallback)
- Extremfall: ATR=+Infinity erfordert Overflow in trSum ueber viele bars mit riesigen TRs
- Kein Guard eingefuehrt: false-Fallback bei ATR=NaN ist definiertes Verhalten, kein Bug

**Negative close-Preis nicht gefiltert:**
- `open <= 0` wird verworfen, `close <= 0` nicht
- Bewusste Entscheidung: negative close ist bei echten Instrumenten unmoeglich
- Policy ist nicht zustaendig fuer allgemeine Datenvalidierung; kein Guard noetig

**Gesamtergebnis:** 9 neue Tests hinzugefuegt (42 Tests total, vorher 33). Alle 1584 Tests in odin-brain bestehen.

## Gemini-Review

**Datum:** 2026-03-03

### Dimension 1: Code-Qualitaet, Bugs, NaN-Safety

**Eingearbeitete Findings:**

1. **LOG.debug mit String.format immer ausgefuehrt (Performance)**
   - `String.format(...)` wurde ausserhalb des Log-Guards ausgefuehrt — erzeugt unnoetig Garbage pro Snapshot auch bei INFO/WARN-Loglevel
   - Fix: `if (triggers && LOG.isDebugEnabled()) { LOG.debug(...) }`

2. **Kommentar "division guard" war irrefuehrend**
   - `open <= 0.0` ist kein Divisionsschutz (es gibt keine Division durch `open`)
   - Kommentar aktualisiert: "stocks cannot have open <= 0"

**Verworfene Findings:**

- "SMA vs. Wilder's RMA" — Dimension 3 bestaetigt: SMA ist fuer den Intraday-Kontext bewusst besser (schnelle Adaption, kein langes Gedaechtnis nach Spikes). SMA bleibt.
- "current.close() NaN propagiert als prevClose" — kein echtes Problem: der Folge-Bar (der NaN-prevClose bekommt) wird korrekt durch `!Double.isFinite(prevClose)` gefiltert (skip, not abort). Verhalten ist korrekt dokumentiert.

### Dimension 2: Konzepttreue

**Bestaetigt:**
- `|close - open|` aus `decisionBars.getLast()` ist konzeptkonform (Entscheidungs-Bar-Bewegung)
- `fiveMinBars` fuer ATR ist korrekt — Interface bleibt unveraendert, kein Zugriff auf `IndicatorResult`
- NaN-Fallback (< 2 Bars → false) ist konzeptkonform
- Default-Faktor 0.6 ist korrekt (rueckwaertskompatibel zur 1.5%-Schwelle)

**Eingearbeitet:**
- JavaDoc fuer `computeAtr` erklaert jetzt explizit, warum SMA statt RMA verwendet wird (schnelle Regime-Adaption im Intraday-Kontext)

### Dimension 3: Praxis und Trading-Szenarien

**Eingearbeitete Erkenntnisse:**

- **SMA besser als RMA fuer Intraday bestaetigt**: Mit ~78 5m-Bars/Session reagiert SMA sofort auf Regime-Wechsel (z.B. volatile Eroeffnung → ruhiger Mittagshandel). Wilders RMA wuerde den Schwellenwert stundenlang nach einem Spike kuenstlich hoch halten.

**Identifizierte Systemrisiken (als neue Design-Entscheidungen dokumentiert):**

**Verworfene Findings (out of scope oder akzeptiertes Verhalten):**

- **`|close - open|` misst nur Kerzenkoerper, nicht True Range (Pin-Bar-Blindheit)**
  - Bei News-Spikes: grosse Wick, kleiner Koerper → Trigger bleibt stumm obwohl Markt volatil
  - Konzept definiert explizit `|close - open|`. Aenderung wuerde neues Konzept erfordern.
  - DEFER: als Future-Enhancement fuer ODIN-098 oder spaetere Analyse

- **ATR-Warmup = 70 Minuten ohne Trigger (kritische Eroeffnungsphase wird verpasst)**
  - 14 × 5 Minuten = 70 Minuten Warmup; die volatilste Phase des Tages faellt weg
  - Loesung: Warmup mit Vortagesdaten (RTH/ETH) befuellen
  - DEFER: erfordert Pipeline-Aenderung (Datenbeschaffung). Bleibt als Verbesserung offen.

- **Over-Triggering in Grind-Maerkten, Under-Triggering in Choppy-Maerkten**
  - Grind: ATR sinkt, Schwelle sinkt, kleine Koerper reichen → zu viele LLM-Calls
  - Choppy: ATR hoch (durch Wicks), Schwelle hoch, Koerper klein → zu wenige LLM-Calls
  - Strukturelles Problem des `|close - open|`-Ansatzes. Akzeptiertes Verhalten in ODIN-097.
  - DEFER: Pin-Bar-Erkennung als separate Signal-Dimension

### Neuer Abschnitt: Bekannte Einschraenkungen

Diese Einschraenkungen sind bewusst akzeptiert und nicht Gegenstand von ODIN-097:

1. **Eroeffnungsphase ohne ATR-Warmup**: Die ersten 70 Minuten nach Session-Start liefern `false` (NaN-ATR-Fallback). Loesung: Warmup-Buffer mit Vortagesdaten befuellen (zukuenftige Pipeline-Aufgabe).
2. **Pin-Bar-Blindheit**: Starke Wick-Bewegungen (z.B. News-Spike) werden nicht erkannt, weil der Trigger auf `|close - open|` statt High-Low-Range basiert.
3. **Grind-Market-Sensitivitaet**: In niedrig-volatilen Trend-Maerkten koennte der Trigger zu haeufig feuern, da ATR und damit der Schwellenwert sinken.
