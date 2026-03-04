## Kontext

Wenn nach einem Trade-Zyklus die Re-Entry-Guardrails evaluiert werden, führt ein temporärer Soft-Block (REGIME_GATE bei TREND_DOWN, LLM_REENTRY_NOT_APPROVED) zum permanenten State DAY_STOPPED. Die Pipeline ignoriert danach alle weiteren Market-Snapshots für den Rest des Tages. Ein Regime, das um 10:00 ET TREND_DOWN war, kann um 11:00 ET wieder BULLISH sein — das System schaut aber nie wieder hin.

Beobachtet im Backtest QS-IREN-20260303-1m-r1 (ID: a4916649): Entry 09:31, Trailing-Stop 09:52, REENTRY_BLOCKED um 10:10 wegen REGIME_GATE: TREND_DOWN. Danach: 0 Entry-Evaluierungen für 6 Stunden trotz laufender Pipeline (704 BAR_CLOSE Events).

## Modul
odin-core

## Abhaengigkeiten
Keine

## Geschaetzter Umfang
S

---

## Scope

### In Scope
- Unterscheidung zwischen Hard-Blocks (permanent → DAY_STOPPED) und Soft-Blocks (transient → COOLDOWN mit neuem Timer)
- Hard-Blocks: `MAX_CYCLES_PER_INSTRUMENT`, `MAX_CYCLES_GLOBAL`, `BUDGET_GATE` → DAY_STOPPED (wie bisher)
- Soft-Blocks: `REGIME_GATE`, `LLM_REENTRY_NOT_APPROVED` → Cooldown-Timer reset, State bleibt COOLDOWN, erneute Evaluation nach Ablauf
- Konfigurierbares Re-Evaluation-Intervall für Soft-Blocks (Default: 30 Minuten)
- Konfigurierbare Max-Retry-Anzahl für Soft-Blocks (Default: 3), danach DAY_STOPPED
- REENTRY_SOFT_BLOCKED Event-Typ zur Unterscheidung von permanenten Blocks im Audit-Log
- Zähler für Soft-Block-Retries im TradingPipeline State

### Out of Scope
- Änderungen an der Regime-Detection selbst (ADX-Schwellen, Regime-Enum-Werte)
- Änderungen an der LLM-Analyse-Logik
- Neue Guardrails hinzufügen
- Änderungen am FLAT_INTRADAY → COOLDOWN Übergang

## Akzeptanzkriterien
- [ ] Wenn `evaluateReentryGuardrails()` einen Hard-Block zurückgibt (MAX_CYCLES_*, BUDGET_GATE), wird wie bisher nach DAY_STOPPED transitioniert
- [ ] Wenn `evaluateReentryGuardrails()` einen Soft-Block zurückgibt (REGIME_GATE, LLM_REENTRY_NOT_APPROVED), bleibt der State COOLDOWN und der Cooldown-Timer wird auf `softBlockReEvalMinutes` (Default: 30) zurückgesetzt
- [ ] Nach Ablauf des Soft-Block-Timers werden die Guardrails erneut evaluiert — inklusive frischer LLM-Analyse und aktuellem Regime
- [ ] Nach `maxSoftBlockRetries` (Default: 3) aufeinanderfolgenden Soft-Blocks wird nach DAY_STOPPED transitioniert (Endlos-Schleifen-Schutz)
- [ ] Soft-Block-Events werden als `REENTRY_SOFT_BLOCKED` im EventLog geloggt (mit Retry-Counter und Reason)
- [ ] Hard-Block-Events werden weiterhin als `REENTRY_BLOCKED` geloggt (unverändert)
- [ ] Wenn nach einem Soft-Block die Guardrails beim nächsten Versuch alle passieren, wird ein neuer Zyklus gestartet (beginNewCycle) und der Soft-Block-Zähler zurückgesetzt
- [ ] Der Soft-Block-Zähler wird beim Start eines neuen Zyklus zurückgesetzt (nicht kumulativ über Zyklen)
- [ ] `PipelineStateMachine.DECISION_LOOP_ACTIVE_STATES` bleibt unverändert (COOLDOWN ist bereits enthalten)
- [ ] Konfigurationswerte `soft-block-re-eval-minutes` und `max-soft-block-retries` sind in `odin-core.properties` mit Defaults definiert

## Technische Details

### Neue Klassen

Keine neuen Klassen erforderlich.

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `TradingPipeline` | `handleCooldown()`: Unterscheidung Hard-/Soft-Block. Bei Soft-Block: Timer reset statt DAY_STOPPED. Neuer Counter `softBlockRetryCount`. Neues Event `REENTRY_SOFT_BLOCKED`. |
| `TradingPipeline` | `evaluateReentryGuardrails()`: Return-Typ oder Konvention anpassen, damit Hard/Soft unterscheidbar ist (z.B. Prefix-Konvention `HARD:` / `SOFT:` im Reason-String, oder dediziertes Record `GuardrailResult(String reason, boolean hardBlock)`) |
| `TradingPipeline` | `beginNewCycle()`: `softBlockRetryCount` zurücksetzen |
| `CoreProperties.PipelineProperties` | Zwei neue Felder: `softBlockReEvalMinutes` (int, @Min(5)), `maxSoftBlockRetries` (int, @Min(1)) |
| `TradingPipelineTest` | Tests für Soft-Block-Retry-Logik, Hard-Block-Verhalten, Max-Retries → DAY_STOPPED |

### Konfiguration

```properties
# Re-evaluation interval for soft-blocks (REGIME_GATE, LLM_REENTRY_NOT_APPROVED)
odin.core.pipeline.soft-block-re-eval-minutes=30

# Maximum consecutive soft-block retries before permanent DAY_STOPPED
odin.core.pipeline.max-soft-block-retries=3
```

