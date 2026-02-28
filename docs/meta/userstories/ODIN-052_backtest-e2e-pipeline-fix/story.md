# Story: ODIN-052 — Backtest End-to-End Pipeline: Simulation & Ergebnisdarstellung

## Metadaten

| Feld | Wert |
|------|------|
| **Titel** | Backtest End-to-End Pipeline: Simulation & Ergebnisdarstellung |
| **Modul** | odin-backtest (primär), odin-data (DataDownload), odin-app (Controller) |
| **Phase** | 1 |
| **Abhängigkeiten** | Keine |
| **Geschätzter Umfang** | L |

---

## Kontext

Wenn ein Nutzer über die UI einen Backtest für ein Instrument (z.B. IREN, Nasdaq) für einen konkreten Handelstag auslöst, werden aktuell keine Ergebnisse angezeigt. Der Backtest-Lauf erreicht entweder nicht den Status COMPLETED, oder er läuft durch ohne gehandelte Tage (`completedDays = 0`). Die wahrscheinlichste Ursache: `BacktestRunner.runSingleDay()` findet keine Bars in der Datenbank und überspringt den Tag stillschweigend (`return null`). Die historischen Daten müssen vor dem Lauf per `DataDownloadService.ensureDataAvailable()` von Yahoo Finance beschafft werden — dieser Schritt scheint nicht zuverlässig zu funktionieren oder wird nicht korrekt ausgelöst.

Ziel dieser Story: den vollständigen Backtest-Pfad von der UI-Anfrage bis zur Ergebnisdarstellung zu reparieren und zu verifizieren. Als konkreter Akzeptanztest dient ein Backtest für **IREN (Nasdaq), 23.02.2026**.

---

## Scope

**In Scope:**
- Diagnose: Warum werden keine Ergebnisse für IREN/23.02.2026 angezeigt? Backend-Logs auswerten, Root Cause identifizieren.
- Fix: Historische Bars für den Backtestlauf korrekt beschaffen (Yahoo Finance / `DataDownloadService`). Dabei Nasdaq-Ticker-Konventionen beachten (Yahoo Finance verwendet z.B. `IREN` direkt für Nasdaq-Aktien).
- Fix: Falls `DataDownloadService.ensureDataAvailable()` nicht korrekt in den Backtest-Start-Pfad eingebunden ist, korrigieren.
- Fix: Falls Bars vorhanden sind, aber der `BacktestRunner` oder `SimulationRunner` trotzdem keine Trades produziert, Pipeline-Bugs identifizieren und beheben.
- Fix: Falls die UI `summaryJson`/`dailyResultsJson` empfängt, aber nicht anzeigt, Frontend-Bug beheben.
- Acceptance Test: Nach dem Fix einen echten Backtest für IREN/23.02.2026 triggern. Er muss COMPLETED status erreichen, mindestens einen gehandelten Tag zeigen, und im UI konkrete Metriken (PnL, Trades, Win-Rate) anzeigen.
- Fehlerfall sauber dokumentieren: Falls der Algorithmus an diesem Tag keinen Entry-Signal erzeugt (valider Ausgang), muss das UI dies sauber kommunizieren (z.B. "0 Trades — kein Entry-Signal") statt gar nichts zu zeigen.

**Out of Scope:**
- Neue Backtest-Features (neue Strategievarianten, neue Metriken)
- Live-Daten-Download über IB TWS (nur Yahoo Finance im Backtest-Kontext)
- Performance-Optimierung der Datenbeschaffung (kein Caching, kein Bulk-Download)
- Änderungen an der Trading-Logik selbst (Entry/Exit-Regeln)

---

## Akzeptanzkriterien

- [ ] **AC-1:** Ein Backtest für IREN (Nasdaq) vom 23.02.2026 bis 23.02.2026 kann erfolgreich über die UI gestartet werden.
- [ ] **AC-2:** Der Backtest erreicht den Status `COMPLETED` innerhalb von maximal 5 Minuten.
- [ ] **AC-3:** Die UI zeigt nach Abschluss konkrete Ergebnisse:
  - Entweder: mindestens eine Tageszeile mit gehandeltem Tag und PnL-Angabe
  - Oder: explizit "0 Trades" mit sauber befülltem `summaryJson` (kein Null-Wert)
