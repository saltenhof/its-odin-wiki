# Protokoll: ODIN-110 — Soft-Block Re-Evaluation in Cooldown

## Working State
- [x] GuardrailResult record implementiert (private record in TradingPipeline)
- [x] softBlockRetryCount Feld hinzugefuegt
- [x] evaluateReentryGuardrails() refactored (String → GuardrailResult)
- [x] handleCooldown() mit Hard/Soft-Block-Logik implementiert
- [x] logReentrySoftBlocked() und EVENT_REENTRY_SOFT_BLOCKED Konstante hinzugefuegt
- [x] Counter-Reset in beginNewCycle()
- [x] CoreProperties um softBlockReEvalMinutes (@Min(5)) und maxSoftBlockRetries (@Min(1)) erweitert
- [x] odin-core.properties Defaults gesetzt (soft-block-re-eval-minutes=30, max-soft-block-retries=3)
- [x] BacktestRunner PipelineProperties-Konstruktor aktualisiert
- [x] Unit-Tests geschrieben (9 neue/angepasste Tests in TradingPipelineTest)
- [x] Integrationstest geschrieben (SoftBlockReEvalIntegrationTest: 1 Test, bestanden)
- [x] 8 weitere Testdateien in odin-backtest und odin-app aktualisiert (PipelineProperties-Konstruktoraufrufe)
- [x] ChatGPT-Sparring fuer Edge Cases (4 zusaetzliche Tests umgesetzt)
- [x] Gemini-Review Dimension 1 (Code-Review)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis-Review)
- [x] Review-Findings bewertet (alle 6 Findings rejected, 0 Code-Aenderungen)
- [x] Commit & Push

## Design-Entscheidungen

### 1. GuardrailResult als private record in TradingPipeline
`GuardrailResult` kapselt ob ein Block vorliegt (boolean blocked), ob er ein Soft-Block ist (boolean soft),
und den Grund (String reason). Als `private record` in TradingPipeline bleibt er ein internes Implementierungsdetail —
kein oeffentlicher API-Typ. Der Record ist immutable per Definition und braucht keinen expliziten equals/hashCode.

### 2. Hard-Block vs. Soft-Block Klassifikation
Hard-Blocks sind strukturelle Limits, die sich innerhalb eines Tages nicht aendern:
- MAX_CYCLES_PER_INSTRUMENT, MAX_CYCLES_GLOBAL: Zaehler koennen nur steigen
- BUDGET_GATE: Per JavaDoc von getRemainingRiskBudget() explizit "Gains do NOT increase the budget" (Konzept 06)

Soft-Blocks sind transiente Marktbedingungen, die sich im Zeitverlauf verbessern koennen:
- LLM_REENTRY_NOT_APPROVED: LLM-Meinung kann sich bei neuem Marktkontext aendern
- REGIME_GATE: Regime-Einschaetzung ist dynamisch

### 3. >= Vergleich bei maxRetries-Pruefung
`softBlockRetryCount >= maxSoftBlockRetries` bedeutet: bei maxSoftBlockRetries=3 werden genau 3
Soft-Block-Evaluierungen erlaubt (Retries 1, 2, 3 — danach wird blockiert). Die Semantik ist
konsistent ueber alle Tests dokumentiert.

### 4. Counter-Reset in beginNewCycle()
softBlockRetryCount wird in beginNewCycle() zurueckgesetzt, damit jeder neue Handelszyklus
mit einem frischen Counter beginnt. Kein Ueberlauf zwischen Zyklen moeglich.

### 5. Konfiguration in odin-core.properties
Defaults: soft-block-re-eval-minutes=30, max-soft-block-retries=3.
Die Parameter sind ueber @ConfigurationProperties in CoreProperties (PipelineProperties Record)
mit @Min-Validierung abgesichert. Ein Spring-Startup-Fehler tritt bei Konfigurations-Verletzungen auf.

## ChatGPT-Sparring

### Session
ChatGPT wurde mit der Implementierung, den bestehenden Tests und den Akzeptanzkriterien konsultiert.
Aufgabe: Edge Cases identifizieren, die in den vorhandenen Tests fehlen.

**Vorgeschlagene Edge Cases (6 Kategorien):**
1. Soft-Block → Hard-Block Wechsel zwischen Retries
2. Timer-Boundary-Conditions (coolingOffUntil == now)
3. Concurrent Fills waehrend Cooldown
4. Double Snapshot am gleichen Timestamp
5. Config Edge Cases (maxRetries=0)
6. softBlockRetryCount Integer Overflow

**Umgesetzte Tests (4):**

| Test | Begruendung |
|---|---|
| `testSoftBlockThenHardBlockOnNextEvaluationStopsImmediately` | Reale Luecke: Bedingungen verschlechtern sich zwischen Retries |
| `testCooldownEvaluatesWhenNowEqualsCoolingOffUntil` | Boundary: isBefore() false bei Gleichheit |
| `testDoubleSnapshotAtSameTimestampDoesNotDoubleIncrementRetryCount` | Schutz vor doppeltem Increment |
| `testMaxSoftBlockRetriesZeroStopsImmediatelyOnFirstSoftBlock` | Config-Edge-Case dokumentiert |

