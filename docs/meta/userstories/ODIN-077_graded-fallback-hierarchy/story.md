# ODIN-077: Graded Fallback Hierarchy

**Modul:** odin-brain, odin-api
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** S

---

## Kontext

Das aktuelle LLM-Verfuegbarkeitsmodell ist binaer: entweder ist eine frische LLM-Analyse verfuegbar, oder das System faellt auf QUANT_ONLY zurueck. In der Praxis treten haeufig Zwischenzustaende auf — eine Analyse ist 3 Minuten alt (noch teilweise nutzbar, aber mit reduzierter Confidence), oder der CircuitBreaker hat 2 von 3 Fehlern gesehen (noch nicht offen, aber instabil). Eine gradierte 5-Stufen-Degradation ermoeglicht differenzierteres Verhalten: statt abruptem Wechsel von "voll" auf "aus" gibt es fliessende Uebergaenge mit reduzierten Confidence-Werten.

## Scope

**In Scope:**

- Neues Enum `FallbackLevel` in `de.its.odin.api.model` mit 5 Stufen (FULL, STALE, QUANT_ELEVATED, EXIT_ONLY, HALTED)
- Freshness-Confidence-Decay-Funktion: LLM-Analyse-Alter → Confidence-Multiplikator (1.0 → 0.7 → 0.4 → 0.0)
- Erweiterung von `LlmAnalysisStore` um `getFreshnessMultiplier(Instant now)` Methode
- Erweiterung von `DecisionArbiter`: Freshness-Multiplikator wird auf LLM-Confidence angewendet
- EXIT_ONLY-Modus als explizite Logik: bei 3+ Timeouts keine neuen Entries, nur Exits
- Konfigurationsparameter fuer Freshness-Thresholds in `BrainProperties.LlmProperties`

**Out of Scope:**

- Aenderung des CircuitBreaker-Mechanismus selbst (bleibt 3-State: CLOSED/OPEN/PROLONGED)
- Automatische Recovery von EXIT_ONLY zurueck zu FULL (manuell oder ueber CircuitBreaker-Recovery)
- Frontend-Anzeige des aktuellen FallbackLevels (separate Story)
- Aenderungen am DegradationMode-Enum (FallbackLevel ist orthogonal, nicht ersetzend)

## Akzeptanzkriterien

- [ ] Neues Enum `FallbackLevel` mit 5 Stufen: `FULL`, `STALE`, `QUANT_ELEVATED`, `EXIT_ONLY`, `HALTED`
- [ ] Freshness-Decay-Funktion in `LlmAnalysisStore`:
  - Alter < 120s → Multiplikator 1.0 (FULL)
  - Alter 120-300s → Multiplikator 0.7 (STALE)
  - Alter 300-600s → Multiplikator 0.4 (QUANT_ELEVATED)
  - Alter > 600s → Multiplikator 0.0 (effektiv QUANT_ONLY)
- [ ] Alle Schwellen konfigurierbar via Properties (nicht hardcoded)
- [ ] `DecisionArbiter` multipliziert LLM-Confidence mit dem Freshness-Multiplikator VOR der Fusion
- [ ] EXIT_ONLY-Modus: Nach 3+ konsekutiven LLM-Timeouts werden neue Entries blockiert, bestehende Positionen werden normal verwaltet (Trailing Stop, Targets)
- [ ] `FusionResult` enthaelt neues Feld `FallbackLevel activeFallbackLevel`
- [ ] Unit-Test: LLM-Analyse 90s alt → Multiplikator = 1.0, FallbackLevel = FULL
- [ ] Unit-Test: LLM-Analyse 200s alt → Multiplikator = 0.7, FallbackLevel = STALE
- [ ] Unit-Test: LLM-Analyse 400s alt → Multiplikator = 0.4, FallbackLevel = QUANT_ELEVATED
- [ ] Unit-Test: LLM-Analyse 700s alt → Multiplikator = 0.0, FallbackLevel = effektiv QUANT_ONLY
- [ ] Unit-Test: 3 konsekutive Timeouts → EXIT_ONLY aktiv, Entry-Intent wird rejected
- [ ] Integrationstest: DecisionArbiter mit zunehmend alten LLM-Analysen zeigt korrekt abnehmende fusedConfidence

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `LlmAnalysisStore` | `odin-brain/.../llm/LlmAnalysisStore.java` | Neue Methode `getFreshnessMultiplier(Instant now)`, `getFallbackLevel(Instant now)` |
| `DecisionArbiter` | `odin-brain/.../arbiter/DecisionArbiter.java` | Freshness-Multiplikator in LLM-Confidence-Berechnung, EXIT_ONLY-Logik |
| `FusionResult` | `odin-brain/.../arbiter/FusionResult.java` | Neues Feld `FallbackLevel activeFallbackLevel` |
| `BrainProperties.LlmProperties` | `odin-brain/.../config/BrainProperties.java` | Neue Felder fuer Freshness-Schwellen und Multiplikatoren |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `FallbackLevel` | `de.its.odin.api.model` | Enum mit 5 Stufen + `allowsNewEntries()` und `allowsExits()` Methoden |

