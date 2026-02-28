# ODIN-029: Daily Performance Recording und Reporting

**Modul:** odin-core
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** Keine

---

## Context

Die `DailyPerformanceEntity` existiert bereits im Datenmodell, aber die systematische Berechnung und Speicherung der Tages-Performance fehlt. Das Konzept fordert: Tages-P&L, Win-Rate, Sharpe, Max-Drawdown, und weitere Metriken pro Tag.

## Scope

**In Scope:**
- Neue Klasse `DailyPerformanceRecorder` in odin-core
- Berechnung am Tagesende (EOD): P&L, Win-Rate, Sharpe, MDD, #Trades, #Cycles
- Speicherung in `DailyPerformanceEntity` via Repository
- Aggregation ueber alle Pipelines eines Tages
- Trade-Journal-Daten: Alle Trades des Tages mit Entry/Exit/P&L/Reason

**Out of Scope:**
- Frontend-Darstellung der Daily Performance (separates Frontend-Ticket)
- Historische Backtesting-Reports (das ist odin-brain/Backtesting)
- Real-time Performance-Updates (nur EOD-Snapshot)

## Acceptance Criteria

- [ ] `DailyPerformanceRecorder.recordDailyPerformance()` wird bei EOD aufgerufen
- [ ] Metriken: totalPnl, winRate, lossRate, avgWin, avgLoss, profitFactor, maxDrawdown, tradeCount, cycleCount
- [ ] Daten kommen aus GlobalRiskManager (P&L, Cycles) und TradeRepository (Trades)
- [ ] Speicherung in DailyPerformanceEntity
- [ ] Trade-Journal: Liste aller Trades mit Details (persistiert als JSON in DailyPerformance)
- [ ] Historische Metriken: Kumulierter Sharpe, Rolling Win-Rate

## Technical Details

**Dateien:**
- `odin-core/src/main/java/de/its/odin/core/service/DailyPerformanceRecorder.java` (neue Klasse)
- `odin-core/src/main/java/de/its/odin/core/service/LifecycleManager.java` (EOD-Hook)

**Patterns:**
- `DailyPerformanceRecorder` als Spring-Bean (Singleton) mit Repository-Zugriff
- Trade-Journal als JSON-Blob in `DailyPerformanceEntity` (via `@JdbcTypeCode(JSON)`)
- Metriken-Berechnung vollstaendig in `DailyPerformanceRecorder` (kein Business-Logic im Repository)

**Konfiguration:**
- Keine spezifische Konfiguration erwartet (EOD-Hook bereits in LifecycleManager)

## Concept References

- `docs/concept/10-observability.md` -- Abschnitt 7 "Report-Artefakte"
- `docs/concept/10-observability.md` -- Abschnitt 7.1 "Trade Journal"
- `docs/concept/10-observability.md` -- Abschnitt 7.2 "Daily Report"
- `docs/concept/09-backtesting-evaluation.md` -- Abschnitt 7 "Geometrische Scoring-Metriken"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- "DDD-Modulschnitt: Persistenz" (Entities in Domain-Modulen)
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints
- `T:\codebase\its_odin\CLAUDE.md` -- Instanziierungsmodell, Transaktionsgrenzen

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (z.B. Trade-Journal-Eintraege als Records)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.performance.*` fuer Konfiguration
- [ ] Transaktionsgrenzen: `TransactionTemplate` statt `@Transactional` (Pro-Pipeline-POJOs)

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Metriken-Berechnung (totalPnl, winRate, profitFactor, maxDrawdown)
- [ ] Unit-Tests: Edge Case — 0 Trades (leerer Tag): alle Metriken auf 0/null, kein Fehler
- [ ] Unit-Tests: Edge Case — nur Verlust-Trades: winRate=0, avgWin=0
- [ ] Unit-Tests: Sharpe-Berechnung (annualisiert aus taeglichen Returns)
- [ ] Unit-Tests: Trade-Journal-Serialisierung (korrekte JSON-Struktur)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `GlobalRiskManager` und `TradeRepository`
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `DailyPerformanceRecorder` + echtem `GlobalRiskManager` (mit mehreren simulierten Trades)
- [ ] Integrationstest: EOD-Hook in `LifecycleManager` loest `DailyPerformanceRecorder.recordDailyPerformance()` korrekt aus
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Diese Story hat Datenbankzugriff (`DailyPerformanceEntity` wird gespeichert).

- [ ] Embedded-Postgres-Tests mit **Zonky** (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migrationen werden im Test ausgefuehrt (insbesondere die Migration, die `DailyPerformanceEntity` anlegt)
- [ ] `DailyPerformanceRepository.save()` wird gegen echte Embedded-DB getestet (nicht nur Mock)
- [ ] Test: JSON-Blob des Trade-Journals wird korrekt persistiert und wieder geladen
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `DailyPerformanceRecorder`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was bei negativem Sharpe? Was bei nur einem Trade (kein Durchschnitt berechenbar)? Was bei Datenbankfehler waehrend EOD-Speicherung?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Division-by-Zero bei leeren Trades, korrekte Sharpe-Formel, JSON-Serialisierungsprobleme beim Trade-Journal"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/10-observability.md` (Abschnitt 7) + `docs/concept/09-backtesting-evaluation.md` (Abschnitt 7) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Metrik-Definitionen, Trade-Journal-Felder und Sharpe-Berechnung dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche Performance-Recording-Szenarien koennten in der Praxis auftreten, die im Konzept nicht behandelt sind? (z.B. EOD-Absturz vor Recording-Abschluss, nachtraegliche Korrekturen)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Sharpe-Berechnungsansatz, Trade-Journal-Format)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- DailyPerformanceEntity und Repository existieren bereits (V011 Migration)
- Trade-Journal ist ein JSON-Blob mit allen Trade-Details des Tages
- Sharpe-Berechnung: Annualisiert aus taeglichen Returns (mindestens 5 Tage fuer sinnvollen Wert)
