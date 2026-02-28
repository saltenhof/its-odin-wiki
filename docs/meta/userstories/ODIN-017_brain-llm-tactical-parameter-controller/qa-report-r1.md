# QS-Report R1: ODIN-017 — LLM Tactical Parameter Controller

**Datum:** 2026-02-22
**Ergebnis: FAIL**

---

## Zusammenfassung

Die Implementierung ist qualitativ hochwertig. Code kompiliert fehlerfrei, 600 Unit-Tests sind grün, und alle wesentlichen funktionalen Anforderungen wurden umgesetzt. **Ein DoD-Punkt ist jedoch nicht erfüllt:** Die Integrationstests (DoD 2.3) fehlen vollständig und wurden laut protocol.md bewusst ausgelassen ("ausgelassen per Story-Prioritaet"). Dies ist nach DoD-Spezifikation Abschnitt 7.2 (Fail-Fast) kein akzeptabler Grund.

---

## 1. Kompilierung

**PASS**

```
mvn compile -pl odin-brain --no-transfer-progress -q
```

Keine Fehler, keine Warnungen.

---

## 2. Unit-Tests (DoD 2.2)

**PASS**

```
Tests run: 600, Failures: 0, Errors: 0, Skipped: 0
```

Testklassen mit ODIN-017-relevanten neuen Tests:

| Testklasse | Neue Tests |
|------------|-----------|
| CallCadencePolicyTest | 17 (neue Klasse) |
| LlmResponseParserTest | +10 (parseTactical, markdown fences, arrays) |
| SchemaValidatorTest | +15 (validateTactical, alle Enums, NaN/Infinity) |
| PlausibilityCheckerTest | +9 (checkTactical, hold caps, confidence caps) |
| ConsistencyCheckerTest | +7 (checkTactical, EXIT+HOLD_RUNNERS contradiction) |
| LlmAnalystOrchestratorTest | +17 (analyzeTactical, circuit breaker, freshness, cadence) |
| PromptBuilderTest | 1 Test aktualisiert |

Alle Story-spezifischen Unit-Test-ACs aus DoD 2.2 (12 Tests gefordert) sind implementiert:
- ✅ Prompt-Aufbau: System-Prompt enthält Bounded-Enum-Definitions und Response-Schema
- ✅ Response-Parsing valides JSON → LlmTacticalOutput korrekt geparst
- ✅ Response-Parsing invalides JSON → Fallback, kein Crash
- ✅ Response-Parsing fehlende Pflichtfelder → Validierungsfehler, kein Crash
- ✅ Schema-Validierung ungültige Enum-Werte → Fehler
- ✅ Schema-Validierung Out-of-Range-Werte (confidence > 1.0) → Fehler
- ✅ Plausibility: EXIT + HOLD_RUNNERS gleichzeitig → korrigiert
- ✅ Consistency: Flip-Flop-Detection
- ✅ Freshness: abgelaufen > 180s → veraltet markiert
- ✅ Kadenz: Normal alle 300s, HIGH_VOLATILITY alle 180s
- ✅ CircuitBreaker: 3 aufeinanderfolgende Failures → QUANT_ONLY Fallback
- ✅ Mocks für HTTP-Clients — keine echten API-Calls

---

## 3. Integrationstests (DoD 2.3)

**FAIL — KRITISCH**

Die DoD-Spezifikation (Abschnitt 2.3) fordert explizit Integrationstests. Die Story definiert 4 konkrete Integrationstests:

1. PromptBuilder + LlmResponseParser + SchemaValidator zusammengeschaltet (reale Klassen)
2. LlmAnalystOrchestrator mit MockLlmProvider — Kadenz-Logik End-to-End
3. CircuitBreaker öffnet nach 3 Failures und schließt nach Cooldown
4. Vollständiger Pfad: Tick-Daten → Prompt → Response → LlmTacticalOutput

**Alle 4 Integrationstests fehlen.** Im Verzeichnis `odin-brain/src/test/java/de/its/odin/brain/llm/` gibt es keine `*IntegrationTest.java`-Datei.

Das protocol.md dokumentiert dies als:
> `[ ] Integrationstests (nicht implementiert, ausgelassen per Story-Prioritaet)`

Dies ist nach DoD-Spezifikation Abschnitt 7.2 kein gültiger Grund. "Story-Priorität" überschreibt die DoD nicht.

**Konsequenz:** FAIL. Story kann nicht als abgeschlossen gelten.

---

## 4. Code-Review (DoD 2.1)

**PASS**

Geprüfte Dateien:
- `PromptBuilder.java` — 3-Tier System-Prompt korrekt implementiert mit Bounded-Enums, Safety-Contract, Confidence-Calibration, Prompt-Version
- `LlmResponseParser.java` — Korrektes Record-Pattern, Markdown-Fence-Stripping, ParsedTacticalResponse korrekt
- `SchemaValidator.java` — Alle 8 Pflichtfelder geprüft, `Double.isFinite()` Guard für confidence (ChatGPT-Finding umgesetzt)
- `PlausibilityChecker.java` — Hold-Duration Cap, UNCERTAIN-Confidence-Cap, 1.0-Confidence-Cap
- `ConsistencyChecker.java` — EXIT+HOLD_RUNNERS Widerspruch korrekt behandelt, Severity-Order korrekt (ChatGPT-Finding umgesetzt)
- `LlmAnalystOrchestrator.java` — 5-Layer Pipeline implementiert, MarketClock verwendet (kein Instant.now()), AtomicReference Thread-Safety, List.copyOf() Immutability
- `CallCadencePolicy.java` — Volatility-abhängige Intervalle, Double.isFinite() Guard
- `LlmAnalysisStore.java` — AtomicReference CAS-Loop, monotone Update-Semantik

