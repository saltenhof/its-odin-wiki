# Protokoll: ODIN-092 -- Databento CSV Parser (Quote + Trade Records)

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

### 1. Nullable BBO-Felder
Die BBO-Felder (bidPx, askPx, bidSz, askSz, bidCt, askCt) sind als `Long`/`Integer` (Wrapper-Typen) definiert, nicht als Primitive. Grund: Databento liefert bei Clear-Events (action=R) leere BBO-Felder. Die DB-Spalten sind nullable (siehe V033). Ausserdem verwendet Databento Long.MAX_VALUE und Integer.MAX_VALUE als Sentinel-Werte fuer "nicht definiert", die ebenfalls auf null gemappt werden.

### 2. Nanosekunden-Konvertierung
Exakt nach Konzeptdokument 003-api-client-design.md Abschnitt 3.4: `ts_event_ns` wird als `long` beibehalten, `Instant` wird auf Mikrosekunden-Praezision gerundet (Sub-Mikrosekunden-Digits abgeschnitten). Dual-Storage: BIGINT fuer exaktes Ordering, TIMESTAMPTZ fuer TimescaleDB-Queries. Verwendet `Math.floorDiv`/`Math.floorMod` fuer korrekte Handhabung hypothetischer negativer Timestamps (Gemini-Finding).

### 3. Trading-Day-Ableitung
Einfache Regel aus dem Issue: `tsEvent.atZone("America/New_York").toLocalDate()`. Events waehrend Pre-Market (04:00 ET) bis After-Hours (20:00 ET) gehoeren zum jeweiligen Kalendertag in ET. Kein spezielles Handling fuer After-Midnight-Events noetig, da Databento nur Events innerhalb der Session liefert.

### 4. Exchange-Ableitung aus Dataset-ID
`XNAS.ITCH` -> `XNAS` (Split am ersten Punkt). Multi-Dot IDs (`XNAS.ITCH.V1`) extrahieren korrekt den ersten Segment. Null/Blank -> `UNKNOWN`. Wird im Konstruktor einmalig extrahiert und fuer alle Lines wiederverwendet.

### 5. Fehlertoleranz
Jede `parseQuoteLine`/`parseTradeLine`-Methode faengt alle Exceptions, loggt auf WARN-Level mit truncated line, und gibt null zurueck. Batch-Methoden (`parseQuoteBatch`/`parseTradeBatch`) liefern `CsvParseResult` mit Parsed/Skipped-Zaehler, um Silent Data Loss zu erkennen (Gemini-Finding).

### 6. Parser ist Pro-Pipeline (kein Spring Bean)
Gemaess Modul-Guardrails: odin-data ist ein Pro-Pipeline-Modul. DatabentoCsvParser ist ein plain Java POJO ohne Spring-Annotations. Wird vom aufrufenden Service (kuenftige Story) instanziiert.

### 7. Preise als long (Fixed-Point 1e-9)
Preise werden direkt als `Long.parseLong()` aus dem CSV gelesen. Keine Float/Double-Konvertierung. Konsistent mit den BIGINT-Spalten in den Tick-Tables (V033, V034).

### 8. BOM- und CR-Handling
UTF-8 BOM (\uFEFF) wird beim Header-Check und beim Zeilen-Split entfernt. Trailing \r (Windows-Zeilenende) wird beim Split entfernt. Beides aus ChatGPT-Sparring als realistische Edge Cases identifiziert.

## Offene Punkte

### Aus Gemini Dimension 3 (notiert, kein Handlungsbedarf fuer diese Story):
- **Consolidated Datasets:** DBEQ.MAX-Datasets wuerden "DBEQ" als Exchange liefern statt des tatsaechlichen Execution Venue. Fuer ODIN aktuell nicht relevant (nur XNAS.ITCH).
- **Timezone-Inflexibilitaet:** Trading-Day-Ableitung hartcodiert auf America/New_York. Bei Expansion auf Crypto/EU-Equities muesste dies parametrisiert werden.
- **Zero/Negative Prices:** Parser validiert keine Business Rules. Downstream-Consumer muessen Preis-Sanitaetschecks implementieren.
- **Out-of-Order Ticks:** Parser garantiert keine chronologische Reihenfolge. Sortierung nach ts_event muss ggf. downstream erfolgen.

## ChatGPT-Sparring

### Vorgeschlagene Edge Cases (bewertet und umgesetzt):
1. **Windows \r\n Line Endings** -> UMGESETZT: Parser strippt trailing \r, Test `parseQuoteLine_windowsLineEnding_symbolTrimmed` hinzugefuegt
2. **UTF-8 BOM** -> UMGESETZT: BOM-Handling in isHeaderLine und splitCsvLine, Tests fuer BOM-Header hinzugefuegt
3. **DST Transitions (Spring Forward / Fall Back)** -> UMGESETZT: Tests `deriveTradingDay_dstSpringForward_correctDay` und `deriveTradingDay_dstFallBack_correctDay`
4. **ts_event vs ts_recv Midnight Crossing** -> UMGESETZT: Test `deriveTradingDay_tsEventBeforeMidnightEt_tsRecvAfterMidnightEt`
5. **Multi-Dot Dataset IDs** -> UMGESETZT: Test `extractExchange_multiDotDatasetId_returnsFirstSegment`
6. **Extra Columns (>18)** -> UMGESETZT: Test `parseQuoteLine_extraColumns_toleratesGracefully`
7. **Header Mid-File** -> UMGESETZT: Test `parseQuoteLines_headerMidFile_skipsHeader`
8. **Nanosecond Boundary (999/001 mod 1000)** -> UMGESETZT: Tests `nanosToInstant_maxTruncation_999ModNanos` und `nanosToInstant_minTruncation_001ModNanos`
9. **Sentinel MAX_VALUE-1** -> UMGESETZT: Test `parseQuoteLine_sentinelMaxValueMinusOne_parsesAsRealValue`
10. **Partial BBO Null** -> UMGESETZT: Test `parseQuoteLine_partialBboNull_mixedNullAndValues`
11. **NaN in Numeric Field** -> UMGESETZT: Test `parseQuoteLine_nanInNumericField_returnsNull`

