# ODIN-082a: Thompson Sampling Kern-Algorithmus

**Modul:** odin-brain
**Phase:** 1
**Abhaengigkeiten:** ODIN-071 (Post-Trade Counterfactual — definiert den Reward-Signal-Kontrakt), ODIN-074 (Regime-Dependent Arbiter Weighting — liefert die `Regime`-Enum-Werte und `RegimeWeight`-Infrastruktur)
**Geschaetzter Umfang:** M

---

## Kontext

ODIN-082 (Adaptive Weighting via Bandit) wird in 3 Sub-Stories aufgeteilt. Diese erste Sub-Story implementiert den reinen mathematischen Kern des Thompson-Sampling-Algorithmus als in sich geschlossene Komponente — ohne ODIN-spezifische Abhaengigkeiten jenseits der `Regime`-Enum. Das Ergebnis ist eine getestete, thread-sichere `ThompsonSamplingBandit`-Klasse mit einer einfachen `BetaDistribution`, die in Sub-Story 082b in den `DecisionArbiter` integriert wird.

## Scope

**In Scope:**

- `BetaDistribution`: Wrapper um Apache Commons Math `BetaDistribution.sample()`, mit konfigurierbarem alpha/beta und `sample()`-Methode
- `ThompsonSamplingBandit`: Kern-Algorithmus — Beta-Verteilungen pro Arm (QUANT, LLM) pro Regime (5 Regimes = 10 Verteilungen), `sampleWeights(Regime)`, `updateReward(Regime, BanditArm, boolean)`, Safety-Bound-Clipping
- Prior-Initialisierung: `Beta(alpha0, beta0)` mit konfigurierbarem Prior (Default: alpha=2.0, beta=2.0)
- Safety Bounds: Gewichte werden auf `[minWeight, maxWeight]` geclippt (Default: [0.20, 0.80]), konfigurierbar
- `BanditArm`-Enum: `QUANT`, `LLM`
- `BanditWeights`-Record: `quantWeight`, `llmWeight` als Ergebnis von `sampleWeights()`
- Thread-Safety: `ConcurrentHashMap` fuer Beta-Parameter; `sampleWeights()` ist read-only (nur sampling), `updateReward()` ist synchronized
- Unit-Tests: Sampling-Statistik, Update-Logik, Safety Bounds, Edge Cases

**Out of Scope:**

- Integration in `DecisionArbiter` (ODIN-082b)
- Persistierung und Anbindung an Counterfactual-Reward-Signal (ODIN-082c)
- EXP3 oder andere Bandit-Algorithmen
- Discount-Faktor / Windowed Counts (Non-Stationaritaet — erst relevant nach hunderten Trades)
- Konfiguration via `BrainProperties` (erfolgt in 082b)

## Akzeptanzkriterien

### BetaDistribution

- [ ] Neue Klasse `BetaDistribution` in `de.its.odin.brain.arbiter`
- [ ] Konstruktor: `BetaDistribution(double alpha, double beta)` — valide alpha/beta > 0 werden per `Objects.requireNonNull` und Guard-Clause sichergestellt
- [ ] Methode `double sample()` — liefert eine Stichprobe aus der Beta-Verteilung via Apache Commons Math `org.apache.commons.math3.distribution.BetaDistribution`
- [ ] Methode `BetaDistribution withUpdate(boolean profitable)` — liefert neue `BetaDistribution`-Instanz (immutable Update): profitable → alpha+1, sonst beta+1
- [ ] Methode `double getAlpha()` und `double getBeta()` fuer Tests und Persistierung
- [ ] JavaDoc vollstaendig

### BanditArm-Enum

- [ ] Neues Enum `BanditArm` in `de.its.odin.brain.arbiter` mit Werten `QUANT` und `LLM`
- [ ] JavaDoc auf Klasse und beiden Werten

### BanditWeights-Record

- [ ] Neues Record `BanditWeights` in `de.its.odin.brain.arbiter` mit Feldern `double quantWeight` und `double llmWeight`
- [ ] Invariante: `quantWeight + llmWeight == 1.0` wird nicht explizit geprueft (LLM-Wert = 1.0 - quantWeight), aber die Berechnung muss das garantieren
- [ ] JavaDoc vollstaendig

### ThompsonSamplingBandit

- [ ] Neue Klasse `ThompsonSamplingBandit` in `de.its.odin.brain.arbiter`
- [ ] Interner Zustand: `ConcurrentHashMap<BanditKey, BetaDistribution>` — `BanditKey` ist ein privater Record mit `(Regime, BanditArm)` als Key
- [ ] Initialisierung bei erstem Zugriff auf ein (Regime, Arm)-Paar mit konfigurierten Prior-Werten (Lazy-Init via `computeIfAbsent`)
- [ ] Methode `BanditWeights sampleWeights(Regime regime)`:
  - Zieht `theta_quant ~ Beta(alpha_quant_R, beta_quant_R)`
  - Zieht `theta_llm ~ Beta(alpha_llm_R, beta_llm_R)`
  - Berechnet `rawQuantWeight = theta_quant / (theta_quant + theta_llm)`
  - Clippt auf `[minWeight, maxWeight]`
  - `llmWeight = 1.0 - clippedQuantWeight`
  - Gibt `BanditWeights(clippedQuantWeight, llmWeight)` zurueck
