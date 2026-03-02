# ODIN-098: Adaptive LLM Sampling — Need-Driven und Phase-Aware Policies

> **GitHub Issue:** saltenhof/its-odin-backend#99
> **Story-ID:** ODIN-098
> **Status:** Todo
> **Size:** L
> **Module:** odin-brain, odin-core
> **Epic:** Brain Pipeline
> **Erstellt:** 2026-03-02
> **Abhaengigkeit:** ODIN-097 (ATR-normalisierte Event-Schwellen)

---

## Kontext

Im IREN-Backtest (23.02.2026) wurde der erste LLM-Call erst 20 Minuten nach Handelsbeginn ausgeloest. Ein klares Entry-Signal bei 14:45 UTC wurde durch den Dual-Key-Mechanismus blockiert, weil keine frische LLM-Analyse vorlag — 102 USD entgangener P&L allein durch verspaeteten Einstieg. Die Ursache: eine statische `TimeBasedSamplingPolicy` mit festem 5-Bar-Intervall, die weder bedarfsgesteuert noch phasenabhaengig sampelt. Diese Story implementiert das Zwei-Saeulen-Modell: Need-Driven-Trigger (bedarfsgesteuert bei Entry-Signalen, State-Transitions, Regime-Wechseln) als primaerer Mechanismus, und Phase-Aware Background-Cadence (phasenspezifische Sampling-Frequenz fuer die U-foermige Intraday-Volatilitaetskurve) als sekundaerer Mechanismus. Beide werden durch einen harten Mindestabstand-Cooldown gedeckelt.

## Modul

odin-brain (primaer), odin-core (PipelineFactory)

## Abhaengigkeiten

saltenhof/its-odin-backend#98 (ODIN-097: ATR-normalisierte Event-Schwellen — PhaseAwareSamplingPolicy delegiert an ATR-basierte EventDrivenSamplingPolicy)

## Geschaetzter Umfang

L

---

## Scope

### In Scope

- `MarketPhase` Enum (OPENING, MORNING, MIDDAY, POWER_HOUR, CLOSE) mit Zeitgrenzen in EST und Factory `fromTimestamp(Instant, ZoneId)`
- `NeedDrivenSamplingPolicy`: Trigger fuer SESSION_START, ENTRY_CANDIDATE, STATE_TRANSITION, REGIME_CHANGE, SIGNIFICANT_MOVE
- `PhaseAwareSamplingPolicy`: Phasenspezifische Hintergrund-Cadence (delegiert an bestehende TimeBasedSamplingPolicy + EventDrivenSamplingPolicy mit dynamischen, phasenspezifischen Parametern)
- `AdaptiveSamplingPolicy`: Orchestrator mit OR-Verknuepfung (Need-Driven ODER Phase-Aware), inkl. hartem Mindestabstand-Cooldown (`min-interval-s=300`)
- Config-Records (`PhaseConfig`, `NeedDrivenConfig`) und Erweiterung `BrainProperties`
- `PipelineFactory` Anpassung: neue Policy-Hierarchie statt bisheriger `CompositeSamplingPolicy`
- Cooldown-Logik: Mindestabstand in SimClock-Zeit, SESSION_START exempt, ENTRY_CANDIDATE queued bei aktivem Cooldown

### Out of Scope

- ATR-Normalisierung der Event-Schwellen (bereits in ODIN-097)
- LLM-Flipping-Hysterese fuer entry_timing_bias und tactic (separat in ODIN-099)
- 3-Minuten-Bars im Backtest (separat in ODIN-096)
- Volatilitaetsadaptive Uebersteuerung (Phase 3 im Konzept — spaetere Erweiterung)
- Zustandsbasierte Phase-Grenzen (Phase 4 im Konzept — spaetere Erweiterung)
- LLM-Prompt-Aenderungen
- Exchange-Kalender fuer Early Close / Trading Halts

## Akzeptanzkriterien

