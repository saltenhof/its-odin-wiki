# Protokoll: ODIN-077 -- Graded Fallback Hierarchy

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (FallbackLevelTest: 13, LlmAnalysisStoreFreshnessTest: 28)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (FallbackHierarchyIntegrationTest: 7)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### 1. FallbackLevel als Enum in odin-api
FallbackLevel liegt in `de.its.odin.api.model` (nicht in odin-brain), da FusionResult (odin-brain)
und potenziell odin-core das Level fuer Audit/Logging benoetigen. Orthogonal zu DegradationMode:
DegradationMode = System-Level (odin-core), FallbackLevel = LLM-Verfuegbarkeitsgrad (odin-brain per Pipeline).

### 2. Rueckwaertskompatible Overloads
`DecisionArbiter.decide(7-param)` delegiert an neues `decide(9-param)` mit Defaults (FULL, 1.0).
`FusionResult.noAction(4-param)` delegiert an neues `noAction(7-param)` mit FallbackLevel.FULL.
Dadurch muessen ~80+ existierende Aufrufe in Tests nicht geaendert werden.

### 3. EXIT_ONLY Praezedenz
Timeout-basiertes EXIT_ONLY ueberschreibt freshness-basierte Level (FULL/STALE/QUANT_ELEVATED),
weil konsekutive Timeouts aktive LLM-Instabilitaet signalisieren, auch wenn vorherige Analyse
noch innerhalb der Freshness-Grenzen liegt. HALTED (keine Analyse oder sehr alt) hat aber
Vorrang vor EXIT_ONLY.

### 4. Freshness-Multiplier-Anwendung
Multiplier wird auf LLM-Confidence NACH Kalibrierung aber VOR min-confidence-Threshold angewendet.
Dies bedeutet: bei QUANT_ELEVATED (0.40) wird z.B. 0.75 * 0.40 = 0.30 --> unter min-confidence
--> LLM-Vote rejected --> Entry wird blockiert durch Dual-Key-Protokoll.

### 5. Timeout-Counter in LlmAnalysisStore
AtomicInteger fuer Thread-Safety. Reset erfolgt bei erfolgreichem `tryUpdateTactical()`.
Separate AtomicReference/AtomicInteger statt kombiniertem State-Object: akzeptabler Trade-off
fuer Einfachheit. Race-Window ist minimal (beide Reads in getFallbackLevel erfolgen fast
gleichzeitig im selben Thread-Kontext).

## Geaenderte Dateien

### Neue Dateien
- `odin-api/.../model/FallbackLevel.java` -- 5-Stufen-Enum mit allowsNewEntries/allowsExits
- `odin-brain/.../llm/LlmAnalysisStoreFreshnessTest.java` -- 28 Unit-Tests
- `odin-brain/.../arbiter/FallbackHierarchyIntegrationTest.java` -- 7 Integrationstests
- `odin-api/.../model/FallbackLevelTest.java` -- 13 Unit-Tests

### Modifizierte Dateien
- `odin-brain/.../config/BrainProperties.java` -- FreshnessProperties Record hinzugefuegt
- `odin-brain/src/main/resources/odin-brain.properties` -- 6 neue Properties
- `odin-brain/.../llm/LlmAnalysisStore.java` -- Freshness-Decay + Timeout-Tracking
- `odin-brain/.../arbiter/FusionResult.java` -- activeFallbackLevel-Feld
- `odin-brain/.../arbiter/DecisionArbiter.java` -- 9-param decide() mit EXIT_ONLY-Check
- 10 Testdateien aktualisiert (FreshnessProperties-Parameter hinzugefuegt)

## ChatGPT-Sparring

### Kernpunkte (akzeptiert)
1. **Design ist korrekt**: Praezedenz HALTED > EXIT_ONLY > Freshness bestaetigt
2. **Freshness-Multiplier nur fuer Entry**: Empfehlung, Exits nicht durch Confidence-Scaling
   zu beeinflussen. Aktuell implementiert: Multiplier wirkt auf LLM-Vote, Exits verwenden
   keinen min-confidence-Threshold -- korrekt.
3. **Boundary-Conditions**: age==120s ist STALE (nicht FULL), age==300s ist QUANT_ELEVATED.
   Konvention: `<` fuer untere Grenze (exklusive obere Grenze). Explizit getestet.
4. **EXIT_ONLY Hysterese**: Empfehlung, nach Recovery 1-2 Erfolge zu fordern bevor Entries
   wieder erlaubt werden. Valide Verbesserung, aber ausserhalb ODIN-077 Scope -- kein Overfitting
   fuer v1.

### Kernpunkte (zur Kenntnis genommen, spaetere Story)
5. **Combined AtomicReference State**: Race-Condition bei unabhaengigen Atomics. Anerkennung:
   theoretisch moeglich, praktisch minimal. Fuer v1 akzeptabel, Refactoring-Option fuer ODIN-078.
6. **Timeout-Taxonomie**: Unterscheidung TIMEOUT vs RETRYABLE_ERROR vs FATAL_SCHEMA --
   aktuell zaehlt nur TIMEOUT. Erweiterung in separater Story.
7. **Transition-Logging als Events**: FallbackLevel-Wechsel als Audit-Events. Implementierung
   erfordert EventLog-Erweiterung -- separates Work Package.

## Gemini-Review

### Dimension 1: Code Quality
- **AtomicReference/AtomicInteger**: Wie ChatGPT, Hinweis auf independent-atomics.
  Entscheidung: v1 akzeptabel, Refactoring-Option notiert.
- **Java 21 Idioms**: Record-Nutzung fuer FreshnessProperties korrekt.
  FallbackLevel als enum mit boolean-Feldern passend fuer aktuellen Use Case.
- **Constants**: Keine Magic Numbers. Defaults in .properties, nicht in Java.

### Dimension 2: Konzepttreue
- 5-Stufen-Hierarchie korrekt abgebildet
- Praezedenz-Logik (HALTED > EXIT_ONLY > Freshness) konsistent
- Hinweis: 600s fuer Intraday-Trading "eine Ewigkeit" -- Thresholds koennten
  volatilitaetsabhaengig werden. Fuer v1 sind konfigurierbare feste Werte akzeptabel.

### Dimension 3: Trading-System-Praxis
- **"Stale Directional Bias" Risk**: Bei QUANT_ELEVATED kann eine 8 Minuten alte Long-Empfehlung
  mit 0.40 Multiplier immer noch ueber dem min-threshold liegen. Entschaerfung: Dual-Key erfordert
  auch Quant-Zustimmung (alle Gates muessen passen). Quant-Gates pruefen Live-Marktdaten.
- **Exit-Mechanismus bei HALTED**: Exits muessen rein quantitativ funktionieren (Trailing Stop,
  Stop Loss, EOD-Flat). Aktuell korrekt: Exits haben keinen min-confidence-Threshold.
- **Multiplier-Shape**: Stufenfoermig (1.0 -> 0.70 -> 0.40 -> 0.0) statt exponentiell.
  Einfacher zu debuggen/loggen. Exponentieller Decay als spaetere Optimierung moeglich.

## Testergebnisse

| Testklasse | Tests | Ergebnis |
|---|---|---|
| FallbackLevelTest | 13 | PASS |
| LlmAnalysisStoreFreshnessTest | 28 | PASS |
| FallbackHierarchyIntegrationTest | 7 | PASS |
| Bestehende odin-brain Tests | 1356 | PASS |
| Bestehende odin-core Tests | 321 | PASS |
| Bestehende odin-app Tests | 305 | PASS |
| Gesamter Build | alle Module | BUILD SUCCESS |
