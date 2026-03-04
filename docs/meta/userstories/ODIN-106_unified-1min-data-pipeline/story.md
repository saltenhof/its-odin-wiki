## Kontext

ODIN hat aktuell zwei getrennte Datenpfade fuer Bar-Downloads: Der `DataController` laedt sowohl 5-Minuten- als auch 1-Minuten-Bars herunter, waehrend der `BacktestExecutionService` nur das konfigurierte Intervall laedt (typischerweise 1-Minute). Diese Dualitaet fuehrt zu inkonsistenten Datenbestaenden — je nachdem welcher Pfad die Daten geladen hat, sind 5-Minuten-Bars vorhanden oder nicht. Das verursacht Folgefehler bei der Chart-KPI-Berechnung (ODIN-107). Diese Story konsolidiert beide Pfade zu einer einzigen, einheitlichen Data Pipeline, die ausschliesslich 1-Minuten-Bars herunterlaed. Hoehere Zeitrahmen (3m, 5m, 15m) werden on-the-fly aus den 1-Minuten-Bars aggregiert, nie gespeichert.

## Modul
odin-app, odin-backtest, odin-persistence

## Abhaengigkeiten
Keine

## Geschaetzter Umfang
M

---

## Scope

### In Scope
- **Einheitlicher Download-Pfad:** `DataDownloadService.ensureDataAvailable()` wird der einzige Einstiegspunkt fuer alle Daten-Downloads (manuell via UI und automatisch via Backtest)
- **Nur 1-Minuten-Bars:** Die Pipeline laedt ausschliesslich 1-Minuten-Bars herunter, unabhaengig vom Aufrufer oder Modus
- **BarInterval-Parameter entfernen:** `ensureDataAvailable()` benoetigt keinen `BarInterval`-Parameter mehr (immer ONE_MINUTE)
- **DataController vereinfachen:** Entfernung des separaten 5-Minuten-Downloads; nur noch 1-Minuten-Download
- **DB-Migration:** CHECK-Constraint auf `intraday_bar.bar_interval_sec` auf `(60)` reduzieren; bestehende 5-Minuten- und 3-Minuten-Rows loeschen
- **Availability-Check korrigieren:** `DataController.checkAvailability()` prueft aktuell nur 5m-Bars — umstellen auf 1m

### Out of Scope
- On-the-fly Bar-Aggregation fuer Indikatoren (ODIN-107)
- Indicator Write Path (ODIN-108)
- MarketDataLevel pro Backtest (ODIN-109)
- Aenderungen am YahooDataFetcher-Interface (unterstuetzt bereits 1-min)
- Aenderungen an L1/Databento-Pfad (laedt keine OHLCV-Bars)

## Akzeptanzkriterien

- [ ] `DataDownloadService.ensureDataAvailable()` hat keinen `BarInterval`-Parameter mehr — laedt immer 1-Minuten-Bars
- [ ] `DataController.downloadData()` laedt nur noch 1-Minuten-Bars herunter (kein 5-Minuten-Download mehr)
- [ ] `DataController.checkAvailability()` prueft Verfuegbarkeit anhand von 1-Minuten-Bars (nicht 5-Minuten)
- [ ] `BacktestExecutionService.downloadMissingData()` nutzt denselben vereinfachten `ensureDataAvailable()`-Aufruf
- [ ] DB-Migration V038 entfernt bestehende 3-Minuten- und 5-Minuten-Rows aus `intraday_bar`
- [ ] DB-Migration V038 aendert CHECK-Constraint auf `bar_interval_sec IN (60)`
- [ ] Neuer manueller Download via UI erzeugt nur noch 1-Minuten-Eintraege in `intraday_bar`
- [ ] Neuer Backtest erzeugt nur noch 1-Minuten-Eintraege in `intraday_bar`
- [ ] Unit-Tests fuer die vereinfachte `DataDownloadService`-Signatur
- [ ] Integrationstest: manueller Download + Backtest nutzen denselben Pfad