**Abgelehnte Vorschlaege (5):**
- LLM Retry-Loop: Bereits durch SoftBlockReEvalIntegrationTest abgedeckt
- Integer Overflow: Theoretisch, maxRetries=3 vs. int max 2,1 Mrd.
- EventLog Failure: Bereits try/catch mit non-fatal Logging vorhanden
- Late Fills: Out of Scope (Pipeline single-threaded pro Instrument)
- Negative Config: Spring @Validated/@Min behandelt das beim Startup

## Gemini-Review

### Dimension 1: Code-Review (Korrektheit, SOLID, Codequality)

**Findings:**

1. **OFF-BY-ONE im Retry-Counter (HIGH laut Gemini)**
   Gemini sah potenzielle Verwirrung ob maxSoftBlockRetries die maximale Anzahl Retries oder
   Evaluierungen bedeutet.
   **Bewertung REJECTED**: Semantik ist konsistent ueber alle Tests. maxSoftBlockRetries = maximale
   Anzahl Soft-Block-Evaluierungen. JavaDoc und Tests dokumentieren das eindeutig.

2. **Manuelles JSON-Building (LOW)**
   Gemini moniert hardcoded JSON-String-Konstruktion im Event-Logging.
   **Bewertung REJECTED**: Kontrollierte Werte (Ticker, hardcodierte Reasons), konsistent mit
   Codebase-Pattern. Kein ObjectMapper-Overhead fuer simple strukturierte Strings.

3. **Null-Safety / Race Conditions**
   Keine Issues gefunden. Pipeline single-threaded pro Instrument — Race Conditions strukturell
   ausgeschlossen.

### Dimension 2: Konzepttreue

**Findings:**

1. **BUDGET_GATE als Soft-Block? (HIGH laut Gemini)**
   Gemini fragte ob BUDGET_GATE nicht als Soft-Block behandelt werden sollte, da sich das
   Budget durch Gewinne theoretisch erhoehen koennte.
   **Bewertung REJECTED**: JavaDoc von getRemainingRiskBudget() explizit: "Gains do NOT increase
   the budget" (Konzept 06). Konservative Hard-Block-Klassifikation ist by-design.

2. **Restliche Klassifikationen**
   LLM_REENTRY_NOT_APPROVED und REGIME_GATE als Soft-Blocks: MATCH — bestaetigt korrekt.
   MAX_CYCLES-Grenzen als Hard-Blocks: MATCH — bestaetigt korrekt.

### Dimension 3: Praxis-Review

**Findings:**

1. **30min Re-Eval zu lang (MEDIUM)**
   Gemini argumentiert: 30min Re-Evaluierungsintervall ist in volatilen Intraday-Maerkten
   moeglicherweise zu konservativ (verpasste Entries).
   **Bewertung REJECTED (kein Code-Change)**: Parameter ist konfigurierbar. Tuning erfolgt
   nach Real-World-Beobachtung. Als zukuenftige Optimierungsoption notiert.

2. **Whipsawing / Flash-Crash-Verhalten**
   Korrekt behandelt: Retry-Exhaustion stoppt bei anhaltenden Soft-Blocks. Kill-Switch
   greift bei extremen Marktbewegungen. Keine Aenderung noetig.

**Ergebnis: 6 Findings, 0 akzeptiert, 0 Code-Aenderungen. Implementierung ist solide.**

## Geaenderte Dateien

### Production Code (4 Dateien)
1. `TradingPipeline.java` — Hauptimplementierung: GuardrailResult Record, softBlockRetryCount Feld,
   evaluateReentryGuardrails() refactored (String → GuardrailResult), handleCooldown() mit
   Hard/Soft-Block-Logik, logReentrySoftBlocked(), EVENT_REENTRY_SOFT_BLOCKED Konstante,
   Counter-Reset in beginNewCycle()
2. `CoreProperties.java` — softBlockReEvalMinutes (@Min(5)) und maxSoftBlockRetries (@Min(1))
   in PipelineProperties Record
3. `odin-core.properties` — Defaults: soft-block-re-eval-minutes=30, max-soft-block-retries=3
4. `BacktestRunner.java` — PipelineProperties-Konstruktor um neue Parameter erweitert

### Test-Dateien (12 Dateien)
5. `TradingPipelineTest.java` — 9 neue/angepasste Tests fuer Soft-Block-Logik
6. `SoftBlockReEvalIntegrationTest.java` — Neuer Integrationstest: Multi-Cycle Soft-Block → Retry → Success
7–16. 8 weitere Testdateien in odin-backtest und odin-app — PipelineProperties-Konstruktoraufrufe aktualisiert

## Testergebnis Final
- **344 Unit-Tests**: 0 Failures, 0 Errors, 0 Skipped
- **1 Integrationstest (SoftBlockReEvalIntegrationTest)**: 0 Failures, 0 Errors
- **Build**: SUCCESS
