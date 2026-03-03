# Protokoll: ODIN-098 — Adaptive LLM Sampling (Need-Driven + Phase-Aware)

## Working State

- [x] Initiale Implementierung (6 neue Klassen, 4 modifizierte Klassen)
- [x] Unit-Tests geschrieben (81 Tests in 5 Klassen)
- [x] Integrationstests geschrieben (AdaptiveSamplingIntegrationTest)
- [x] ChatGPT-Sparring (Architektur, Code-Qualitaet, Trading-Relevanz)
- [x] Gemini-Review Dimension 1 (Code-Qualitaet)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis / Trading-Relevanz)
- [x] Review-Findings bewertet
- [x] Commit & Push

## Design-Entscheidungen

### Zwei-Saeulen-Modell: Need-Driven (primaer) + Phase-Aware (sekundaer)

**Problem:** Bisheriges CompositeSamplingPolicy (zeitbasiert OR event-basiert) sampelt gleichmaessig ueber den Tag. Das verschwendet LLM-Budget im ruhigen Mittagshandel und reagiert nicht auf Pipeline-Zustandsaenderungen.

**Entscheidung:** Zwei-Saeulen-Modell mit OR-Logik:
1. **Need-Driven (primaer):** Triggert nur wenn die Pipeline frischen LLM-Kontext BRAUCHT (SESSION_START, ENTRY_CANDIDATE, STATE_TRANSITION, REGIME_CHANGE)
2. **Phase-Aware (sekundaer):** U-foermige Hintergrund-Kadenz fuer Situational Awareness (OPENING=1, MORNING=3, MIDDAY=5, POWER_HOUR=3, CLOSE=2 bars)

**Warum OR statt AND:** AND wuerde Need-Driven-Trigger unterdruecken wenn Phase-Aware nicht faellig ist. OR garantiert: jeder Trigger allein reicht.

### Cooldown in Simulationszeit statt Wall-Clock

**Problem:** Backtests laufen in komprimierter Zeit. `Instant.now()` wuerde Cooldowns instant ablaufen lassen.

**Entscheidung:** Cooldown wird ueber `snapshot.marketTime()` gemessen. `AdaptiveSamplingPolicy.recordSampleTimestamp(Instant)` wird nach jedem erfolgreichen LLM-Call aus TradingPipeline aufgerufen.

### Signaling via Mutable Flags statt Event-Queue

**Problem:** NeedDrivenSamplingPolicy muss von TradingPipeline ueber FSM-Transitions und Regime-Changes informiert werden, aber das LlmSamplingPolicy-Interface kennt nur `shouldSample(MarketSnapshot, long, long)`.

**Optionen:**
1. Interface erweitern (bricht Kompatibilitaet)
2. Event-Queue zwischen Pipeline und Policy
3. Mutable Flags mit Signal-Methoden

**Entscheidung:** Option 3 — Signal-Methoden auf NeedDrivenSamplingPolicy (exponiert via AdaptiveSamplingPolicy). TradingPipeline nutzt `instanceof AdaptiveSamplingPolicy` Pattern. Flags werden nach jeder shouldSample-Evaluation consumed.

**Warum kein Interface-Break:** LlmSamplingPolicy wird auch von TimeBasedSamplingPolicy und EventDrivenSamplingPolicy implementiert. Diese brauchen keine Signals. instanceof-Pattern ist hier angemessen (nur 1 Aufrufer).

### SESSION_START Cooldown-Exemption

**Entscheidung:** SESSION_START wird VOR dem Cooldown-Check geprueft. Der allererste RTH-Bar bekommt immer frischen LLM-Kontext, unabhaengig von einem evtl. noch laufenden Cooldown.

**Warum:** Opening-Setups (Gap-Resolution, Opening Range) sind die profitabelsten Intraday-Setups. Ohne sofortigen LLM-Kontext verpasst die Strategie die kritischste Phase.

### ENTRY_CANDIDATE Queueing statt Discard

