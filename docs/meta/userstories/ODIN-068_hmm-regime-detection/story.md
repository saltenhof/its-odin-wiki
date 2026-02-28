# ODIN-068: HMM Regime Detection

## Modul

odin-brain, odin-api

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

L

## Kontext

ODINs `RegimeResolver` bestimmt das Marktregime deterministisch ueber EMA-Kreuzung, ADX und VWAP-Position. Regime-Confidence wird heuristisch aus Signalstaerke abgeleitet, nicht empirisch kalibriert. Hidden Markov Models (HMM) liefern stattdessen kalibrierte Wahrscheinlichkeitsverteilungen ueber Regime-Zustaende — z.B. P(TREND_UP) = 0.73, P(RANGE) = 0.22, P(BEAR) = 0.05. Das eliminiert scharfe Regime-Grenzen und macht die Hysterese-Heuristik (2-Bar-Confirmation) obsolet. Laut Literatur bringt HMM-basierte Regime-Erkennung signifikante Verbesserungen (Sharpe 1.9, Drawdown-Reduktion von 56% auf 24% laut QuantConnect).

## Scope

### In Scope

- **3-State-HMM** (Bull/Neutral/Bear) trainiert auf 5-Min-Returns
- **Forward-Algorithmus** fuer Online-Regime-Wahrscheinlichkeiten (kein Viterbi, da wir aktuelle Zustandsverteilung brauchen, nicht optimalen Pfad)
- **Baum-Welch-Training** auf historischen Daten (Offline, vor RTH-Open)
- **Integration als sekundaerer Regime-Provider** neben bestehendem `RegimeResolver`
- **Regime-Wahrscheinlichkeiten als kalibrierte `regimeConfidence`** fuer den Arbiter
- **Konfiguration** ob HMM oder deterministischer Resolver primaer ist (Feature-Flag)

### Out of Scope

- Statistical Jump Models als HMM-Alternative (spaetere Evaluierung)
- Online-Training waehrend der Trading-Session (nur Forward-Pass online, Training offline)
- Eigene HMM-Library (bestehende Java-Libraries verwenden, z.B. Apache Commons Math oder Smile ML)
- Mehr als 3 States (z.B. 5 States mit HIGH_VOL, CRASH — spaetere Erweiterung)
- Automatische Regime-Gewichtung im Arbiter (das ist ODIN-074, Thema 12)

## Akzeptanzkriterien

- [ ] `HmmRegimeDetector` implementiert Forward-Algorithmus auf 3-State-HMM (Bull, Neutral, Bear)
- [ ] `HmmRegimeDetector.detect(returns[])` liefert `RegimeProbabilities` Record mit `P(Bull)`, `P(Neutral)`, `P(Bear)` — Summe = 1.0 (+/- 1e-9)
- [ ] `HmmTrainer` implementiert Baum-Welch-Algorithmus und liefert trainierte Modell-Parameter (Transition-Matrix A, Emissions-Parameter B, Initial-Verteilung pi)
- [ ] Trainierte Parameter werden als JSON serialisiert und koennen vor RTH-Open geladen werden
- [ ] `RegimeProbabilities` wird in `RegimeResolver` als alternative/ergaenzende Regime-Bestimmung integriert
- [ ] Feature-Flag `odin.brain.regime.hmm-enabled=false` (Default: aus, opt-in)
- [ ] Wenn HMM aktiv: `regimeConfidence` = `max(P(Bull), P(Neutral), P(Bear))` (hoechste Wahrscheinlichkeit)
- [ ] Wenn HMM aktiv: `regime` = Zustand mit hoechster Wahrscheinlichkeit, Mapping: Bull->TREND_UP, Bear->TREND_DOWN, Neutral->RANGE_BOUND
- [ ] Unit-Test verifiziert: Forward-Algorithmus mit bekannten HMM-Parametern liefert korrekte Posterior-Wahrscheinlichkeiten (Referenzwerte aus Lehrbuch)
- [ ] Unit-Test verifiziert: Baum-Welch konvergiert fuer synthetische Daten (3 Cluster mit unterschiedlichen Mittelwerten/Varianzen)
- [ ] Unit-Test verifiziert: Summe aller Regime-Wahrscheinlichkeiten = 1.0

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `HmmRegimeDetector` | odin-brain | `de.its.odin.brain.regime` | Forward-Algorithmus. Input: trainierte Parameter + neue Returns. Output: RegimeProbabilities. Inkrementell (haelt internen State fuer Forward-Pass) |
| `HmmTrainer` | odin-brain | `de.its.odin.brain.regime` | Baum-Welch-Training. Input: historische 5-Min-Returns. Output: HmmParameters (A, B, pi). Offline-Berechnung |
| `HmmParameters` | odin-brain | `de.its.odin.brain.regime` | Record: transitionMatrix (double[3][3]), emissionMeans (double[3]), emissionVariances (double[3]), initialDistribution (double[3]) |
| `RegimeProbabilities` | odin-api | `de.its.odin.api.dto` | Record: bullProbability, neutralProbability, bearProbability. Validiert Summe = 1.0 |
| `HmmParameterStore` | odin-brain | `de.its.odin.brain.regime` | Serialisiert/deserialisiert HmmParameters als JSON. Speicherort konfigurierbar |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `RegimeResolver` (`de.its.odin.brain.rules.RegimeResolver`) | Neuer optionaler Konstruktor-Parameter `HmmRegimeDetector`. Wenn vorhanden und aktiviert: HMM-Regime statt deterministischem Regime verwenden |
| `BrainProperties` (`de.its.odin.brain.config.BrainProperties`) | Neue Nested-Record `HmmProperties` mit `enabled` (boolean), `trainingLookbackDays` (int), `convergenceThreshold` (double) |
| `PipelineFactory` (`de.its.odin.core.pipeline.PipelineFactory`) | HmmRegimeDetector erstellen und an RegimeResolver uebergeben wenn konfiguriert |
| `IndicatorResult` (`de.its.odin.api.dto.IndicatorResult`) | Optionales Feld `RegimeProbabilities hmmRegime` fuer Downstream-Logging |

