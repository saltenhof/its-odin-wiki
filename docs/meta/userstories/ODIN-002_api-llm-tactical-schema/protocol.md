# ODIN-002: Protocol — LLM Tactical Parameter Schema

Datum: 2026-02-21
Modul: odin-api
Komponente: LlmTacticalOutput + Enums
Spec-Dokument: docs/concept/04-llm-integration.md, docs/backend/architecture/05-llm-integration.md

---

## Implementierungsnotizen

### Dateien erstellt

- `odin-api/src/main/java/de/its/odin/api/event/LlmTacticalOutput.java` — Haupt-Record
- `odin-api/src/main/java/de/its/odin/api/model/LlmAction.java` — Enum (ENTER, HOLD, EXIT, NO_TRADE)
- `odin-api/src/main/java/de/its/odin/api/model/ExitBias.java` — Enum (HOLD_RUNNERS, TRAIL_NORMAL, TRAIL_TIGHT, TRAIL_WIDE)
- `odin-api/src/main/java/de/its/odin/api/model/TrailMode.java` — Enum (EMA_HL, ATR, STRUCTURE)
- `odin-api/src/main/java/de/its/odin/api/model/ProfitProtectionProfile.java` — Enum (AGGRESSIVE, MODERATE, RELAXED)
- `odin-api/src/main/java/de/its/odin/api/model/TargetPolicy.java` — Enum (FIXED_TARGETS, TRAIL_ONLY, HYBRID)
- `odin-api/src/main/java/de/its/odin/api/model/EntryTiming.java` — Enum (IMMEDIATE, WAIT_PULLBACK, WAIT_CONFIRMATION)
- `odin-api/src/main/java/de/its/odin/api/model/SizeModifier.java` — Enum (FULL, REDUCED, MINIMAL)

### Dateien geändert

- `odin-api/src/main/java/de/its/odin/api/event/LlmAnalysis.java` — @Deprecated(since="ODIN-002", forRemoval=true) hinzugefügt

### Tests erstellt

- `LlmTacticalOutputTest.java` — 36 Unit-Tests (Surefire)
- `LlmTacticalOutputIntegrationTest.java` — 13 Integration-Tests (Failsafe)

---

## Designentscheidungen

### 1. Enum-Werte: Story.md vs. Concept 04

Die story.md definiert abweichende Enum-Werte gegenüber dem älteren docs/concept/04-llm-integration.md:

| Enum | Story-Werte | Konzept-Werte |
|------|-------------|---------------|
| ExitBias | HOLD_RUNNERS, TRAIL_NORMAL, TRAIL_TIGHT, TRAIL_WIDE | HOLD, NEUTRAL, EXIT_SOON, EXIT_NOW |
| TrailMode | EMA_HL, ATR, STRUCTURE | WIDE, NORMAL, TIGHT |
| ProfitProtectionProfile | AGGRESSIVE, MODERATE, RELAXED | OFF, STANDARD, AGGRESSIVE |

**Entscheidung:** Story.md ist die maßgebliche Spec für ODIN-002. Die story.md-Werte wurden implementiert. Die abweichenden Konzept-Werte repräsentieren eine spätere Konzept-Evolution (das Konzept wurde nach der Story-Erstellung überarbeitet). Die Diskrepanz wird als Open Point dokumentiert für eine zukünftige Story, die das Konzept gegen die Implementation harmonisiert.

### 2. Was das LLM NICHT liefert (Anti-Halluzination)

Bewusst ausgeschlossen aus LlmTacticalOutput:
- **Preise jeder Art:** Entry-Preise, Limit-Preise, Stop-Level, Target-Preise — deterministische OMS-Verantwortung
- **Stückzahlen/Lot-Größen:** Risk-Modell-Verantwortung
- **Order-Typen:** OMS-Entscheidung
- **Repricing-Grenzen:** OMS-Entscheidung

`keyLevels: List<Double>` bleibt als Context-Feature erhalten (gemäß story.md AC), ist aber explizit als "context-only, never used as order prices" dokumentiert und durch den Klassennamen-Kontext klar abgegrenzt.

### 3. Compact Constructor

Nach ChatGPT-Sparring wurde ein Compact Constructor eingebaut, der folgende Invarianten erzwingt:
- `confidenceScore` muss finite sein (verhindert NaN/Infinity durch LLM-Deserialisierer)
- `confidenceScore` muss in [0.0, 1.0] liegen
- `availableFromMarketTime >= requestMarketTime` (zeitliche Konsistenz)
- `latencyMs >= 0`, `inputTokens >= 0`, `outputTokens >= 0`
- `holdDurationBars == null || holdDurationBars >= 1`
- Defensive Kopien von `keyLevels` und `catalysts` (deep immutability)

### 4. LlmAnalysis Backward-Compatibility