### Konfiguration

```properties
# Freshness decay thresholds (seconds)
odin.brain.llm.freshness.full-max-age-s=120
odin.brain.llm.freshness.stale-max-age-s=300
odin.brain.llm.freshness.quant-elevated-max-age-s=600
# Freshness multipliers
odin.brain.llm.freshness.stale-multiplier=0.70
odin.brain.llm.freshness.quant-elevated-multiplier=0.40
# EXIT_ONLY trigger
odin.brain.llm.freshness.exit-only-timeout-count=3
```

### Design-Entscheidung: FallbackLevel vs. DegradationMode

`FallbackLevel` ist **orthogonal** zu `DegradationMode`:

- `DegradationMode` beschreibt System-Level-Zustaende (NORMAL, DATA_HALT, EMERGENCY) und wird in odin-core verwaltet
- `FallbackLevel` beschreibt den LLM-Verfuegbarkeitsgrad und wird in odin-brain pro Pipeline bestimmt
- Beide koennen gleichzeitig aktiv sein: System = NORMAL, FallbackLevel = STALE

## Konzept-Referenzen

- `theme-backlog.md` Thema 15: "Gradierte Fallback-Hierarchie (5-Stufen-Degradation)" — IST-Zustand (3 Stufen), SOLL-Zustand (5 Stufen), Decay-Tabelle
- `docs/backend/architecture/05-llm-integration.md` — CircuitBreaker, Freshness-Check, QUANT_ONLY
- `docs/backend/architecture/00-system-overview.md` — Degradation Modes, Kill-Switch

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, api-Modul fuer Enums
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints
- `CLAUDE.md` — Coding-Regeln (ENUM statt String, Records fuer DTOs)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention eingehalten

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer Freshness-Decay-Funktion (alle 4 Schwellen)
- [ ] Unit-Tests fuer EXIT_ONLY-Logik
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: LlmAnalysisStore + DecisionArbiter mit alternden Analysen
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen, Akzeptanzkriterien, Tests
- [ ] Edge Cases abgefragt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review
- [ ] Dimension 2: Konzepttreue-Review
- [ ] Dimension 3: Praxis-Review (Race Conditions bei Zeitvergleichen, Clock-Skew)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **MarketClock verwenden:** Alle Zeitvergleiche muessen ueber `MarketClock.now()` laufen, nicht `Instant.now()`. Die `LlmAnalysisStore`-Methode bekommt den aktuellen Zeitstempel als Parameter.
- **Bestehende `entryFreshnessMaxS` und `managementFreshnessMaxS`:** Diese Properties existieren bereits in `BrainProperties.LlmProperties`. Die neuen Freshness-Decay-Schwellen sind feiner granuliert und ersetzen die binaere Pruefung. Pruefen ob die bestehenden Properties migriert oder parallel beibehalten werden sollen — Empfehlung: bestehende Properties als "harte" Grenze beibehalten (Alter > managementFreshnessMaxS = QUANT_ONLY), Decay-Multiplier als graduelles Abschwaechen davor.
- **EXIT_ONLY vs. QUANT_ONLY:** EXIT_ONLY ist strenger als QUANT_ONLY (keine neuen Entries auch nicht von Quant). Der Implementierer muss entscheiden, wie EXIT_ONLY mit dem bestehenden CircuitBreaker-State (OPEN → PROLONGED) interagiert. Empfehlung: EXIT_ONLY ersetzt den aktuellen direkten Sprung von QUANT_ONLY zu TRADE_HALT bei prolonged Outage.