## Technische Details

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Aenderung |
|--------|-------|-----------|
| `DataDownloadService` | odin-backtest | `ensureDataAvailable()`: `BarInterval`-Parameter entfernen, intern immer `ONE_MINUTE_SEC = 60` verwenden. `ensureL0DataAvailable()` ebenfalls vereinfachen. |
| `DataController` | odin-app | `DEFAULT_BAR_INTERVAL` (5m) und `CHART_BAR_INTERVAL` (1m) Konstanten entfernen. `downloadData()` ruft nur noch einen `ensureDataAvailable()`-Aufruf mit immer 1-Minuten auf. `checkAvailability()` prueft 1-Minuten-Bars. |
| `BacktestExecutionService` | odin-app | `downloadMissingData()` ruft `ensureDataAvailable()` ohne BarInterval auf. |
| `BacktestController` | odin-app | `DEFAULT_BAR_INTERVAL` Konstante bleibt (BacktestRunEntity braucht sie), wird aber nur noch fuer Entity-Konstruktion verwendet. |

### Neue Klassen

Keine

### Konfiguration

Keine Aenderung — bestehende `odin.data.market-data-level` Property steuert weiterhin den Provider (Yahoo vs. Databento).

### DB-Migration V038

```sql
-- V038: Consolidate to 1-minute-only bar storage.
-- Higher timeframes (3m, 5m, 15m) are computed on-the-fly from 1-minute bars.

-- Remove existing 3-minute and 5-minute bars
DELETE FROM odin.intraday_bar WHERE bar_interval_sec IN (180, 300);

-- Tighten CHECK constraint to only allow 1-minute bars
ALTER TABLE odin.intraday_bar DROP CONSTRAINT intraday_bar_interval_ck;
ALTER TABLE odin.intraday_bar ADD CONSTRAINT intraday_bar_interval_ck CHECK (bar_interval_sec = 60);
```

## Konzept-Referenzen
- `docs/concept/09-backtesting-evaluation.md` — Backtest-Datenpfad, DataDownloadService
- ODIN-101 (Konzept: Live-Datenstreaming) — definiert die Zielarchitektur mit einheitlichem Datenpfad

## Guardrail-Referenzen
- `CLAUDE.md` — Backend Coding Rules: Records fuer DTOs, keine Magic Numbers, JavaDoc auf allen public Klassen
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflichten

## Notizen fuer den Implementierer

1. **YahooDataFetcher** unterstuetzt weiterhin den `intervalSeconds`-Parameter (60 und 300). Die 300er-Unterstuetzung wird NICHT entfernt — sie schadet nicht und koennte fuer kuenftige Anwendungsfaelle nuetzlich sein. Es wird nur nicht mehr aufgerufen.
2. **BarInterval-Enum** behaelt alle drei Werte (ONE_MINUTE, THREE_MINUTES, FIVE_MINUTES). Das Enum wird fuer Aggregationsziele weiterhin benoetigt (z.B. in BarAggregationService, ChartInterval). Nur die Verwendung fuer DB-Speicherung entfaellt.
3. **Migration V038** loescht bestehende 3m/5m-Bars. Das ist irreversibel, aber akzeptabel — sie koennen jederzeit aus 1-Minuten-Bars neu berechnet werden.
4. **IntradayBarQueryService** bleibt unveraendert — Abfragen mit `bar_interval_sec=300` liefern nach der Migration einfach leere Resultate. Die Umstellung der Aufrufer erfolgt in ODIN-107.
5. **L1/Databento-Pfad** ist nicht betroffen — Databento liefert Quote-Ticks/Trade-Ticks, keine OHLCV-Bars. Der Pfad geht durch `ensureL1DataAvailable()` und speichert in `quote_tick`/`trade_tick`.

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `DataDownloadService` (neue Signatur)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Download-Pfad erzeugt nur 1-Minuten-Bars in DB
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klasse(n), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch ChatGPT — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Implementierungsfehler, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Implementierung vs. Konzept)
- [ ] Dimension 3: Praxis-Review (unbehandelte reale Szenarien)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten (Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, ChatGPT-Review)

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`
