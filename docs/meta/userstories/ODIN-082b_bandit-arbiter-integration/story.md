# ODIN-082b: Bandit-Arbiter-Integration

**Modul:** odin-brain, odin-api, odin-core
**Phase:** 1
**Abhaengigkeiten:** ODIN-082a (Thompson Sampling Kern — liefert `ThompsonSamplingBandit`, `BanditWeights`, `BanditArm`)
**Geschaetzter Umfang:** M

---

## Kontext

ODIN-082 (Adaptive Weighting via Bandit) wird in 3 Sub-Stories aufgeteilt. Diese zweite Sub-Story integriert den in ODIN-082a implementierten `ThompsonSamplingBandit` in den bestehenden `DecisionArbiter`. Der Arbiter bekommt einen Konfigurationsschalter: bei `adaptive-weights.enabled=true` werden Gewichte per Thompson Sampling gezogen statt statisch aus dem `RegimeWeightProfile` (ODIN-074). Zusaetzlich wird `FusionResult` um ein Traceability-Feld erweitert und die `PipelineFactory` wird angepasst, um den Bandit zu instanziieren und zu injizieren.

## Scope

**In Scope:**

- Konfigurationsschalter `odin.brain.arbiter.adaptive-weights.enabled` (Default: false)
- Neue `AdaptiveWeightProperties`-Records in `BrainProperties`
- `DecisionArbiter`-Erweiterung: akzeptiert optional `ThompsonSamplingBandit`; nutzt `sampleWeights(fusedRegime)` pro Decision-Zyklus wenn adaptiv aktiv
- Fallback-Logik: Wenn Bandit weniger als `min-trades-for-activation` Trades hat (Default: 20), statische Gewichte verwenden
- `FusionResult`-Erweiterung: neues Feld `boolean adaptiveWeightsUsed`
- `PipelineFactory`-Anpassung: instanziiert `ThompsonSamplingBandit` wenn `enabled=true`
- Integrationstests: `DecisionArbiter` mit `ThompsonSamplingBandit` liefert bounded Gewichte

**Out of Scope:**

- Persistierung der Beta-Parameter (ODIN-082c)
- Anbindung an Counterfactual-Reward-Signal / `updateReward()` (ODIN-082c)
- `ThompsonSamplingBandit` Kern-Implementierung (ODIN-082a)
- Visualisierung der Beta-Verteilungen im Frontend (separate Story)

## Akzeptanzkriterien

### Konfiguration (BrainProperties)

- [ ] Neues Nested-Record `AdaptiveWeightProperties` in `BrainProperties`:
  ```java
  record AdaptiveWeightProperties(
      boolean enabled,
      @DecimalMin("0.0") double priorAlpha,
      @DecimalMin("0.0") double priorBeta,
      @Min(0) int minTradesForActivation,
      @DecimalMin("0.0") double minWeight,
      @DecimalMin("0.0") double maxWeight
  )
  ```
- [ ] Default-Werte in `odin-app/src/main/resources/application.properties`:
  ```properties
  odin.brain.arbiter.adaptive-weights.enabled=false
  odin.brain.arbiter.adaptive-weights.prior-alpha=2.0
  odin.brain.arbiter.adaptive-weights.prior-beta=2.0
  odin.brain.arbiter.adaptive-weights.min-trades-for-activation=20
  odin.brain.arbiter.adaptive-weights.min-weight=0.20
  odin.brain.arbiter.adaptive-weights.max-weight=0.80
  ```
- [ ] `BrainProperties` hat neues Feld `@NotNull @Valid AdaptiveWeightProperties adaptiveWeights`
- [ ] `@Validated` bleibt auf `BrainProperties`

### FusionResult-Erweiterung

- [ ] `FusionResult` bekommt neues Feld `boolean adaptiveWeightsUsed`
- [ ] Alle bestehenden `new FusionResult(...)` Aufrufe in `DecisionArbiter` werden angepasst (neues Feld als letztes Parameter)
- [ ] `FusionResult.noAction()` Factory-Methode wird angepasst (adaptiveWeightsUsed = false)
- [ ] Das neue Feld wird in `DecisionArbiter.logFusion()` in den JSON-Payload aufgenommen: `"adaptiveWeightsUsed":true/false`
- [ ] JavaDoc des Records aktualisiert

### DecisionArbiter-Erweiterung

