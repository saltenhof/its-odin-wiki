# ODIN-026 Implementation Protocol — DegradationManager

## Story-Zusammenfassung

**Story:** ODIN-026 — Degradation Mode FSM (DegradationManager)
**Modul:** odin-core
**Typ:** Neue Komponente (Singleton Service)

**Ziel:** Implementierung eines kontrollierten Degradierungs-Mechanismus vor dem Kill-Switch.
Das System soll nicht direkt den Kill-Switch aktivieren, sondern graduell degradieren:
NORMAL → QUANT_ONLY → DATA_HALT → EMERGENCY.

**State Transitions:**
- NORMAL → QUANT_ONLY: 3 aufeinanderfolgende LLM-Fehler
- NORMAL/QUANT_ONLY → DATA_HALT: Datenfeed >30s stale (Decision Loop gestoppt)
- any → EMERGENCY + Kill-Switch: Datenfeed >60s stale ODER Broker-Reject (sofort)
- QUANT_ONLY → NORMAL: erster LLM-Erfolg

---

## Implementierte Dateien

### Neu erstellt

| Datei | Beschreibung |
|-------|-------------|
| `odin-core/src/main/java/de/its/odin/core/service/DegradationManager.java` | Kern-Implementierung: Singleton State Machine |
| `odin-core/src/test/java/de/its/odin/core/service/DegradationManagerTest.java` | Unit Tests (29 Tests) |
| `odin-core/src/test/java/de/its/odin/core/service/DegradationManagerIntegrationTest.java` | Integrations-Tests (6 Tests) |

### Modifiziert

| Datei | Änderung |
|-------|---------|
| `odin-core/src/main/java/de/its/odin/core/config/CoreProperties.java` | `DegradationProperties` Record + `@AssertTrue` Validierung hinzugefügt |
| `odin-core/src/main/resources/odin-core.properties` | Degradation-Defaults: `llm-failure-threshold=3`, `data-stale-halt-seconds=30`, `data-stale-emergency-seconds=60` |
| `odin-core/src/main/java/de/its/odin/core/config/CoreConfiguration.java` | `degradationManager` Bean + `degradationManager` Parameter in `pipelineFactory` + `lifecycleManager` Beans |
| `odin-core/src/main/java/de/its/odin/core/pipeline/TradingPipeline.java` | Integration: Decision-Loop-Block, LLM-Notification, Entry-Block, QUANT_ONLY LLM-Skip |
| `odin-core/src/main/java/de/its/odin/core/pipeline/PipelineFactory.java` | `DegradationManager` Feld + Konstruktor + `createPipeline` Pass-through |
| `odin-core/src/main/java/de/its/odin/core/service/LifecycleManager.java` | `degradationManager.reset()` in `shutdown()` hinzugefügt |
| `odin-core/src/test/java/de/its/odin/core/pipeline/TradingPipelineTest.java` | `DegradationManager` Mock hinzugefügt |
| `odin-core/src/test/java/de/its/odin/core/pipeline/PipelineFactoryTest.java` | `DegradationManager` Mock hinzugefügt |
| `odin-core/src/test/java/de/its/odin/core/service/LifecycleManagerTest.java` | `DegradationManager` Mock + `testShutdownResetsServices` erweitert |
| `odin-backtest/src/main/java/de/its/odin/backtest/BacktestRunner.java` | `DegradationManager` Instanziierung + Fixes für `CoreProperties`/`PipelineFactory` Konstruktoren |
| `odin-backtest/src/test/java/de/its/odin/backtest/BacktestRunnerTest.java` | `buildCoreProperties()` auf vollständige Signatur aktualisiert |
| `odin-backtest/src/test/java/de/its/odin/backtest/BacktestRunnerGovernanceIntegrationTest.java` | `buildCoreProperties()` auf vollständige Signatur aktualisiert |

---

## Test-Ergebnisse

### Nach initialer Implementierung

```
odin-core:    145 Tests, 0 Failures, 0 Errors
odin-backtest: (nicht kompilierbar — pre-existing Bugs in BacktestRunner)
```

### Nach Review-Fixes und vollständiger Korrektur