LlmAnalysis wird mit `@Deprecated(since="ODIN-002", forRemoval=true)` markiert. Migration zu LlmTacticalOutput ist für eine separate Story vorgesehen, nachdem CachedAnalyst angepasst wurde.

---

## ChatGPT-Sparring Ergebnis

### KRITISCH (alle behoben)

1. **List mutability** → Compact Constructor mit `List.copyOf()` implementiert
2. **NaN/∞ confidenceScore** → `Double.isFinite()` Check in Compact Constructor implementiert
3. **keyLevels als Preis-Kanal** → Behalten (story.md AC), aber Dokumentation massiv verstärkt: "context-only, never used as order prices"

### WICHTIG (teilweise behoben)

1. **Fehlende Bounds auf Freitext/Listen** → `@Size(max=200)` auf alternativeScenario hinzugefügt; `@Size(max=5)` auf catalysts
2. **Zeit-Konsistenzinvariante** → `availableFromMarketTime >= requestMarketTime` im Compact Constructor erzwungen
3. **holdDurationBars Einheits-Diskrepanz** → In JavaDoc als "3-minute reference bars" explizit dokumentiert (Abgrenzung zu LlmAnalysis.holdDurationBars "5-minute bars")
4. **requestId für Korrelation** → Als Open Point dokumentiert (nicht in ODIN-002 scope)

### HINWEIS (umgesetzt)

1. **@Deprecated mit since/forRemoval** → `@Deprecated(since="ODIN-002", forRemoval=true)` implementiert

---

## Gemini-Review Ergebnis (3-dimensional)

### Dimension 1: Code-Bugs und technische Korrektheit

- **KRITISCH: Null-Safety bei Collections** → Behoben durch Compact Constructor mit `List.copyOf()`
- **WICHTIG: reentryAllowed als Primitiv** → Behalten. Defaultwert `false` ist konservativ (kein Re-Entry ohne explizite Freigabe). Akzeptables Verhalten.
- **WICHTIG: Enum-Werte-Diskrepanz** → Nicht geändert. Story.md ist die maßgebliche Spec für ODIN-002. Die Diskrepanz zum älteren Konzept-Dokument wird als Open Point dokumentiert.

### Dimension 2: Konzepttreue

- **KRITISCH: Missing pattern_candidates, setup_type, tactic, urgency_level** → Diese Felder sind im Konzept definiert, aber explizit Out-of-Scope für ODIN-002 gemäß story.md. Als Open Point dokumentiert für eine Folge-Story.
- **WICHTIG: opportunity_zones fehlen** → Gleiche Begründung. Out-of-Scope für ODIN-002.
- **WICHTIG: EntryTiming fehlt ALLOW_RE_ENTRY** → Das Konzept modelliert Re-Entry als Enum-Wert in entry_timing_bias, die Story modelliert es separat als reentryAllowed boolean. Story ist maßgeblich; das Design ist fachlich vertretbar (klare Trennung).

### Dimension 3: Praxis-Gaps

- **WICHTIG: Cross-Instrument Isolation (Broad Market Regime)** → Valider operativer Punkt. Out-of-Scope für dieses odin-api-Schema (gehört in Pipeline-Orchestrierung). Als Open Point dokumentiert.
- **WICHTIG: Per-Pipeline Token-Quota** → Operational concern, gehört in LLM-Konfiguration (odin-brain). Als Open Point dokumentiert.
- **HINWEIS: currentPriceAtRequest** → Interessant für Arbiter-Logik. Als Verbesserungsvorschlag für spätere Story dokumentiert.

---

## Open Points

1. **Enum-Werte-Harmonisierung:** Story.md-Enum-Werte vs. Konzept 04 divergieren. Eine Folge-Story sollte die Werte harmonisieren und das Konzept aktualisieren, sobald die LLM-Client-Implementierung (odin-brain) startet.

2. **Fehlende Decision-Features:** `pattern_candidates`, `setup_type`, `tactic`, `urgency_level` aus Konzept 04 Section 3.2 sind nicht in LlmTacticalOutput enthalten. Diese gehören in eine Folge-Story, ggf. zusammen mit der LLM-Client-Implementierung.

3. **opportunity_zones fehlen:** Konzept 04 definiert diese als Kontext-Feature. Aus ODIN-002 ausgeklammert; Folge-Story erforderlich.

4. **requestId für Deduplication/Tracing:** Bei Retries innerhalb derselben MarketTime fehlt eine eindeutige Request-Correlation-ID. ChatGPT empfiehlt UUID. Für spätere Story.

5. **Broad Market Regime Signal:** Für Multi-Instrument-Betrieb fehlt ein Cross-Pipeline-Signal (z.B. SPY-Regime). Gehört in Pipeline-Orchestrierung, nicht in dieses Schema.

6. **Per-Pipeline Token-Quota:** Gemini identifiziert das Risiko, dass ein volatiles Instrument das globale Budget aufzehrt. Konfigurationsthema für odin-brain.