- [ ] `DecisionArbiter` bekommt optionalen Konstruktor-Parameter `ThompsonSamplingBandit bandit` (nullable — null = statische Gewichte)
- [ ] Bestehende Konstruktoren bleiben erhalten (Rueckwaertskompatibilitaet fuer bestehende Tests)
- [ ] Neues private Feld `ThompsonSamplingBandit bandit` (nullable)
- [ ] Neues private Feld `int adaptiveWeightsMinTrades` aus Konfiguration
- [ ] Neues private Feld `boolean adaptiveWeightsEnabled` aus Konfiguration
- [ ] Neue private Methode `boolean shouldUseAdaptiveWeights()`: `bandit != null && adaptiveWeightsEnabled && bandit.getTotalTradeCount() >= adaptiveWeightsMinTrades`
- [ ] Methode `ThompsonSamplingBandit.getTotalTradeCount()` muss in ODIN-082a ergaenzt werden (gibt Summe aller Updates zurueck — fuer den Fallback-Check)
- [ ] In `buildEntryFusion()`: wenn `shouldUseAdaptiveWeights()`, rufe `bandit.sampleWeights(fusedRegime)` auf und uebergib die Gewichte an `FusionResult` (stochastische Gewichte); sonst statische Gewichte (unveraendert, aktuell implizit 50/50 — ODIN-074 liefert echte Werte wenn vorhanden)
- [ ] `FusionResult` enthaelt `adaptiveWeightsUsed = shouldUseAdaptiveWeights()`
- [ ] **Hinweis zum aktuellen Stand:** Derzeit nutzt `DecisionArbiter` keine expliziten quantWeight/llmWeight Felder in `FusionResult` — diese Sub-Story fuehrt sie als Traceability-Felder ein. Die eigentliche Gewichts-Verwendung (Entry-Score-Fusion) wird in einem separaten Task spezifiziert — **der Implementierer prueft in `DecisionArbiter` wie Gewichte aktuell einfliessen und passt entsprechend an.**

### ThompsonSamplingBandit-Erweiterung (Rueckwirkung auf 082a)

- [ ] `ThompsonSamplingBandit.getTotalTradeCount()` — zaehlt die Summe aller Updates ueber alle (Regime, Arm)-Paare (zaehlt wie oft `updateReward()` aufgerufen wurde); dient dem Fallback-Schwellenwert-Check
- [ ] Interner `AtomicInteger totalUpdateCount` der bei jedem `updateReward()` um 1 erhoehen wird

### PipelineFactory-Anpassung

- [ ] `PipelineFactory` instanziiert `ThompsonSamplingBandit` wenn `brainProperties.adaptiveWeights().enabled() == true`
- [ ] `ThompsonSamplingBandit`-Instanz wird in `DecisionArbiter`-Konstruktor uebergeben
- [ ] Wenn `enabled == false`: `null` wird uebergeben (bestehender Code-Pfad ohne adaptive Gewichte)
- [ ] `PipelineFactory` hat Zugriff auf `BrainProperties` (bereits vorhanden)

### Integrationstests

- [ ] `DecisionArbiterBanditIntegrationTest`:
  - `DecisionArbiter` mit `ThompsonSamplingBandit` (enabled, genuegend Trades) liefert `FusionResult` mit `adaptiveWeightsUsed=true`
  - Gewichte in `FusionResult` liegen in `[0.20, 0.80]`
  - Mit `bandit=null` bleibt `adaptiveWeightsUsed=false`
  - Mit `totalTradeCount < minTradesForActivation` bleibt `adaptiveWeightsUsed=false` (Fallback)
  - 100 aufeinanderfolgende Decisions mit aktivem Bandit: alle Gewichte in Safety-Bounds

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `DecisionArbiter` | `odin-brain/src/main/java/de/its/odin/brain/arbiter/DecisionArbiter.java` | Optionaler `ThompsonSamplingBandit`, `shouldUseAdaptiveWeights()`, `adaptiveWeightsUsed` in FusionResult |
| `FusionResult` | `odin-brain/src/main/java/de/its/odin/brain/arbiter/FusionResult.java` | Neues Feld `boolean adaptiveWeightsUsed` |
| `BrainProperties` | `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` | Neues `AdaptiveWeightProperties`-Record + Feld |
| `PipelineFactory` | `odin-core/src/main/java/de/its/odin/core/pipeline/PipelineFactory.java` | Instanziierung und Injektion des `ThompsonSamplingBandit` |
| `ThompsonSamplingBandit` | `odin-brain/src/main/java/de/its/odin/brain/arbiter/ThompsonSamplingBandit.java` | Neues `getTotalTradeCount()` + interner `AtomicInteger` |

### Konstruktor-Varianten DecisionArbiter

```java
// Bestehend (bleibt erhalten):
public DecisionArbiter(GateCascadeEvaluator, BrainProperties, IntentSigner)
public DecisionArbiter(GateCascadeEvaluator, BrainProperties, IntentSigner, DiagnosticSink)

// Neu (Bandit-faehig):
public DecisionArbiter(GateCascadeEvaluator, BrainProperties, IntentSigner, DiagnosticSink, ThompsonSamplingBandit bandit)
```

Alternativ: Der bestehende 4-Parameter-Konstruktor wird intern auf den 5-Parameter-Konstruktor delegiert mit `bandit=null`.

### Konfigurationsnamensraum

`odin.brain.arbiter.adaptive-weights.*` — Namespace-Segment `arbiter` ist neu. Sicherstellen dass `BrainProperties` das Nested-Record korrekt mapped.