### Mathematisches Modell

**3-State Gaussian HMM:**
- States: S = {Bull, Neutral, Bear}
- Emissions: 5-Min-Returns ~ N(mu_s, sigma_s^2) pro State s
- Transition: A[i][j] = P(S_t = j | S_{t-1} = i)

**Forward-Algorithmus (Online):**
```
alpha_t(j) = [sum_i alpha_{t-1}(i) * A[i][j]] * B(j, o_t)
P(S_t = j | O_1..t) = alpha_t(j) / sum_k alpha_t(k)
```

### Konfiguration

```properties
odin.brain.regime.hmm-enabled=false
odin.brain.regime.hmm-training-lookback-days=60
odin.brain.regime.hmm-convergence-threshold=1e-6
odin.brain.regime.hmm-max-iterations=100
odin.brain.regime.hmm-parameters-path=data/hmm-params.json
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 4: HMM-basierte Regime-Erkennung, Abschnitte IST-Zustand, SOLL-Zustand, QuantConnect-Referenz
- `docs/backend/architecture/06-rules-engine.md` — Regime-Bestimmung, RegimeResolver-Architektur
- `docs/backend/architecture/04-kpi-engine.md` — Indikator-Berechnung (HMM integriert sich analog)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen (neues Package `de.its.odin.brain.regime`)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Instanziierungsmodell (Pro-Pipeline-POJOs), Port-Abstraktion

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (HmmParameters, RegimeProbabilities)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.regime.*`
- [ ] Port-Abstraktion eingehalten

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `HmmRegimeDetector` (Forward-Algorithmus mit Referenzwerten)
- [ ] Unit-Tests fuer `HmmTrainer` (Konvergenz auf synthetischen Daten)
- [ ] Unit-Tests fuer `HmmParameterStore` (Serialisierung/Deserialisierung)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: `RegimeResolver` mit `HmmRegimeDetector` End-to-End
- [ ] Integrationstest: Training -> Serialisierung -> Deserialisierung -> Detection
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases diskutiert (numerische Unterlauefe, degenerierte Parameter)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (numerische Stabilitaet, Log-Space-Berechnung)
- [ ] Dimension 2: Konzepttreue-Review (HMM-Theorie vs. Implementierung)
- [ ] Dimension 3: Praxis-Review (Regime-Switching-Latenz, Parameter-Drift)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Numerische Stabilitaet ist das Hauptrisiko.** Forward-Algorithmus multipliziert viele kleine Wahrscheinlichkeiten — Unterlauf nach wenigen hundert Schritten. MUSS im Log-Space berechnet werden (Log-Sum-Exp Trick). Referenz: Rabiner (1989), Section III-A.
- **Keine eigene HMM-Library:** Pruefen ob Apache Commons Math (`org.apache.commons.math3.distribution`) oder Smile ML (`smile.sequence.HMM`) geeignet ist. Wenn keine Library passend: eigene Implementierung (Forward + Baum-Welch sind jeweils ~50 Zeilen).
- **Emissions-Modell:** Gauss-Verteilung pro State ist Standard fuer Financial Returns. Jeder State hat eigenes `mu` (Mittelwert) und `sigma^2` (Varianz). Bull: mu > 0, niedrige Varianz. Bear: mu < 0, hohe Varianz. Neutral: mu ~ 0, niedrige Varianz.
- **Training-Daten:** 60 Handelstage * 78 Bars = ~4700 Datenpunkte. Ausreichend fuer 3-State-HMM mit 2 Emissions-Parametern pro State.
- **Feature-Flag:** HMM ist Default-off. Das erlaubt schrittweise Aktivierung und A/B-Vergleich im Backtest (deterministisch vs. HMM).
- **Mapping auf bestehende Regime-Enum:** Bull -> TREND_UP, Bear -> TREND_DOWN, Neutral -> RANGE_BOUND. Die 20 Subregimes werden weiterhin deterministisch bestimmt; HMM ersetzt nur die Top-Level-Regime-Zuordnung.