```
odin-core:    146 Tests, 0 Failures, 0 Errors
odin-backtest: 143 Tests, 0 Failures, 0 Errors
BUILD SUCCESS
```

---

## ChatGPT-Review (Runde 1)

**Datum:** 2026-02-23
**Owner:** `odin-026-impl`

### Findings

**1. State Transitions — GUT**
Kernübergänge korrekt implementiert. Empfehlung: Konfigurations-Invariante erzwingen
(`dataStaleEmergencySeconds > dataStaleHaltSeconds`).

**2. Thread-Safety — OK**
AtomicReference + synchronized konsistent. Empfehlung: `AtomicInteger` ist redundant wenn
alle Mutations ohnehin synchronized sind — `int` reicht. Kill-Switch-Aufrufe außerhalb
synchronized hätten Risiko bei blockierendem Service (akzeptabel in diesem Design).

**3. TradingPipeline-Integration — VERBESSERUNGSBEDARF**
Kritisch: QUANT_ONLY läuft weiterhin LLM-Calls auf dem Hot-Path — unterläuft den Zweck.
In `resolveLatestLlmAnalysis()` früh prüfen ob `QUANT_ONLY` und dann `null` zurückgeben.

**4. Test-Vollständigkeit — OK mit Lücken**
Fehlt: Test für QUANT_ONLY → DATA_HALT Übergang (kritischer Bug). Fehlt: Test dass
QUANT_ONLY keine LLM-Calls mehr macht.

**5. Code-Qualität — OK**
JSON-Escaping unvollständig (Newlines/Control-Chars nicht escaped). Risiko für
Log-Parser-Corruption im Produktionsbetrieb.

---

## ChatGPT-Review (Runde 2 — Klärungen)

**AtomicInteger vs int:** Bei ausschließlich synchronized Mutations reicht `int`. AtomicInteger
nur sinnvoll wenn der Zähler lock-free von außen lesbar sein soll (hier nicht der Fall).
→ **Umgesetzt: auf `int` umgestellt.**

**QUANT_ONLY + LLM:** `null` ist die sauberere Semantik. Einen alten LLM-Erfolg weiterzuverwenden
wäre faktisch weiterhin LLM-getrieben. Recovery-Mechanismus braucht separaten Probe-Call
(Backoff) — für Produktionsausbau vorgemerkt, aktuell: erste erfolgreiche Antwort triggert Recovery.
→ **Umgesetzt: QUANT_ONLY Early-Exit in `resolveLatestLlmAnalysis()`.**

**Broker-Reject Verdrahtung:** Nur sicherheitskritische Fälle (Exit/Liquidate-Rejects)
sollten EMERGENCY auslösen, nicht Entry-Rejects. Aktuell ist `onBrokerReject()` auf
expliziten Aufruf ausgelegt (DQ-Eskalationspfad). Akzeptabler Scope für v1.
→ **Kein sofortiger Handlungsbedarf (v1 scope klar definiert).**

**JSON-Escaping:** Dedizierte `escapeJsonString()` Hilfsmethode ist die richtige Wahl für
diesen Anwendungsfall (nicht Jackson ObjectMapper wegen Latenzkritikalität im synchronized Pfad).
→ **Umgesetzt: vollständige `escapeJsonString()` Methode implementiert.**

---

## Gemini-Review

**Datum:** 2026-02-23
**Owner:** `odin-026-impl-gemini`

### Findings

**Dimension 1 — Code-Qualität: HOCH**
- Exzellente JavaDoc und Namensgebung
- `AtomicInteger` redundant (alle Mutations synchronized) → **umgesetzt: auf `int` umgestellt**
- JSON-Zusammensetzung per String.format fehleranfällig → **umgesetzt: `escapeJsonString()` implementiert**

**Dimension 2 — Konzepttreue: KRITISCHER BUG GEFUNDEN**

> "Kritischer Bug: Fehlender Übergang von QUANT_ONLY zu DATA_HALT. In `onDataFeedStale()` wird
> geprüft `currentMode.get() == DegradationMode.NORMAL`. Wenn das System in QUANT_ONLY ist, blockiert
> diese Bedingung den Übergang zu DATA_HALT. Der Decision-Loop läuft bei veralteten Marktdaten weiter
> bis zur 60-Sekunden-Marke für EMERGENCY."