### Achtung: Stochastik und Tests

Thompson Sampling ist per Design stochastisch — `sampleWeights()` kann bei jedem Aufruf andere Werte liefern. Fuer deterministische Tests: `ThompsonSamplingBandit`-Testinstanz mit festem Random-Seed erstellen (Apache Commons Math `BetaDistribution` akzeptiert einen `RandomGenerator` — in 082a bereits als optionalen Konstruktorparameter vorsehen oder Test-Subklasse nutzen).

## Konzept-Referenzen

- `theme-backlog.md` Thema 22: "Adaptive Gewichtung via Bandit-Algorithmen (Thompson Sampling)" — Integration in Arbiter, Fallback-Logik
- `docs/backend/architecture/05-llm-integration.md` — Fusion-Protokoll, Arbiter-Struktur, `FusionResult`
- `docs/backend/architecture/06-rules-engine.md` — `DecisionArbiter` im Decision-Loop-Kontext

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln (odin-core haengt von odin-brain ab, nicht umgekehrt)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln: Records fuer DTOs, `@ConfigurationProperties` als Record, `@Validated`, kein `var`

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api,odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (`AdaptiveWeightProperties`)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.arbiter.adaptive-weights.*`
- [ ] Port-Abstraktion: gegen Interfaces aus `de.its.odin.api.port` programmieren wo anwendbar

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `DecisionArbiter` mit aktivem/inaktivem Bandit (Schalter, Fallback-Schwellenwert)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `ThompsonSamplingBandit` in Unit-Tests (deterministisch)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] `DecisionArbiterBanditIntegrationTest`: reale Klassen, kein Mocking — alle Szenarien aus Akzeptanzkriterien
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der den vollstaendigen Decision-Zyklus mit aktivem Bandit abdeckt

### 2.4 Tests — Datenbank (nicht zutreffend)
- Diese Sub-Story hat keinen Datenbankzugriff — Pflichtpunkt entfaellt.

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: `DecisionArbiter.java` (relevant), `FusionResult.java`, `BrainProperties.java`, Test-Klassen, Akzeptanzkriterien
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert wenn `getTotalTradeCount()` genau auf dem Schwellenwert ist? Wie verhaelt sich der Arbiter bei schnellem Wechsel enabled/disabled? Race-Condition zwischen `sampleWeights()` und `updateReward()` in Multi-Pipeline-Setup?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review — `DecisionArbiter`-Erweiterung auf Bugs: Null-Safety fuer `bandit`? Korrekte Konstruktor-Delegation? `FusionResult`-Aenderung rueckwaertskompatibel mit bestehenden Tests?
- [ ] Dimension 2: Konzepttreue-Review — Entspricht die Fallback-Logik (minTradesForActivation) dem Konzept in `theme-backlog.md` Thema 22? Ist der Konfigurationsschalter-Default (`false`) korrekt?
- [ ] Dimension 3: Praxis-Review — Was passiert in Multi-Pipeline-Setup (2 Pipelines teilen sich einen Bandit oder nicht)? Soll der Bandit pro Pipeline oder global sein?
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis vorhanden
- [ ] Alle Pflichtabschnitte enthalten: Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review
- [ ] Wird live waehrend der Arbeit aktualisiert

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### Multi-Pipeline-Frage (eskalieren wenn unklar)

ODIN hat bis zu 2-3 parallele Pipelines. Der `ThompsonSamplingBandit` lernt aus Trade-Outcomes. Frage: Soll ein globaler Bandit (ueber alle Pipelines geteilt) oder ein Bandit pro Pipeline verwendet werden? Argument fuer global: mehr Lern-Daten. Argument fuer pro-Pipeline: instrument-spezifisches Lernverhalten. Default-Annahme: **Ein globaler Bandit pro `PipelineFactory`** (alle Pipelines einer PipelineFactory-Instanz teilen sich einen Bandit). Falls das im Gemini-Review oder ChatGPT-Sparring als Problem identifiziert wird: eskalieren und User fragen.

### Bestehende DecisionArbiter-Tests nicht brechen

`FusionResult` bekommt ein neues Feld. Alle bestehenden `new FusionResult(...)` und `FusionResult.noAction(...)` Aufrufe muessen angepasst werden. Suche mit: `grep -r "new FusionResult\|FusionResult.noAction" odin-brain/src/` um alle Aufrufstellen zu finden.

### Kein Gewichts-Scoring heute

Der `DecisionArbiter` hat aktuell keinen expliziten "Quant-Score-Gewichtung vs LLM-Score-Gewichtung"-Mechanismus der direkt an Zahlen gebunden ist (die Fusion ist eher logisch/dual-key). Die `BanditWeights` werden vorerst als Traceability-Felder in `FusionResult` abgelegt und ins EventLog geschrieben. Die tatsaechliche Verwendung der Gewichte in der Score-Fusion ist eine separate konzeptionelle Entscheidung. Bei Unsicherheit: User fragen bevor implementiert wird.