Keine Verstöße gegen Coding-Regeln:
- ✅ Kein `var` — explizite Typen
- ✅ Keine Magic Numbers — `private static final` Konstanten durchgehend verwendet
- ✅ Records für DTOs (ParsedTacticalResponse, TacticalPlausibilityResult, etc.)
- ✅ ENUMs statt Strings für endliche Mengen (LlmAction, Regime, ExitBias, etc.)
- ✅ JavaDoc auf allen public Klassen, Methoden und Attributen
- ✅ Keine TODO/FIXME-Kommentare
- ✅ Code-Sprache: Englisch
- ✅ Port-Abstraktion: LlmAnalyst-Interface implementiert
- ✅ MarketClock wird verwendet — kein `Instant.now()` im Trading-Codepfad
- ✅ System.nanoTime() nur für Latenz-Messung (akzeptabel, kein Trading-Pfad)
- ✅ LLM liefert KEINE Preise/Stückzahlen/Ordertypen (Safety-Contract im System-Prompt)

---

## 5. DoD-Check vollständig

| DoD-Punkt | Status | Anmerkung |
|-----------|--------|-----------|
| 2.1 Code-Qualität | ✅ PASS | Alle Regeln eingehalten |
| 2.2 Unit-Tests (≥10 neue) | ✅ PASS | 75+ neue Tests, 600 total grün |
| 2.3 Integrationstests | ❌ FAIL | 0 von 4 geforderten Tests implementiert |
| 2.4 Datenbank | N/A | Kein DB-Zugriff in dieser Story |
| 2.5 ChatGPT-Sparring | ✅ PASS | 2 Runden dokumentiert, Findings eingearbeitet |
| 2.6 Gemini-Review (3 Dimensionen) | ✅ PASS | Alle 3 Dimensionen dokumentiert, Findings bewertet |
| 2.7 protocol.md | ✅ PASS | Vollständig ausgefüllt (alle Pflichtabschnitte) |
| 2.8 Commit & Push | — | Noch ausstehend (QS muss erst bestehen) |

---

## 6. Scope-Prüfung

- ✅ In-Scope: 3-Tier Prompt, Response-Parser, 5-Layer Validation, CircuitBreaker, TTL/Freshness, Kadenz, Claude + OpenAI
- ✅ Out-of-Scope respektiert: Kein Pre-Market Profiling, kein Dual-Key Arbiter, kein Broker/OMS
- ✅ Backward-Compat: Deprecated `analyze()`, `getLatestAnalysis()`, `LlmResponseParser.parse()` etc. korrekt annotiert mit `@Deprecated(since = "ODIN-017", forRemoval = true)`

---

## 7. Konzepttreue

Konzept-Referenzen `docs/concept/04-llm-integration.md` wurden durch Gemini-Review Dimension 2 geprüft. Befund: Implementation entspricht dem Konzept. Abweichung beim JSON-Parser (String-basiert statt Library) dokumentiert als akzeptierte technische Schuld.

---

## Nacharbeitsauftrag

**Implementiere die 4 fehlenden Integrationstests** in einer neuen Datei:
`odin-brain/src/test/java/de/its/odin/brain/llm/LlmTacticalPipelineIntegrationTest.java`

Geforderte Tests (aus DoD 2.3 der Story):

1. **PromptBuilder + LlmResponseParser + SchemaValidator** zusammengeschaltet mit realen Klassen:
   - Baue System-Prompt, baue User-Message, parse valide JSON-Response, validiere gegen Schema → VALID

2. **LlmAnalystOrchestrator mit MockLlmProvider — Kadenz-Logik End-to-End:**
   - Erster Call: shouldCallNow=true (null lastCallTime) → Call wird ausgeführt
   - Zweiter Call sofort: shouldCallNow=false → kein zweiter Call
   - Dritter Call nach Intervall: shouldCallNow=true → Call wird ausgeführt

3. **CircuitBreaker öffnet nach 3 Failures und schließt nach Cooldown:**
   - 3x Failure-Responses → circuitBreaker.isOpen() == true
   - Nach Cooldown-Zeit → schließt wieder

4. **Vollständiger Pfad Tick-Daten → Prompt → Response → LlmTacticalOutput:**
   - MockLlmProvider liefert valide JSON-Response
   - orchestrator.analyzeTactical() → LlmTacticalOutput mit korrekten Enum-Werten

Klasse muss `*IntegrationTest` heißen (Failsafe-Konvention). Reale Klassen verwenden, minimal mocken (nur den HTTP-Provider mocken).