- [ ] Methode `void updateReward(Regime regime, BanditArm arm, boolean profitable)`:
  - Aktualisiert `BetaDistribution` fuer (regime, arm): ersetzt durch `distribution.withUpdate(profitable)`
  - `synchronized`-Block auf dem ConcurrentHashMap-Eintrag fuer den spezifischen Key
- [ ] Methode `BetaDistribution getDistribution(Regime regime, BanditArm arm)` — package-private fuer Tests und Persistierung
- [ ] Methode `void setDistribution(Regime regime, BanditArm arm, BetaDistribution distribution)` — package-private fuer Startup-Wiederherstellung (ODIN-082c)
- [ ] Konstruktor: `ThompsonSamplingBandit(double priorAlpha, double priorBeta, double minWeight, double maxWeight)`
- [ ] JavaDoc vollstaendig

### Unit-Tests

- [ ] `BetaDistributionTest`:
  - `sample()` liegt im Intervall `[0.0, 1.0]`
  - `withUpdate(true)` ergibt alpha+1, beta unveraendert
  - `withUpdate(false)` ergibt beta+1, alpha unveraendert
  - Unveraenderlichkeit: urspruengliche Instanz bleibt nach `withUpdate()` unmodifiziert
- [ ] `ThompsonSamplingBanditTest`:
  - Bei uninformativem Prior (2.0, 2.0): 1000 Samples im Mittel nahe bei 0.5 (Toleranz +-0.1)
  - Nach 10 profitablen QUANT-Updates in Regime TREND_UP: QUANT-Alpha > initialer Alpha
  - Safety Bounds: Sample liegt immer in `[minWeight, maxWeight]` (100 Samples)
  - `updateReward()` fuer QUANT-Arm aendert nur QUANT-Verteilung, LLM-Arm bleibt unveraendert
  - Getrennte Regime: Update in TREND_UP aendert nichts in RANGE_BOUND
  - Thread-Safety: 10 parallele Threads die gleichzeitig updaten produzieren konsistenten Zustand (keine Exceptions)
  - 5 profitable QUANT-Trades + 2 unprofitable → `Beta(7.0, 4.0)` fuer QUANT-Arm (priorAlpha=2.0, priorBeta=2.0)
  - `getDistribution()` gibt die korrekte Instanz zurueck
  - `setDistribution()` setzt eine externe Instanz korrekt (Round-Trip fuer Persistierung)

## Technische Details

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `BetaDistribution` | `de.its.odin.brain.arbiter` | Immutable Beta-Verteilung; Wrapper um Apache Commons Math |
| `BanditArm` | `de.its.odin.brain.arbiter` | Enum fuer die zwei Bandit-Arme (QUANT, LLM) |
| `BanditWeights` | `de.its.odin.brain.arbiter` | Record fuer das Sampling-Ergebnis |
| `ThompsonSamplingBandit` | `de.its.odin.brain.arbiter` | Kern-Algorithmus: 10 Beta-Verteilungen, Sampling, Updates |

### Algorithmus

```
Pro sampleWeights(Regime R):
1. theta_quant ~ Beta(alpha_QUANT_R, beta_QUANT_R)
2. theta_llm   ~ Beta(alpha_LLM_R,   beta_LLM_R)
3. rawQuantWeight = theta_quant / (theta_quant + theta_llm)
4. clippedWeight  = clip(rawQuantWeight, minWeight, maxWeight)
5. return BanditWeights(clippedWeight, 1.0 - clippedWeight)

Pro updateReward(Regime R, BanditArm arm, boolean profitable):
6. distribution = getDistribution(R, arm)
7. setDistribution(R, arm, distribution.withUpdate(profitable))
```

### Apache Commons Math Dependency

Pruefen ob `org.apache.commons:commons-math3` bereits im odin-brain POM vorhanden ist. Falls nicht: in `odin-brain/pom.xml` als Dependency ergaenzen (ohne Version — wird im Parent POM verwaltet). Aktuelle stabile Version: `3.6.1`.

### Thread-Safety-Design

- `ConcurrentHashMap` als Container — kein globales Lock
- `sampleWeights()`: Liest nur (kein State-Update) — sicher ohne Lock, da `ConcurrentHashMap.get()` thread-safe ist und `BetaDistribution` immutable ist
- `updateReward()`: Nutzt `ConcurrentHashMap.compute()` fuer atomares Replace — kein explizites `synchronized` noetig

### Regime-Enum