### Implementierungs-Skizze für `handleCooldown()`

```java
private void handleCooldown(MarketSnapshot snapshot) {
    if (coolingOffUntil == null || marketClock.now().isBefore(coolingOffUntil)) {
        return;
    }

    resolveLatestLlmAnalysis(snapshot);
    GuardrailResult result = evaluateReentryGuardrails(snapshot);

    if (result == null) {
        // All guardrails passed — begin new cycle
        softBlockRetryCount = 0;
        beginNewCycle(snapshot);
        return;
    }

    if (result.hardBlock()) {
        logReentryBlocked(result.reason());
        fsm.transition(PipelineState.DAY_STOPPED);
        return;
    }

    // Soft block — check retry limit
    softBlockRetryCount++;
    if (softBlockRetryCount >= pipelineProperties.maxSoftBlockRetries()) {
        logReentryBlocked(result.reason() + " (max retries exhausted: " + softBlockRetryCount + ")");
        fsm.transition(PipelineState.DAY_STOPPED);
        return;
    }

    // Soft block with retries remaining — reset cooldown timer
    coolingOffUntil = marketClock.now().plus(Duration.ofMinutes(
            pipelineProperties.softBlockReEvalMinutes()));
    logReentrySoftBlocked(result.reason(), softBlockRetryCount);
}
```

## Konzept-Referenzen
- `odin-core/src/main/java/de/its/odin/core/pipeline/TradingPipeline.java` — Zeile 836-912: `handleCooldown()` und `evaluateReentryGuardrails()`, aktuelle Einmal-Evaluation mit DAY_STOPPED
- `odin-core/src/main/java/de/its/odin/core/pipeline/PipelineStateMachine.java` — Zeile 57,61: COOLDOWN→{OBSERVING, DAY_STOPPED, FORCED_CLOSE}, DAY_STOPPED→{EOD}
- `odin-core/src/main/java/de/its/odin/core/config/CoreProperties.java` — Zeile 53-60: PipelineProperties Record (dort neue Felder ergänzen)
- `odin-core/src/main/resources/odin-core.properties` — Zeile 21: bestehender `stop-out-cooldown-minutes` als Referenz für Naming-Konvention

## Guardrail-Referenzen
- `CLAUDE.md` — Coding-Regeln: kein `var`, keine Magic Numbers, Records für DTOs, JavaDoc, MarketClock statt Instant.now()
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Review-Pflicht

## Notizen fuer den Implementierer

1. **GuardrailResult statt String**: Die aktuelle Methode `evaluateReentryGuardrails()` gibt einen String zurück. Für die Hard/Soft-Unterscheidung bietet sich ein privates Record `GuardrailResult(String reason, boolean hardBlock)` an. Alternativ: Prefix-Konvention im String (weniger elegant, aber weniger Refactoring). Das Record ist sauberer und empfohlen.

2. **Kein neuer FSM-State nötig**: COOLDOWN bleibt COOLDOWN — der Unterschied ist nur, ob `coolingOffUntil` zurückgesetzt wird oder nach DAY_STOPPED transitioniert wird. Die FSM-Transitions bleiben unverändert.

3. **DECISION_LOOP_ACTIVE_STATES bleibt unverändert**: COOLDOWN ist bereits in den aktiven States enthalten (Zeile 81). Die Snapshots werden also bereits verarbeitet — `handleCooldown()` wird aufgerufen, und dort passiert die Timer-Prüfung.

4. **Soft-Block-Zähler ist Pipeline-lokal**: `softBlockRetryCount` ist ein einfaches int-Feld in TradingPipeline. Wird bei `beginNewCycle()` zurückgesetzt. Nicht persistent.

5. **LLM-Refresh bei jedem Retry**: `resolveLatestLlmAnalysis(snapshot)` wird bereits am Anfang von `handleCooldown()` aufgerufen (Zeile 852). Das ist korrekt — bei jedem Retry-Versuch wird die aktuellste LLM-Analyse geholt.

6. **Bestehende Tests**: `TradingPipelineTest` und `OmsMultiCycleIntegrationTest` testen das aktuelle Verhalten. Diese Tests müssen angepasst werden, da sich das Verhalten bei REGIME_GATE und LLM-Block ändert.

7. **Default 30 Minuten**: Bewusst länger als der Standard-Cooldown (15 min), um keine Hektik zu erzeugen. Bei 3 Retries × 30 min = 90 min maximale Wartezeit, dann DAY_STOPPED. Das deckt eine typische Regime-Wende ab, ohne endlos zu kreisen.

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (GuardrailResult)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.core.pipeline.*` fuer Konfiguration
- [ ] MarketClock statt Instant.now()

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer Soft-Block-Retry-Logik in TradingPipelineTest
- [ ] Test: Soft-Block → Timer reset → erneute Evaluation → Guardrails passieren → neuer Zyklus
- [ ] Test: Soft-Block → Timer reset → max Retries erreicht → DAY_STOPPED
- [ ] Test: Hard-Block → sofort DAY_STOPPED (Regression)
- [ ] Test: softBlockRetryCount wird bei neuem Zyklus zurückgesetzt
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstests mit realen Klassen
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest: Multi-Cycle-Szenario mit Soft-Block → Retry → Success

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: TradingPipeline, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Timer-Edge-Cases, Concurrent-Fills während Cooldown)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Race Conditions, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Soft/Hard-Block-Klassifikation korrekt?)
- [ ] Dimension 3: Praxis-Review (Gibt es Marktszenarien die wir übersehen?)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`