**Entscheidung:** Wenn ein ENTRY_CANDIDATE waehrend Cooldown auftritt, wird er in ein internes `queuedEntryCandidate`-Flag uebernommen und nach Cooldown-Ablauf automatisch dequeued.

**Warum:** ENTRY_CANDIDATE bedeutet: der Quant-Teil hat einen validen Entry identifiziert, aber das LLM hat keinen frischen Kontext. Verwerfen wuerde reale Trades verpassen. Queuen stellt sicher, dass der naechste LLM-Call kommt, sobald der Cooldown ablaeuft.

## Neue Dateien

| Datei | Typ | Beschreibung |
|-------|-----|--------------|
| `odin-brain/.../sampling/MarketPhase.java` | Enum | 5 Intraday-Phasen mit fromTimestamp() Factory |
| `odin-brain/.../config/NeedDrivenConfig.java` | Record | Config fuer Need-Driven Trigger (5 boolean flags) |
| `odin-brain/.../config/PhaseConfig.java` | Record | Config pro Phase (intervalBars, eventThresholdAtrFactor) |
| `odin-brain/.../sampling/NeedDrivenSamplingPolicy.java` | Policy | Primaere Saule: signal-basierte Trigger |
| `odin-brain/.../sampling/PhaseAwareSamplingPolicy.java` | Policy | Sekundaere Saule: phasen-abhaengige Kadenz |
| `odin-brain/.../sampling/AdaptiveSamplingPolicy.java` | Policy | Orchestrator: OR-Logik + Cooldown |

## Modifizierte Dateien

| Datei | Aenderung |
|-------|-----------|
| `BrainProperties.java` | LlmSamplingProperties erweitert: +minIntervalS, +NeedDrivenProperties, +PhaseProperties |
| `PipelineFactory.java` | CompositeSamplingPolicy ersetzt durch AdaptiveSamplingPolicy |
| `TradingPipeline.java` | Signal-Aufrufe fuer State Transitions, Regime Changes, Sample Timestamps |
| `odin-brain.properties` | 15 neue Properties fuer adaptive Sampling-Konfiguration |
| 13 Test-Dateien | LlmSamplingProperties Konstruktor-Kompatibilitaet (2 → 5 Parameter) |

## Tests

| Testklasse | Tests | Typ |
|------------|-------|-----|
| MarketPhaseTest | 21 | Unit |
| NeedDrivenSamplingPolicyTest | 20 | Unit |
| PhaseAwareSamplingPolicyTest | 15 | Unit |
| AdaptiveSamplingPolicyTest | 22 | Unit |
| AdaptiveSamplingIntegrationTest | 3 | Integration |
| **Gesamt** | **81** | |

Alle 81 Tests bestehen. Keine Regressionen in bestehenden Tests.

## ChatGPT-Sparring

**Datum:** 2026-03-03
**Prompt-Thema:** Architektur, Code-Qualitaet, Trading-Relevanz

### Eingearbeitete Findings

Keine Code-Aenderungen noetig — alle Findings sind entweder bereits adressiert oder werden als bewusste Design-Entscheidungen akzeptiert (siehe unten).

### Verworfene / Deferred Findings

1. **Cooldown-Sicherheit als API erzwingen (recordSampleTimestamp als Footgun)**
   - Vorschlag: onSampleTaken(barIndex, Instant) statt separatem recordSampleTimestamp
   - DEFER: Gueltig, aber Interface-Aenderung. Fuer naechste Iteration.

2. **Queueing aller Need-Driven-Trigger waehrend Cooldown (nicht nur ENTRY_CANDIDATE)**
   - STATE_TRANSITION und REGIME_CHANGE gehen waehrend Cooldown verloren
   - DEFER: Konzept definiert explizit nur ENTRY_CANDIDATE-Queueing. Erweiterung moeglich.