- [ ] Erster LLM-Call erfolgt beim ersten RTH-Bar (SESSION_START), nicht nach 5-Bar-Warmup
- [ ] ENTRY_CANDIDATE-Trigger: Wenn Quant ALLOW_ENTRY liefert und `lastLlmAnalysis == null` oder TTL abgelaufen, wird sofort ein LLM-Call ausgeloest
- [ ] STATE_TRANSITION-Trigger: Bei FSM-State-Wechsel (z.B. OBSERVING → POSITIONED) wird LLM-Call ausgeloest
- [ ] REGIME_CHANGE-Trigger: Bei Regime-Hysterese (2x konsekutive Decision-Bars neues Regime) wird LLM-Call ausgeloest
- [ ] SIGNIFICANT_MOVE-Trigger: Intra-Bar-Bewegung > k × ATR(14) loest LLM-Call aus (phasenspezifischer ATR-Faktor)
- [ ] Phase-Aware Cadence: OPENING=jeder Bar, MORNING=alle 3 Bars, MIDDAY=alle 5 Bars, POWER_HOUR=alle 3 Bars, CLOSE=alle 2 Bars
- [ ] Harter Mindestabstand (`min-interval-s=300`) wird fuer ALLE Trigger eingehalten (Ausnahme: SESSION_START)
- [ ] Bei aktivem Cooldown wird ENTRY_CANDIDATE gequeued (nicht verworfen) und nach Cooldown-Ablauf sofort ausgefuehrt
- [ ] Single-Flight-Regel bleibt erhalten: maximal ein laufender LLM-Call, weitere Trigger werden koalesziert
- [ ] `MarketPhase.fromTimestamp()` bestimmt Phase korrekt fuer alle EST-Zeitgrenzen inkl. DST
- [ ] Alle bestehenden Sampling-Tests bestehen weiterhin (Rueckwaertskompatibilitaet der bestehenden Policies)
- [ ] Konfiguration ueber Properties steuerbar (alle Trigger und Phasen-Intervalle ein-/ausschaltbar)

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `MarketPhase` | odin-brain | `de.its.odin.brain.llm.sampling` | Enum: OPENING (09:30-10:00 EST), MORNING (10:00-11:30), MIDDAY (11:30-14:00), POWER_HOUR (14:00-15:30), CLOSE (15:30-16:00). Factory `fromTimestamp(Instant, ZoneId)` |
| `NeedDrivenSamplingPolicy` | odin-brain | `de.its.odin.brain.llm.sampling` | Implementiert `LlmSamplingPolicy`. Trigger-Erkennung aus MarketSnapshot: ENTRY_CANDIDATE (Quant ALLOW_ENTRY + kein frisches LLM), STATE_TRANSITION (FSM-Wechsel), REGIME_CHANGE (Regime-Hysterese), SIGNIFICANT_MOVE (ATR-basiert) |
| `PhaseAwareSamplingPolicy` | odin-brain | `de.its.odin.brain.llm.sampling` | Implementiert `LlmSamplingPolicy`. Bestimmt Phase via `MarketPhase.fromTimestamp()`. Delegiert an TimeBasedSamplingPolicy (phasenspezifisches intervalBars) und EventDrivenSamplingPolicy (phasenspezifischer ATR-Faktor) |
| `AdaptiveSamplingPolicy` | odin-brain | `de.its.odin.brain.llm.sampling` | Implementiert `LlmSamplingPolicy`. Orchestrator: (1) SESSION_START-Check, (2) Cooldown-Check, (3) Need-Driven, (4) Phase-Aware, (5) Queued-ENTRY_CANDIDATE. ENTRY_CANDIDATE-Queueing als boolean-Flag |
| `PhaseConfig` | odin-brain | `de.its.odin.brain.config` | Record: `intervalBars` (int), `eventThresholdAtrFactor` (double) — pro Phase |
| `NeedDrivenConfig` | odin-brain | `de.its.odin.brain.config` | Record: `enabled` (boolean), `sessionStart` (boolean), `entryCandidate` (boolean), `stateTransition` (boolean), `regimeChange` (boolean) |

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Package | Aenderung |
|--------|-------|---------|-----------|
| `PipelineFactory` | odin-core | `de.its.odin.core.pipeline` | Zeilen 351-355: Statt `CompositeSamplingPolicy(Time, Event)` neue `AdaptiveSamplingPolicy(NeedDriven, PhaseAware)` instanziieren. Config aus BrainProperties uebergeben |
| `BrainProperties` | odin-brain | `de.its.odin.brain.config` | `LlmSamplingProperties` erweitern: `minIntervalS` (int, default 300), `NeedDrivenProperties` (nested record mit boolean-Flags), `PhaseProperties` (Map oder 5 nested records mit je intervalBars + eventThresholdAtrFactor) |

