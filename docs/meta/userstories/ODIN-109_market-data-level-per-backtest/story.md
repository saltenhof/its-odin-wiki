## Kontext

Das `MarketDataLevel` (L0_NO_EXTENDED_VOLUME, L0_FULL_VOLUME, L1_QUOTES, L2_DEPTH) ist aktuell eine **globale** Konfiguration (`odin.data.market-data-level` in application.properties), die beim Boot-Zeitpunkt verdrahtet wird. Das bedeutet: Alle Backtests laufen mit demselben Data-Level, und ein Wechsel erfordert Backend-Neustart. Fuer A/B-Vergleiche (z.B. "wie performt die Strategie mit L0-Yahoo vs. L1-Databento?") ist das unpraktisch.

Diese Story macht das MarketDataLevel **pro Backtest waehlbar**: Die UI bietet eine Dropdown-Box bei der Backtest-Erstellung, das Backend speichert das Level im `BacktestRunEntity` und routet den Daten-Download entsprechend. Die Provider-Verdrahtung (welcher Adapter fuer welches Level) bleibt Spring-Boot-Time — das Routing pro Backtest waehlt nur aus, welchen bereits verdrahteten Provider es nutzt.

## Modul
odin-api, odin-backtest, odin-app, odin-persistence, its-odin-ui

## Abhaengigkeiten
ODIN-106 (Unified 1-Min-Only Data Pipeline)

## Geschaetzter Umfang
M

---

## Scope

### In Scope
- **BacktestRunEntity:** Neues Feld `marketDataLevel` (Enum `MarketDataLevel`) mit Default `L0_NO_EXTENDED_VOLUME`
- **BacktestRequest DTO:** Neues optionales Feld `marketDataLevel` (Default: `L0_NO_EXTENDED_VOLUME`)
- **API:** `POST /api/v1/backtests` akzeptiert `marketDataLevel` im Request-Body
- **DB-Migration:** Neue Spalte `market_data_level` in `odin.backtest_run` Tabelle
- **DataDownloadService:** Neuer Parameter `MarketDataLevel` in `ensureDataAvailable()` — bestimmt den Provider statt der globalen Config
- **Provider-Registry:** Spring-Boot-Time-Verdrahtung aller verfuegbaren Provider; Runtime-Routing per Level
- **Frontend:** Dropdown-Box im Backtest-Erstellungsformular mit allen vier MarketDataLevel-Werten, L0_NO_EXTENDED_VOLUME vorausgewaehlt
- **Backend-Response:** `marketDataLevel` im Backtest-Detail-Response zurueckliefern

### Out of Scope
- Aenderung der Provider-Implementierungen (Yahoo, Databento) — bleiben unveraendert
- Live-Trading-Modus: MarketDataLevel bleibt dort global (kein Run-Level noetig)
- Databento API-Key-Management (bestehend via .env)
- Kosten-Warnung bei L1/L2-Auswahl (sinnvoll, aber separate Verbesserung)

## Akzeptanzkriterien

- [ ] `BacktestRunEntity` hat ein Feld `marketDataLevel` vom Typ `MarketDataLevel` mit Default `L0_NO_EXTENDED_VOLUME`
- [ ] `BacktestRequest` akzeptiert optionales `marketDataLevel` (null = Default L0_NO_EXTENDED_VOLUME)
- [ ] DB-Migration fuegt Spalte `market_data_level TEXT NOT NULL DEFAULT 'L0_NO_EXTENDED_VOLUME'` zur Tabelle `odin.backtest_run` hinzu
- [ ] `DataDownloadService.ensureDataAvailable()` nutzt das uebergebene `MarketDataLevel` statt der globalen Config
- [ ] Ein Backtest mit `L0_NO_EXTENDED_VOLUME` laedt Daten via Yahoo Finance
- [ ] Ein Backtest mit `L1_QUOTES` laedt Daten via Databento (sofern API-Key konfiguriert)
- [ ] Wenn L1/L2 gewaehlt aber kein Databento-Provider verfuegbar: klarer Fehler beim Backtest-Start (nicht erst beim Download)
- [ ] Frontend zeigt Dropdown mit 4 Optionen, huebsch formatiert (z.B. "L0 — Yahoo (Standard)", "L1 — Databento Quotes")
- [ ] Backtest-Detail-Ansicht zeigt das genutzte MarketDataLevel an
- [ ] Unit-Tests fuer Provider-Routing basierend auf MarketDataLevel
- [ ] Integrationstest: Backtest-Erstellung mit explizitem MarketDataLevel

## Technische Details

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Aenderung |
|--------|-------|-----------|
| `BacktestRunEntity` | odin-backtest | Neues Feld `marketDataLevel` (`MarketDataLevel`, `@Column(name = "market_data_level")`), Default `L0_NO_EXTENDED_VOLUME`. JPA `@Enumerated(EnumType.STRING)`. |
| `BacktestRequest` | odin-app | Neues optionales Feld `MarketDataLevel marketDataLevel` (nullable, Default im Controller). |
| `BacktestController` | odin-app | `startBacktest()`: `marketDataLevel` aus Request uebernehmen, an Entity und `BacktestExecutionService` weitergeben. Validierung: wenn L1/L2 gewaehlt, pruefen ob Provider verfuegbar. |
| `BacktestExecutionService` | odin-app | `downloadMissingData()`: `MarketDataLevel` aus Entity an `DataDownloadService` weiterreichen. |
| `DataDownloadService` | odin-backtest | `ensureDataAvailable()`: Neuer Parameter `MarketDataLevel level` (ersetzt internen `dataProperties.marketDataLevel()`-Aufruf). |
| `BacktestDetailResponse` (o.ae.) | odin-app | `marketDataLevel` im Response-DTO mitliefern. |