- [ ] **AC-4:** Bei 0 Trades zeigt die UI eine verständliche Erklärung (kein leerer Bildschirm, kein Spinner der ewig läuft).
- [ ] **AC-5:** Die Backend-Logs zeigen KEINE "No data for {tradingDate}, skipping"-Warnung für 23.02.2026 (= Daten wurden erfolgreich geladen).
- [ ] **AC-6:** `BacktestRunEntity.totalDays` ist ≥ 1 für einen Ein-Tages-Backtest auf einem Handelstag.
- [ ] **AC-7:** Der Backtest-Durchlauf übersteht alle Verification-Checks (`BACKTEST VERIFICATION`) ohne `[FAIL]`.

---

## Technische Details

### Root-Cause-Diagnose (zuerst)

1. Backend starten (SIMULATION-Modus): Port 3300
2. Backtest über REST auslösen: `POST /api/v1/backtests` mit Body `{"name":"iren-test","instruments":["IREN"],"startDate":"2026-02-23","endDate":"2026-02-23","exchange":"NASDAQ"}`
3. Status pollen: `GET /api/v1/backtests/{id}`
4. Backend-Logs auswerten: nach "skipping", "FAIL", "No data", "error", "exception" suchen

### Datenbeschaffung

**Klassen:**
- `DataDownloadService` (odin-data oder odin-backtest): `ensureDataAvailable(instruments, dateRange)` — lädt Yahoo Finance 5m-Bars
- `IntradayBarJdbcRepository.loadDay(tradingDate, instruments, barIntervalSeconds)` — liest aus `odin.intraday_bar`
- `BacktestRunner.runSingleDay(...)` — ruft `barRepository.loadDay()` auf

**Prüfpunkte:**
- Wird `ensureDataAvailable()` VOR dem ersten `loadDay()` aufgerufen?
- Funktioniert der Yahoo-Finance-Download für `IREN` (Nasdaq-Ticker)?
- Werden die geladenen Bars korrekt in die DB persistiert?
- Ergibt der Datums-Check für 23.02.2026 einen US-Handelstag (nicht Wochenende, kein Feiertag)?

**Yahoo Finance Ticker-Konvention für Nasdaq:** Nasdaq-Aktien verwenden das direkte Symbol ohne Suffix (z.B. `IREN` nicht `IREN.O`). Bei Problemen ggf. mit `IREN` und `IREN.NQ` testen.

### Pipeline-Pfad

```
BacktestController.startBacktest()
  → BacktestExecutionService.executeBacktest()
    → DataDownloadService.ensureDataAvailable()   ← muss VOR run() stehen
    → BacktestRunner.run()
      → BacktestRunner.runSingleDay(tradingDate)
        → IntradayBarJdbcRepository.loadDay()     ← muss Bars finden
        → SimulationRunner.run()                   ← simuliert Trading-Tag
      → BacktestRunner.aggregateResults()
        → BacktestRunEntity.summaryJson = ...      ← muss befüllt werden
```

### Frontend-Anzeige

- `BacktestDetailPage`: pollt `GET /api/v1/backtests/{id}` bis `status = COMPLETED`
- Tab "Summary": rendert aus `summaryJson` — Felder: `totalPnl`, `sharpeDaily`, `winRateDaily`, `maxDrawdown`, `totalTrades`
- Tab "Tage": lädt `GET /api/v1/backtests/{id}/days` — zeigt Tages-Ergebnisse
- Falls `summaryJson = null` oder `totalDays = 0`: UI muss das sauber kommunizieren (kein leerer Tab, kein Spinner)

### Fehlerfall-Handling