3. **Allokationen pro Bar in PhaseAwareSamplingPolicy**
   - Neue TimeBasedSamplingPolicy/EventDrivenSamplingPolicy pro shouldSample-Call
   - DEFER: Micro-Optimierung. Kein messbarer Impact bei <100 Pipelines.

4. **Out-of-order Timestamp Robustheit**
   - Duration.between kann negativ werden bei Clock-Skew oder Replay-Reset
   - DEFER: Defensive Programmierung, aber kein realer Produktionsfall. Logging genuegt.

5. **Early-Close / Holiday-Kalender**
   - RTH_CLOSE immer 16:00 ET. Passt nicht fuer Early-Close-Tage.
   - DEFER: Erfordert Trading-Calendar-Service. Out of scope fuer ODIN-098.

6. **Thread-Safety fuer Signal-Methoden**
   - Signal-Methoden koennten von anderen Threads aufgerufen werden
   - NOT APPLICABLE: Pipeline ist single-threaded per Design. JavaDoc dokumentiert dies explizit.

## Gemini-Review

**Datum:** 2026-03-03

### Dimension 1: Code-Qualitaet

**Bestaetigt:**
- Java 21 Coding Rules vollstaendig eingehalten
- JavaDoc auf sehr hohem Niveau
- Null-Safety und Edge-Case-Handling robust (Fail-Fast, defensives Map.copyOf)
- Test-Qualitaet herausragend (Boundary-Tests sekundengenau, DST-Tests)

**Findings:**
- Code-Duplikation in Test-Helper-Methoden (createFiveMinBarsWithAtr, snapshot)
  - Akzeptiert: Test-Isolation hat Vorrang vor DRY in Unit-Tests

### Dimension 2: Konzepttreue

**Bestaetigt:**
- Alle 5 Konzeptpunkte exakt implementiert
- SESSION_START Cooldown-Exemption korrekt (Pruefung VOR Cooldown-Check)
- ENTRY_CANDIDATE Queueing vollstaendig
- Flag Consumption in jedem Pfad
- Bar-Intervalle und Cooldown korrekt parametrisiert (nicht hartcodiert)

**Fazit:** Implementierung ist dem Konzept aeusserst treu. Keine Abweichungen.

### Dimension 3: Praxis / Trading-Relevanz

**Bestaetigt:**
- Phasen-Grenzen branchenstandard fuer US-Equities
- U-foermige Kadenz (1,3,5,3,2) ist guter Startpunkt
- DST-Handling sauber

**Deferred Findings:**
1. **Fehlender EXIT_CANDIDATE Trigger**
   - DEFER: Exits werden aktuell ueber STATE_TRANSITION (POSITIONED → FLAT) abgedeckt. Separater EXIT_CANDIDATE als Future-Enhancement.

2. **Phasen-abhaengiger Cooldown**
   - Vorschlag: minIntervalSeconds als Teil der PhaseConfig (OPENING=60s, MIDDAY=300-600s)
   - DEFER: Guter Vorschlag. Erfordert PhaseConfig-Erweiterung und Cooldown-Logik-Umbau.

3. **eventThresholdAtrFactor Tuning pro Phase**
   - Bereits implementiert: PhaseConfig hat individuelle eventThresholdAtrFactor pro Phase.

## Bekannte Einschraenkungen

1. **Globaler Cooldown statt phasen-abhaengig**: 300s Cooldown gilt gleichermassen in OPENING und MIDDAY. Kann in der OPENING-Phase zu 3-5 Minuten Verzoegerung bei Entry-Signalen fuehren.
2. **STATE_TRANSITION/REGIME_CHANGE Drop waehrend Cooldown**: Nur ENTRY_CANDIDATE wird gequeued. Andere Need-Driven-Trigger werden waehrend Cooldown verbraucht ohne Wirkung.
3. **Keine Early-Close Unterstuetzung**: MarketPhase nimmt immer 16:00 ET als RTH-Ende an.

## Offene Punkte

Keine innerhalb des Scope von ODIN-098.