### Neue Klassen

Keine (Provider-Registry kann ueber bestehende Spring-Bean-Mechanismen abgebildet werden — `Map<MarketDataLevel, HistoricalDataProvider>` oder `Optional<HistoricalDataProvider>` mit Conditional Beans).

### Konfiguration

Keine neue Property. Die globale `odin.data.market-data-level` bleibt als Default fuer Live-Trading bestehen. Per-Backtest-Level uebersteuert sie nur fuer den jeweiligen Run.

### DB-Migration V039

```sql
-- V039: Add market_data_level to backtest_run for per-backtest provider routing.
ALTER TABLE odin.backtest_run
    ADD COLUMN market_data_level TEXT NOT NULL DEFAULT 'L0_NO_EXTENDED_VOLUME';
```

### Frontend-Aenderungen

| Datei | Aenderung |
|-------|-----------|
| Backtest-Erstellungsformular | Neue Dropdown-Komponente fuer `marketDataLevel`. Optionen: `L0 — Yahoo Finance (Standard)`, `L0 — Yahoo inkl. Extended Volume`, `L1 — Databento Quotes`, `L2 — Databento Depth`. Default: erste Option. |
| Backtest-Request-Type | `marketDataLevel?: string` Feld hinzufuegen. |
| Backtest-Detail-Ansicht | `marketDataLevel` anzeigen (z.B. als Badge oder Info-Zeile). |

## Konzept-Referenzen
- `MarketDataLevel` Enum (`odin-api/.../model/MarketDataLevel.java`) — Vier Werte: L0_NO_EXTENDED_VOLUME, L0_FULL_VOLUME, L1_QUOTES, L2_DEPTH
- `DataWiringConfig` (`odin-app/.../config/DataWiringConfig.java`) — Aktuelles Boot-Time-Wiring der Provider via `@ConditionalOnProperty`
- ODIN-106 — Voraussetzung: Einheitlicher Download-Pfad
- ODIN-101 (Konzept: Live-Datenstreaming) — Backtest-Konfiguration wird spaeter im Streaming-Kontext relevant

## Guardrail-Referenzen
- `CLAUDE.md` — Backend Coding Rules: ENUM statt String, Records fuer DTOs, `@ConfigurationProperties` als Record
- `CLAUDE.md` — Frontend Architecture: TypeScript strict, union types, CSS Modules
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

## Notizen fuer den Implementierer

1. **Provider-Verdrahtung bleibt Boot-Time:** Die Spring-Beans fuer Yahoo und Databento werden weiterhin beim Start verdrahtet. Das per-Backtest `MarketDataLevel` steuert nur, WELCHEN bereits vorhandenen Provider der `DataDownloadService` nutzt. Kein Runtime-Bean-Wiring.
2. **L1/L2 ohne Databento:** Wenn ein User L1 waehlt aber kein Databento-API-Key konfiguriert ist (kein `DatabentoBatchService`-Bean vorhanden), soll der Backtest **sofort** fehlschlagen — nicht erst nach 30 Minuten beim Download-Versuch. Validierung im Controller oder Service.
3. **Globale Config als Fallback:** Die globale `odin.data.market-data-level` Property bleibt bestehen und wird weiterhin fuer Live-Trading genutzt. Im Backtest-Kontext hat das Entity-Feld Vorrang.
4. **MarketDataLevel-Enum hat `hasQuotes()` Methode:** Nutzen fuer L0/L1-Routing statt String-Vergleiche.
5. **Frontend-Formatierung:** Die Enum-Werte sind technisch (L0_NO_EXTENDED_VOLUME). Im Frontend huebsch formatieren mit Beschreibung. Beispiel: "L0 Standard — Yahoo Finance (kostenlos)" / "L1 Quotes — Databento (kostenpflichtig)".
6. **Bestehende Backtests:** Die Migration setzt Default `L0_NO_EXTENDED_VOLUME` fuer alle existierenden Runs — das ist korrekt, da alle bisherigen Runs mit Yahoo liefen.

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
- [ ] Unit-Tests fuer Provider-Routing in `DataDownloadService`
- [ ] Unit-Tests fuer `BacktestController` Validierung (L1 ohne Provider → Fehler)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Backtest mit L0 → Yahoo-Download
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet
- [ ] Edge Cases identifiziert (L1 ohne Key, Level-Wechsel zwischen Runs, bestehende Daten anderer Level)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch ChatGPT — Drei Dimensionen
- [ ] Dimension 1: Code-Review
- [ ] Dimension 2: Konzepttreue-Review
- [ ] Dimension 3: Praxis-Review
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`
