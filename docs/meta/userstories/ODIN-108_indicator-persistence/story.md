## Kontext

Die `indicator_snapshot`-Tabelle existiert in der Datenbank mit vollstaendigem Schema (EMA, RSI, ATR, Bollinger, ADX, VWAP, AVWAP, VPIN, DMI), der Read-Path ist implementiert (`IndicatorSnapshotRepository` + `IndicatorQueryService.loadPersistedIndicators()`), aber der **Write-Path fehlt komplett** — die Tabelle ist leer. Aktuell werden alle Chart-KPIs on-the-fly neu berechnet, was (a) nicht exakt die Werte reproduziert, die die Pipeline beim Trading gesehen hat, und (b) bei fehlendem Datenmaterial (ODIN-107-Problem) versagt.

Diese Story implementiert einen modus-agnostischen `IndicatorPersister`, der `IndicatorResult`-Objekte von der Pipeline entgegennimmt und in die `indicator_snapshot`-Tabelle schreibt. Der Persister arbeitet identisch in Live-Trading und Backtest — die KPI-Engine berechnet, der Persister schreibt, die UI liest. Klare Trennung der Zustaendigkeiten.

## Modul
odin-brain, odin-core, odin-app

## Abhaengigkeiten
ODIN-107 (On-the-fly Bar-Aggregation fuer Indikatoren)

## Geschaetzter Umfang
M

---

## Scope

### In Scope
- **Neuer Baustein `IndicatorPersister`:** Empfaengt `IndicatorResult`, mappt auf `IndicatorSnapshotEntity`, schreibt via `IndicatorSnapshotRepository`
- **Pipeline-Integration:** An der Stelle in der Pipeline, wo `KpiEngine.onSnapshot()` aufgerufen wird, wird das Ergebnis auch an den `IndicatorPersister` weitergereicht
- **Modus-Agnostik:** Funktioniert identisch in Live-Trading (TradingPipeline) und Backtest (SimulationRunner) — selbe Pipeline, selber Persister
- **Batch-Optimierung:** Buffered Writes (z.B. alle N Bars oder pro Minute flushen) um DB-Last zu minimieren
- **Run-ID-Zuordnung:** Jeder Snapshot erhaelt die `runId` des aktuellen TradingRun/Backtest fuer spaetere Zuordnung

### Out of Scope
- Aenderungen an der KPI-Engine selbst (bleibt reiner Berechner)
- Aenderungen am Read-Path (`IndicatorQueryService.loadPersistedIndicators()` — existiert bereits)
- Live-Streaming der Indicator-Werte an die UI (ODIN-101)
- On-the-fly Fallback-Berechnung entfernen (bleibt als Backup fuer historische Daten ohne persisted Indicators)

## Akzeptanzkriterien

- [ ] Neuer `IndicatorPersister`-Baustein existiert mit klarer Single-Responsibility: empfange `IndicatorResult`, persistiere als `IndicatorSnapshotEntity`
- [ ] Nach einem Backtest-Lauf enthaelt `indicator_snapshot` Eintraege fuer jeden Bar-Zeitpunkt des Runs
- [ ] Nach einem Live-Trading-Tag enthaelt `indicator_snapshot` Eintraege fuer jeden Bar-Zeitpunkt
- [ ] Die `runId` in `indicator_snapshot` stimmt mit der `runId` des TradingRun ueberein
- [ ] `IndicatorQueryService.loadPersistedIndicators()` liefert die persistierten Werte (bestehender Read-Path funktioniert)
- [ ] Chart-UI zeigt KPIs aus der Tabelle an (statt on-the-fly Neuberechnung)
- [ ] KPI-Engine (`KpiEngine`) bleibt unveraendert — keine DB-Abhaengigkeit in der Engine
- [ ] Persister nutzt Batch-/Buffer-Strategie (nicht pro Bar ein DB-Write)
- [ ] Unit-Tests fuer Mapping `IndicatorResult` → `IndicatorSnapshotEntity`
- [ ] Integrationstest: Backtest-Lauf → indicator_snapshot enthaelt korrekte Werte

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `IndicatorPersister` | odin-brain | `de.its.odin.brain.kpi` | Empfaengt `IndicatorResult` + RunContext (runId, instrumentId), mappt auf `IndicatorSnapshotEntity`, buffert und flusht via `IndicatorSnapshotRepository`. Kein Wissen ueber Pipeline-Logik. |

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Aenderung |
|--------|-------|-----------|
| `TradingPipeline` (oder `BarProcessor`) | odin-core | Nach `kpiEngine.onSnapshot()` das Ergebnis an `IndicatorPersister.persist()` weiterreichen. Minimaler Eingriff: eine Zeile. |
| `IndicatorSnapshotEntity` | odin-brain | Ggf. Factory-Methode `fromIndicatorResult(IndicatorResult, UUID runId, String instrumentId)` fuer sauberes Mapping hinzufuegen. |

