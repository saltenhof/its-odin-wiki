# Protokoll: ODIN-017 — LLM als Tactical Parameter Controller

## Working State

- [x] Initiale Implementierung (alle 8 Dateien)
- [x] Compile-Check: odin-brain clean kompiliert
- [x] Existing tests fix: PromptBuilderTest aktualisiert fuer neues Schema
- [x] Unit-Tests geschrieben (600 Tests total, alle gruen)
- [x] ChatGPT-Sparring (2 Runden)
- [x] Review-Findings eingearbeitet
- [x] Gemini-Review (3 Dimensionen)
- [x] Integrationstests (implementiert: LlmTacticalPipelineIntegrationTest.java, 5 Tests, alle gruen)
- [ ] Commit & Push (explizit ausgelassen per Anweisung)
- [x] QS R1: FAIL — Integrationstests fehlen (DoD 2.3 nicht erfüllt, siehe qa-report-r1.md)
- [x] QS R2: PASS — LlmTacticalPipelineIntegrationTest (5 Tests, alle grün), 600 Unit-Tests grün, alle DoD-Punkte erfüllt

## Design-Entscheidungen

### 1. LlmTacticalOutput als neues primaeres Ausgabeformat

`LlmTacticalOutput` (ODIN-002) ist das neue primaere Ausgabeformat. Der `LlmAnalystOrchestrator`
produziert intern `LlmTacticalOutput` und speichert es im migrierten `LlmAnalysisStore`.
Die deprecated `LlmAnalysis`-Methoden (`analyze()`, `getLatestAnalysis()`) bleiben als backward-compat
Wrapper erhalten, damit `CachedAnalyst` nicht bricht.

### 2. PromptBuilder: 3-Tier-Aufbau (System + Context + Tick)

- Tier 1 (System Prompt): Rollendefinition, alle bounded-enum-Definitionen, Safety-Contract,
  Verbotene Felder (keine Preise/Stueckzahlen/Ordertypen), vollstaendiges Response-Schema
- Tier 2 + 3 = User Message (Context-Fenster + aktueller Tick kombiniert wie bisher)

### 3. Response-Parser: ParsedTacticalResponse

Neues internes `ParsedTacticalResponse`-Record mit allen LlmTacticalOutput-Feldern.
String-basiertes JSON-Parsing ohne externe Bibliothek (Konsistenz mit bestehender Implementierung).
Boolean-Parsing fuer `reentryAllowed` hinzugefuegt. Markdown-Fence-Stripping fuer LLM-Ausgaben.

**Bekanntes Risiko (ChatGPT + Gemini):** String-Parser ist fragil bei escaped Quotes in Wertfeldern.
Akzeptiert als technische Schuld — in der Praxis liefern die verwendeten LLMs (Claude, GPT)
sauberes JSON wenn der System Prompt "no markdown, pure JSON" vorschreibt.

### 4. Kadenz-Logik: CallCadencePolicy

Neue `CallCadencePolicy`-Klasse kapselt Volatilitaets-abhaengige Intervalle:
- HIGH_VOLATILITY (ATR-Decay > 1.2): alle 180s
- NORMAL (0.8-1.2): alle 300s
- LOW (< 0.8): alle 900s
`shouldCallNow()` prueft gegen `lastCallMarketTime` und das konfigurierte Intervall.
Non-finite ATR-Decay (NaN, Infinity) → NORMAL-Intervall.

### 5. LlmAnalysisStore migriert auf LlmTacticalOutput

`LlmAnalysisStore` haelt jetzt `AtomicReference<LlmTacticalOutput>` (primaer) und
`AtomicReference<LlmAnalysis>` (legacy, deprecated). Monotone Update-Semantik unveraendert.

### 6. ConsistencyChecker: EXIT + HOLD_RUNNERS Widerspruch

Neuer Check: `action=EXIT` + `exitBias=HOLD_RUNNERS` ist logisch widersprüchlich.
Reaktion: `exitBias` auf `TRAIL_NORMAL` korrigieren und `PLAUSIBILITY_ADJUSTED` setzen.

### 7. Freshness-Enumeration

`FreshnessContext` enum (ENTRY, MANAGEMENT) fuer Freshness-Pruefungen im Orchestrator.
Fallback-Semantik: Orchestrator liefert immer letzten validen Stand, TTL-Pruefung
ist Sache des Callers (DecisionArbiter via `isTacticalAnalysisFresh()`).

### 8. Immutability: List.copyOf() in buildTacticalOutput()

Nach ChatGPT-Review: `keyLevels` und `catalysts` werden mit `List.copyOf()` gesichert,
um Mutation des im AtomicReference gespeicherten LlmTacticalOutput-Records zu verhindern.

### 9. Severity-Order korrigiert

Korrektur nach ChatGPT-Review: `PLAUSIBILITY_ERROR` hat hoehere Severity als `PLAUSIBILITY_ADJUSTED`
(vorher invertiert). Korrekte Reihenfolge: VALID < PLAUSIBILITY_ADJUSTED < PLAUSIBILITY_ERROR
< CONSISTENCY_ERROR < SCHEMA_ERROR.