`Regime` ist definiert in `de.its.odin.api.model.Regime` (odin-api). Alle 5 Werte sind relevant: `TREND_UP`, `TREND_DOWN`, `RANGE_BOUND`, `HIGH_VOLATILITY`, `UNCERTAIN`.

## Konzept-Referenzen

- `theme-backlog.md` Thema 22: "Adaptive Gewichtung via Bandit-Algorithmen (Thompson Sampling)" — Beta-Prior-Ansatz, Update-Regel, Algorithmus-Pseudocode
- Research-Hybrid Empfehlung 5: Thompson Sampling fuer adaptive Signal-Gewichtung

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Namensregeln (`de.its.odin.brain.arbiter`)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln: kein `var`, keine Magic Numbers, Records fuer DTOs, ENUM statt String

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (`BanditWeights`)
- [ ] ENUM statt String fuer endliche Mengen (`BanditArm`)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `de.its.odin.brain.arbiter` fuer alle neuen Klassen
- [ ] Port-Abstraktion: keine Ports noetig in dieser Sub-Story (reine Algorithmus-Schicht)

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `BetaDistribution` (sample range, withUpdate-Logik, Unveraenderlichkeit)
- [ ] Unit-Tests fuer `ThompsonSamplingBandit` (Sampling-Statistik, Update-Logik, Safety Bounds, Thread-Safety, Regime-Isolation)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Keine Spring-Kontexte noetig — Plain Java Tests
- [ ] Alle in den Akzeptanzkriterien aufgelisteten Test-Cases implementiert

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] `ThompsonSamplingBanditIntegrationTest`: Vollstaendiger Zyklus — Prior-Init → mehrere Updates → `sampleWeights()` liefert shifted Gewichte (kein Mocking, echte Klassen)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank (nicht zutreffend)
- Diese Sub-Story hat keinen Datenbankzugriff — Pflichtpunkt entfaellt.

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: `BetaDistribution.java`, `ThompsonSamplingBandit.java`, alle Test-Klassen, Akzeptanzkriterien
- [ ] ChatGPT nach Grenzfaellen gefragt: identische Verteilungen, extreme Priors (alpha=0.001), Overflow bei vielen Updates (alpha=10000), Division-by-Zero wenn theta_quant + theta_llm = 0
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review — Pruefe auf Bugs: Random-Number-Generation korrekt? Numerical Stability bei extremen alpha/beta? Null-Safety? `ConcurrentHashMap.compute()` korrekt fuer atomic replace?
- [ ] Dimension 2: Konzepttreue-Review — Entspricht die Implementierung dem Thompson-Sampling-Algorithmus aus `theme-backlog.md` Thema 22? Stimmen Prior-Defaults, Safety-Bound-Logik, Update-Regel?
- [ ] Dimension 3: Praxis-Review — Nicht behandelte Themen: Was passiert bei Regime-Wechsel waehrend eines laufenden Trades? Wie verhaelt sich der Bandit wenn alle Trades in einem Regime profitabel sind (extreme Konvergenz)?
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis vorhanden
- [ ] Alle Pflichtabschnitte enthalten: Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review
- [ ] Wird live waehrend der Arbeit aktualisiert

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### Beta-Verteilung: Apache Commons Math

Java hat keine Standard-Beta-Distribution in `java.util`. Apache Commons Math bietet `org.apache.commons.math3.distribution.BetaDistribution`. Diese Klasse akzeptiert `alpha` und `beta` im Konstruktor und bietet `sample()` direkt. Die Klasse ist NICHT thread-safe — deshalb ist jede `BetaDistribution`-Instanz in `ThompsonSamplingBandit` immutable: `withUpdate()` erstellt eine neue Instanz. Der `ConcurrentHashMap.compute()`-Aufruf ersetzt atomar die alte Instanz durch die neue.

### Immutable BetaDistribution vs. Mutable

Die Entscheidung fuer Immutabilitaet (statt Mutable State) ist bewusst: `BetaDistribution.withUpdate()` erstellt eine neue Instanz. Vorteil: kein Lock auf Lese-Pfad (`sampleWeights()` greift ohne Lock auf den aktuellen Snapshot zu). Nachteil: GC-Druck durch viele kurzlebige Objekte. Bei max. 800 Updates pro Jahr (konservative Schaetzung) ist das vernachlaessigbar.

### Division-by-Zero Schutz

Wenn `theta_quant + theta_llm == 0.0` (theoretisch unmoeglich, aber defensiv behandeln): Fallback auf `rawQuantWeight = 0.5`. Apache Commons Math liefert bei `alpha → 0` und `beta → 0` gelegentlich sehr kleine positive Werte, kein echtes Zero.

### Sequentielle Abhaengigkeit zu 082b und 082c

Diese Sub-Story MUSS abgeschlossen und committed sein, bevor 082b (Arbiter-Integration) und 082c (Persistierung) beginnen koennen. Der Grund: 082b baut direkt auf `ThompsonSamplingBandit` und `BanditWeights` auf. 082c baut auf `getDistribution()`/`setDistribution()` auf.