### Architektur-Skizze

```
TradingPipeline / SimulationRunner
       │
       │ onSnapshot(MarketSnapshot)
       ▼
   KpiEngine ──────► IndicatorResult
       │                    │
       │ (Trading-Logik)    │ persist(result, runId, instrumentId)
       ▼                    ▼
   EntryRules          IndicatorPersister
   ExitRules                │
   RegimeDetector           │ buffer + flush
                            ▼
                   IndicatorSnapshotRepository
                            │
                            ▼
                   indicator_snapshot (DB)
```

### Konfiguration

```properties
# Buffer size for indicator persistence (flush after N results)
odin.brain.kpi.persistence.buffer-size=50
```

## Konzept-Referenzen
- `IndicatorSnapshotEntity` (`odin-brain/.../persistence/IndicatorSnapshotEntity.java`) — Vollstaendiges Entity mit 20 Feldern (EMA9/21/50/100, RSI14, ATR14, Bollinger, ADX14, +DI/-DI, VWAP, AVWAP, VPIN, etc.)
- `IndicatorSnapshotRepository` (`odin-brain/.../persistence/IndicatorSnapshotRepository.java`) — JPA-Repository mit Read-Methoden (Write-Methoden via JPA auto-generated `save()`/`saveAll()`)
- `IndicatorResult` (`odin-api/.../dto/IndicatorResult.java`) — Record mit allen Indicator-Werten als Berechnungsergebnis der KpiEngine
- ODIN-101 (Konzept: Live-Datenstreaming) — Die persistierten Indicators sind die Datenbasis fuer spaeteres UI-Streaming

## Guardrail-Referenzen
- `CLAUDE.md` — Backend Architecture: "Events are immutable", "Program against interfaces in de.its.odin.api.port"
- `CLAUDE.md` — DDD Persistence: "Entities live in their domain modules"
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

## Notizen fuer den Implementierer

1. **Separation of Concerns ist zentral:** Die KPI-Engine berechnet. Der Persister schreibt. Das sind zwei getrennte Bausteine. Die KPI-Engine darf KEINE DB-Abhaengigkeit bekommen.
2. **IndicatorSnapshotEntity hat bereits alle Felder.** Das Mapping von `IndicatorResult` sollte 1:1 sein. Pruefen, ob alle Felder im `IndicatorResult`-Record vorhanden sind und ob es Namensunterschiede gibt.
3. **Buffer-Flush:** Am Ende eines Trading-Tages (oder bei Pipeline-Shutdown) muss der Buffer geflusht werden, damit keine Daten verloren gehen. Flush auch bei Fehler/Abbruch.
4. **RunId-Beschaffung:** Die Pipeline kennt die RunId (ueber TradingRun oder BacktestRunEntity). Der Persister muss sie bei Konstruktion oder pro Aufruf erhalten.
5. **Bestehende leere Tabelle:** Die Tabelle existiert, hat korrekte Indizes. Kein Schema-Aenderung noetig.
6. **On-the-fly Fallback beibehalten:** `IndicatorQueryService` soll weiterhin on-the-fly berechnen koennen als Fallback (z.B. fuer historische Daten vor Einfuehrung des Persisters). Der Fallback-Pfad bleibt, wird aber nach und nach weniger genutzt.

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `IndicatorPersister` (Mapping, Buffering, Flush)
- [ ] Unit-Test fuer `IndicatorSnapshotEntity.fromIndicatorResult()` Mapping
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `IndicatorSnapshotRepository`

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: Pipeline-Durchlauf schreibt indicator_snapshot-Eintraege
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet
- [ ] Edge Cases identifiziert (Buffer-Overflow, Partial Flush, Duplicate Runs)
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