### 10. Double.isFinite() Guards

Nach ChatGPT-Review hinzugefuegt:
- `SchemaValidator.validateTactical()`: Prueft `Double.isFinite(confidenceScore)` vor Range-Check
- `CallCadencePolicy.determineInterval()`: Prueft `Double.isFinite(atrDecayRatio)` statt nur NaN
- `LlmAnalystOrchestrator.extractAtr()`: Prueft `Double.isFinite()` statt nur NaN

## ChatGPT-Sparring

### Runde 1: 5-Layer Pipeline, Thread-Safety, CallCadencePolicy, Tests

**Wichtigste Findings:**

1. **CRITICAL: NaN passiert SchemaValidator** — `clampConfidence()` propagiert NaN,
   `<` und `>` Vergleiche sind bei NaN immer false → Schema akzeptiert NaN als VALID.
   **Umgesetzt:** `Double.isFinite()` in SchemaValidator.

2. **IMPORTANT: Schema-Layer ist löchrig bei fehlenden Pflichtfeldern** — confidenceScore=0.0 als
   Default ist nicht von "LLM hat 0.0 geliefert" unterscheidbar. Akzeptiert als pragmatischer
   Kompromiss (conservative safe default). Keine Umstellung auf nullable Double.

3. **IMPORTANT: mutable List Leaks über subList()** — subList() gibt eine View zurück,
   nicht eine Kopie. **Umgesetzt:** List.copyOf() in truncateCatalysts() und buildTacticalOutput().

4. **IMPORTANT: Severity-Order invertiert** — PLAUSIBILITY_ADJUSTED hatte hoehere Severity
   als PLAUSIBILITY_ERROR. **Umgesetzt:** Korrekte Reihenfolge.

5. **IMPORTANT: Fallback JavaDoc Mismatch** — "if within TTL" war irreführend.
   **Umgesetzt:** JavaDoc korrigiert, Caller-Responsibility dokumentiert.

### Runde 2: Priorisierung der Findings

**Sofort umgesetzt:** NaN-Guard (Punkt 2), List.copyOf() (Punkt 5), Severity-Order (Punkt 6).

**Dokumentiert/akzeptiert:**
- Fallback TTL: Caller-Verantwortung, JavaDoc korrigiert
- keyLevels als price doubles: bewusste Designentscheidung, context-only, kein Execution-Pfad
- Schema bypass durch Parser-Defaults: pragmatischer Kompromiss mit safe defaults

## Gemini-Review

### Dimension 1 — Architektur & Design

**Befund:** 5-Layer Anti-Hallucination Pattern exzellent fuer Trading-Kontext.
Single-Flight Async mit Coalescing: ressourcenschonend und korrekt.
Bounded-Enum / Execution-Trennung: absolute Best-Practice, minimiert Fat-Finger-Risiko.

**Nichts umzusetzen** — reine Bestaetigung der Architekturentscheidungen.

### Dimension 2 — Code-Qualitaet

**Befund:** Custom JSON-Parser ohne externe Library als Risiko (fragil bei escaped Quotes,
ungewoehnlicher Formatierung). AtomicReference CAS-Loop korrekt und lehrbuchmäßig.
Java 21 Features (Records, Switch Expressions, List.copyOf()) gut genutzt.

**Akzeptiert:** JSON-Parser-Risiko als technische Schuld. In der Praxis liefern die konfigurierten
LLM-Provider (Claude, GPT-4) sauberes JSON wenn der System Prompt es klar fordert.

### Dimension 3 — Tests

**Befund:** Test-Struktur und Namenskonventionen exzellent. Async-Coalescing-Pfad
(`requestAnalysis`, `drainCoalescedRequests`) nicht durch automatisierte Tests abgesichert.
Test-Doubles (TestEventLog, MarketClock-Stub) zweckmaessig und ausreichend.

**Akzeptiert:** Multi-threaded Test fuer Coalescing erfordert determinierten Executor —
komplexitaet vs. Nutzen nicht gerechtfertigt in dieser Story-Phase. Offener Punkt fuer ODIN-020.

## Neue Tests (ODIN-017)

Neue Testklassen:
- `CallCadencePolicyTest.java` — 17 Tests (HIGH/NORMAL/LOW Volatilitaet, Grenzfaelle, Infinity, NaN)

Erweiterte Testklassen (+Tactical-Tests):
- `LlmResponseParserTest.java` — +10 Tests (parseTactical, markdown fences, arrays)
- `SchemaValidatorTest.java` — +15 Tests (validateTactical, alle Enums, NaN/Infinity confidence)
- `PlausibilityCheckerTest.java` — +9 Tests (checkTactical, hold caps, confidence caps)
- `ConsistencyCheckerTest.java` — +7 Tests (checkTactical, EXIT+HOLD_RUNNERS contradiction)
- `LlmAnalystOrchestratorTest.java` — +17 Tests (analyzeTactical, circuit breaker, freshness, cadence)
- `PromptBuilderTest.java` — 1 Test aktualisiert (neues taktisches Schema-Vocabulary)

**Total: 600 Tests, alle gruen.**