Falls an einem Tag keine Trades entstehen (gültiger Fall: kein Entry-Signal), muss das UI das sauber darstellen:
- Summary: "0 Trades, PnL: $0.00, 0 gehandelte Tage"
- Tage-Tab: leere Tabelle mit Hinweistext "Kein Entry-Signal"
- Kein Spinner, der dauerhaft läuft

---

## Konzept-Referenzen

| Dokument | Abschnitt |
|----------|-----------|
| `docs/concept/intraday-agent-concept.md` | Kap. 17: Backtesting & Shadow Mode Pipeline (Simulation Flow, Paper-Mode) |
| `docs/backend/architecture/02-realtime-pipeline.md` | Abschnitt Sim-Replay: HistoricalMarketDataFeed-Betrieb |
| `docs/backend/architecture/06-rules-engine.md` | FSM States im Sim-Betrieb: OBSERVING → SEEKING_ENTRY → Transition |

---

## Guardrail-Referenzen

| Guardrail | Pfad |
|-----------|------|
| Modulstruktur | `docs/backend/guardrails/module-structure.md` |
| Entwicklungsprozess | `docs/backend/guardrails/development-process.md` |
| CLAUDE.md | `T:/codebase/its_odin/CLAUDE.md` |

---

## Definition of Done

### 2.1 Code-Qualität
- [ ] Implementierung vollständig gemäß Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-backtest,odin-app`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records für DTOs
- [ ] ENUM statt String für endliche Mengen
- [ ] JavaDoc auf allen neuen/geänderten public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare
- [ ] Code-Sprache: Englisch

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests für neue/geänderte Logik in `*Test`-Klassen
- [ ] Bug-Reproduzierungstest: Test der den Fehler (fehlende Daten → kein Ergebnis) abdeckt

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] `*IntegrationTest` für den Backtest-Durchlauf mit Stub-Daten (ohne echten Yahoo-Download)
- [ ] Test belegt: wenn Bars in DB vorhanden → BacktestRun status = COMPLETED, summaryJson != null

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit: relevante Klassen, Akzeptanzkriterien, vorhandene Tests
- [ ] ChatGPT nach Edge-Cases gefragt (leerer Handelstag, Ticker nicht gefunden, API-Timeout)
- [ ] Relevante Vorschläge umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini (drei Dimensionen)
- [ ] Dimension 1: Code-Review (Bugs, Null-Safety, Ressource-Leaks)
- [ ] Dimension 2: Konzepttreue (passt der Fix zum Backtest-Konzept Kap. 17?)
- [ ] Dimension 3: Praxis-Review (welche Szenarien könnten in der Praxis noch auftreten?)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten erstellt und aktuell gehalten

### 2.8 Abschluss
- [ ] Commit mit aussagekräftiger Message
- [ ] Push auf Remote

---

## Notizen für den Implementierer

- **Zuerst diagnostizieren, dann fixen.** Backend starten, echten Backtest auslösen, Logs lesen. Root Cause muss klar sein bevor Code geändert wird.
- **23.02.2026 ist ein Montag** — das ist ein regulärer US-Handelstag (kein Feiertag). Die Exchange-Prüfung sollte keinen Feiertag erkennen.
- **Yahoo Finance Rate-Limits:** Der Download kann bei Timeouts oder Rate-Limits scheitern. Retry-Logik prüfen. Falls nötig, minimal implementieren (1 Retry, 5s Delay).
- **Kein Live-IB-API-Zugriff im Backtest.** Der Backtest läuft in Simulation. `BrokerSimulator` statt `IbBrokerGateway` wird verwendet.
- **Falls kein Entry-Signal:** Das ist ein valider Ausgang und kein Fehler. Der Algorithmus hat ggf. keinen Setup A oder D erkannt. Die UI muss das sauber darstellen.
- **Ticker-Mapping:** Falls Yahoo Finance ein anderes Ticker-Format braucht als das, was im UI eingegeben wird, muss eine Mapping-Logik her (z.B. Exchange-Suffix).
- **Keine Änderungen an der Trading-Logik** — nur Pipeline-Fix. Falls Entry-Regeln das Problem sind, eskalieren.