### Konfiguration

```properties
# === Adaptive LLM Sampling ===

# Hard minimum cooldown between any two LLM calls (seconds)
odin.brain.llm.sampling.min-interval-s=300

# Need-Driven Trigger
odin.brain.llm.sampling.need-driven.enabled=true
odin.brain.llm.sampling.need-driven.session-start=true
odin.brain.llm.sampling.need-driven.entry-candidate=true
odin.brain.llm.sampling.need-driven.state-transition=true
odin.brain.llm.sampling.need-driven.regime-change=true

# Phase-Aware Background-Cadence (interval in decision-bars per phase)
odin.brain.llm.sampling.phase.opening.interval-bars=1
odin.brain.llm.sampling.phase.morning.interval-bars=3
odin.brain.llm.sampling.phase.midday.interval-bars=5
odin.brain.llm.sampling.phase.power-hour.interval-bars=3
odin.brain.llm.sampling.phase.close.interval-bars=2

# Event-driven thresholds per phase (ATR-relative, used by PhaseAwareSamplingPolicy)
odin.brain.llm.sampling.phase.opening.event-threshold-atr-factor=0.4
odin.brain.llm.sampling.phase.morning.event-threshold-atr-factor=0.6
odin.brain.llm.sampling.phase.midday.event-threshold-atr-factor=1.0
odin.brain.llm.sampling.phase.power-hour.event-threshold-atr-factor=0.6
odin.brain.llm.sampling.phase.close.event-threshold-atr-factor=0.4
```

### Algorithmus (Pseudocode)

```
AdaptiveSamplingPolicy.shouldSample(snapshot):
  1. IF isFirstRthBar(snapshot) AND config.sessionStart → return true  (SESSION_START, cooldown-exempt)
  2. IF timeSinceLastCall(snapshot) < minIntervalS → queue ENTRY_CANDIDATE if present, return false
  3. IF needDrivenPolicy.shouldSample(snapshot) → return true
  4. IF phaseAwarePolicy.shouldSample(snapshot) → return true
  5. IF queuedEntryCandidate exists → return true  (dequeued after cooldown)
  6. return false
```

## Konzept-Referenzen

- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 1: Problemstellung (1.1-1.4 IREN-Backtest-Analyse)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 2.1: Need-Driven Trigger (5 Trigger-Typen, Tabelle, ENTRY_CANDIDATE als Kern-Innovation)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 2.1a: Mindestabstand-Constraint (Cooldown-Logik, SESSION_START-Exemption, ENTRY_CANDIDATE-Queueing)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 2.2: Phase-Aware Background-Cadence (5 Phasen, Cadence-Tabelle)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 2.4: Kostenbetrachtung (Backtest und Live)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 3.1: Klassenmodell (Architektur-Hierarchie)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 3.2: Phase-Bestimmung (MarketPhase Enum, fromTimestamp)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 3.3: Konfiguration (vollstaendige Properties-Liste)
- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 3.6: Integration mit bestehendem Code (PipelineFactory, TradingPipeline)
- `docs/concept/04-llm-integration.md` — §6 LLM Call-Cadence, §6.4 Single-Flight-Regel

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln R1-R13, CSpec (Namespace `odin.brain.llm.sampling.*`)
- `CLAUDE.md` — "Use MarketClock — no Instant.now() in trading code paths"
- `CLAUDE.md` — "Program against interfaces in de.its.odin.api.port"