### Nicht umgesetzt (begruendet):
- **Quoted Fields:** Databento CSV verwendet keine quoted Fields (rein numerische Daten). Overhead nicht gerechtfertigt.
- **Parallel Parsing Test:** Parser ist stateless, thread-safety durch Konstruktor-finale Felder garantiert. Separater Concurrency-Test waere redundant.
- **FastCSV/Zero-Alloc:** Vorzeitige Optimierung. ODIN-Volumina (100k-300k Lines) sind nicht der Bottleneck. Build first, measure, then optimize.

## Gemini-Review (Runde 2 — 3 separate Aufrufe, owner=ODIN-092-worker, 2026-03-01)

### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
**Prompt:** Code-level bugs, edge cases, thread safety, error handling, type safety.
**Findings aus Runde 2:**
1. BUG: successRatio-Berechnung falsch — Header zaehlt als `skipped`, deshalb erreicht successRatio() nie 1.0 fuer normale CSV-Dateien -> UMGESETZT. Batch-Loop filtert jetzt header/blank explizit heraus (continue), nur echte Parse-Fehler zaehlen als skipped. CsvParseResult.successRatio() berechnet nun parsed/(parsed+skipped) statt parsed/totalLines.
2. BUG: exception.getMessage() kann null bei NullPointerException sein -> Log-Eintrag wuerde nur "null" zeigen -> UMGESETZT. Auf exception.toString() umgestellt (liefert immer "ClassName: message" oder "ClassName").
3. MINOR: Timezone-Hardcoding America/New_York -> BELASSEN/NOTIERT. Nur US Equities im Scope, offen fuer spaetere Erweiterung.
4. THREAD-SAFETY: Excellent — alle Felder final, alle statischen Konstanten immutable. Kein Handlungsbedarf.
5. TYPE-SAFETY: Wrapper-Typen fuer nullable BBO-Felder korrekt und vollstaendig.

### Dimension 2: Konzepttreue-Review
**Prompt:** Feldvollstaendigkeit, Typen, Timestamp-Konvertierung, Trading-Day-Ableitung vs. Konzeptdokument.
**Findings aus Runde 2:**
1. MATCH: MBP-1 18-Spalten-Layout exakt implementiert (MBP1_COL_* Constants). instrument_id und depth korrekt aus Record ausgelassen (wie im Konzept vorgesehen).
2. MATCH: ParsedTradeTick-Felder exakt wie im Konzeptdokument 003 Abschnitt 3.3 definiert.
3. POSITIVE DEVIATION: Wrapper-Typen (Long/Integer) fuer BBO-Felder statt Primitive im Konzept-Entwurf -> AKZEPTIERT. Korrekte Behandlung von Sentinel-Werten und nullable DB-Spalten. Explizit begruendet.
4. MATCH: Timestamp-Konvertierung korrekt — Math.floorDiv/floorMod ist Verbesserung gegenueber Konzept-Pseudo-Code, erhaelt die volle Spezifikationserfullung.
5. MATCH: Trading-Day-Ableitung via America/New_York korrekt.
6. POSITIVE DEVIATION: CsvParseResult-Wrapper statt einfacher List -> AKZEPTIERT. Enhancement fuer Silent-Data-Loss-Detektion.
7. VERWORFEN (Gemini-Behauptung aus Runde 1 bestaetigt als falsch): Trades haben nur 13-14 Spalten -> Falsch. 18 Spalten korrekt, Issue definiert explizit.

### Dimension 3: Praxis-Review
**Prompt:** Performance 100k+ Lines, Memory/GC, Error-Handling-Resilience, Logging, Testbarkeit.
**Findings aus Runde 2:**
1. GC-DRUCK (Performance): String.split+trim erzeugt ~4M temporaere Objekte pro 100k Lines. -> DEFERRED gemaess CLAUDE.md "Build first, measure, then optimize". ODIN-Volumina (100k-300k) sind kein Hot-Path-Engpass.
2. ERROR-HANDLING: exception.getMessage() kann null sein -> UMGESETZT (siehe Dimension 1, Bug 2).
3. LOG-FLOODING: Bei korrupter Datei koennte der Parser 100k WARN-Logs produzieren -> NOTIERT als offener Punkt. Circuit-Breaker/Log-Sampling als zukuenftige Optimierung. Scope dieser Story: Zeilenfehlertoleranz, kein Rate-Limiting.
4. TESTABILITY: Excellent — stateless POJO, package-private Hilfsmethoden direkt testbar ohne Mocking.
5. CACHED TRADING-DAY: 99.9% der Zeilen gehoeren zum selben Tag, Caching wuerde 100k LocalDate-Allokationen sparen -> DEFERRED. Vorzeitige Optimierung fuer aktuellen Use Case.
6. HARDCODED TIMEZONE -> NOTIERT als offener Punkt (unveraendert aus Runde 1).