→ **Umgesetzt: Bedingung geändert auf `!= DATA_HALT && != EMERGENCY`.**
→ **Neuer Test hinzugefügt: `dataFeedStaleFromQuantOnlyShouldTransitionToDataHalt`.**

Konfirmat von Gemini (Runde 2): "Datenverlust übertrumpft den LLM-Ausfall bei weitem.
Die Eskalation QUANT_ONLY → DATA_HALT muss zwingend erlaubt sein."

**Dimension 3 — Produktionstauglichkeit: GUT**
- MarketClock korrekt verwendet (kein `Instant.now()`)
- `@Validated` + Constraints korrekt
- Empfehlung: `@AssertTrue` für `dataStaleEmergencySeconds > dataStaleHaltSeconds` → **umgesetzt**
- Recovery von DATA_HALT ist bewusst manuell (Spec-konform, Limitation dokumentiert)

---

## Implementierte Review-Fixes

| Finding | Quelle | Fix |
|---------|--------|-----|
| QUANT_ONLY → DATA_HALT Transition fehlte | Gemini (kritisch) | Bedingung in `onDataFeedStale()` auf `!= DATA_HALT && != EMERGENCY` geändert |
| Neuer Test für QUANT_ONLY → DATA_HALT | ChatGPT/Gemini | `dataFeedStaleFromQuantOnlyShouldTransitionToDataHalt` hinzugefügt |
| `AtomicInteger` redundant | ChatGPT + Gemini | Auf `int` umgestellt |
| JSON-Escaping unvollständig | ChatGPT + Gemini | `escapeJsonString()` Hilfsmethode mit vollständiger Control-Char-Behandlung |
| Config-Invariante fehlt | ChatGPT + Gemini | `@AssertTrue isEmergencyThresholdAfterHalt()` in `DegradationProperties` |
| QUANT_ONLY läuft LLM auf Hot-Path | ChatGPT | Early-Exit in `resolveLatestLlmAnalysis()` bei QUANT_ONLY |
| LifecycleManager `shutdown()` reset fehlte | Pre-Review-Analyse | `degradationManager.reset()` in `shutdown()` hinzugefügt |
| BacktestRunner: `CoreProperties` Konstruktor veraltet | Compile-Fehler | Alle 4 Konstruktor-Aufrufe auf vollständige Signatur aktualisiert |

---

## Nicht umgesetzte Findings (begründet)

| Finding | Begründung |
|---------|-----------|
| Broker-Reject Verdrahtung in `handleBrokerEvent()` | Scope v1: `onBrokerReject()` ist für expliziten DQ-Eskalationspfad ausgelegt. Entry vs. Exit-Reject-Unterscheidung erfordert BrokerEvent-Schema-Erweiterung — separates Ticket |
| Probe/Backoff-Mechanismus für QUANT_ONLY Recovery | Aktuell: erster erfolgreicher LLM-Aufruf triggert Recovery. Backoff-Mechanismus ist sinnvoll für Produktionsausbau — separates Ticket |
| DATA_HALT automatische Recovery | Bewusst: Spec definiert manuelle Recovery für DATA_HALT. Verhalten dokumentiert in `onDataFeedRecovered()` JavaDoc |

---

## Offene Punkte / Folge-Tickets

1. **ODIN-0XX: Broker-Reject Verdrahtung** — `handleBrokerEvent()` in `TradingPipeline` bei Exit-Rejects `degradationManager.onBrokerReject()` aufrufen. Erfordert Entry/Exit-Unterscheidung im `BrokerEvent`.
2. **ODIN-0XX: QUANT_ONLY Recovery Probe** — Backoff-Mechanismus mit konfigurierbarem Probe-Intervall statt "erster Aufruf triggert Recovery".
3. **ODIN-0XX: DATA_HALT automatische Recovery** — Optionaler Recovery-Pfad wenn Datenfeed wieder gesund ist und System in DATA_HALT.