## Notizen fuer den Implementierer

1. **Sampling-Policies sind POJOs, keine Spring Beans.** Sie werden in `PipelineFactory` (Zeilen 351-355) instanziiert und per Constructor-Injection konfiguriert. Config kommt aus `BrainProperties` (Spring-managed).

2. **MarketSnapshot ist die einzige Datenquelle.** `LlmSamplingPolicy.shouldSample(MarketSnapshot)` Signatur bleibt erhalten. Alle benoetigten Informationen (Bar-Timestamp, ATR, Pipeline-State, Quant-Gate-Ergebnis, Regime) muessen aus dem Snapshot lesbar sein. Prüfe ob alle Felder vorhanden sind — ggf. Snapshot erweitern.

3. **Cooldown in SimClock-Zeit.** Mindestabstand ueber `MarketClock` messen, nicht `Instant.now()`. Snapshot-Timestamp ist die Zeitquelle.

4. **ENTRY_CANDIDATE-Queueing:** Implementiere als einfaches `boolean pendingEntryCandidate`-Flag, nicht als komplexe Queue. Wird bei Cooldown-Ablauf geprueft und zurueckgesetzt.

5. **PhaseAwareSamplingPolicy Delegation:** Diese Policy erzeugt intern KEINE neuen Policy-Instanzen pro Phase. Stattdessen: bestehende Time- und Event-Policies mit dynamischen Parametern aufrufen (z.B. durch Setter oder Factory-Methode mit phasenspezifischen Werten).

6. **CompositeSamplingPolicy bleibt erhalten** als bestehender Code, wird aber nicht mehr in PipelineFactory verwendet. Nicht loeschen — koennte fuer andere Zwecke nuetzlich sein.

7. **Boundary-Cases testen:** Exakt 10:00:00 EST (OPENING → MORNING Grenze), DST-Wechsel (US Eastern: zweiter Sonntag im Maerz / erster Sonntag im November), erster vs. zweiter RTH-Bar, Cooldown-Ablauf genau auf 300s.

8. **Single-Flight beachten:** Die bestehende Single-Flight-Regel (nur ein LLM-Call gleichzeitig) lebt in der `TradingPipeline`, nicht in der SamplingPolicy. Die Policy sagt nur "jetzt samplen", die Pipeline entscheidet ob ein Call laeuft.

## Definition of Done

### Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer Config-Objekte (PhaseConfig, NeedDrivenConfig)
- [ ] ENUM statt String (MarketPhase)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.brain.llm.sampling.*`

### Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer MarketPhase, NeedDrivenSamplingPolicy, PhaseAwareSamplingPolicy, AdaptiveSamplingPolicy
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer MarketSnapshot
- [ ] Test: MarketPhase.fromTimestamp() alle Phasen-Grenzen inkl. Boundary
- [ ] Test: SESSION_START (erster RTH-Bar → Trigger, Cooldown-exempt)
- [ ] Test: ENTRY_CANDIDATE mit und ohne Cooldown (queued vs. sofort)
- [ ] Test: STATE_TRANSITION und REGIME_CHANGE Trigger
- [ ] Test: Phase-Cadence-Verhalten (verschiedene Phasen, verschiedene Bar-Counts)
- [ ] Test: Cooldown-Logik (timeSinceLastCall < minIntervalS → false)

### Tests — Komponentenebene (Integrationstests)
- [ ] Mindestens 1 Integrationstest: vollstaendiger Sampling-Zyklus ueber alle 5 Phasen
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit neuen Klassen, Akzeptanzkriterien, Tests
- [ ] Edge Cases und Grenzfaelle abgefragt
- [ ] Ergebnis im `protocol.md` dokumentiert

### Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Null-Safety, Race Conditions bei Single-Flight)
- [ ] Dimension 2: Konzepttreue-Review (vs. concept.md Abschnitte 2+3)
- [ ] Dimension 3: Praxis-Review (Early Close, Trading Halts, unerwartete Szenarien)
- [ ] Findings bewertet und berechtigte Findings behoben

### Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

