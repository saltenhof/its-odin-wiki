# ODIN-082: Adaptive Weighting via Bandit (Thompson Sampling)

**Modul:** odin-brain, odin-api, odin-core
**Phase:** 1
**Abhaengigkeiten:** ODIN-071 (Post-Trade Counterfactual — liefert die Reward-Signale), ODIN-074 (Regime-Dependent Arbiter Weighting — liefert die statische Gewichts-Infrastruktur)
**Geschaetzter Umfang:** XL

---

## Kontext

ODIN-074 fuehrt statische regime-abhaengige Gewichte fuer die Quant/LLM-Fusion ein. Diese Gewichte basieren auf Design-Annahmen und Research-Empfehlungen. Adaptive Gewichtung via Thompson Sampling (ein Multi-Armed-Bandit-Algorithmus) ersetzt die statischen Gewichte durch empirisch lernergeleitete Gewichte: Pro Signalquelle und Regime wird eine Beta-Prior-Verteilung gefuehrt, die nach jedem Trade-Outcome bayesianisch aktualisiert wird. Der Algorithmus balanciert Exploration (neue Gewichtungen ausprobieren) und Exploitation (bewaehrte Gewichtungen nutzen) automatisch.

## Scope

**In Scope:**

- Beta-Verteilungen (alpha, beta) pro Signalquelle (Quant, LLM) pro Regime (5 Regimes → 10 Verteilungen)
- Thompson Sampling: Pro Decision-Zyklus eine Zufallsstichprobe aus Beta(alpha, beta) → relatives Gewicht
- Bayesian Update nach Trade-Outcome: Profitabler Trade → alpha+1, Verlusttrade → beta+1
- Persistierung der Beta-Parameter (ueberleben Neustarts)
- Konfigurierbare Prior-Staerke und Reward-Definition
- Safety Bounds: Gewichte duerfen nie unter/ueber konfigurierbare Grenzen fallen
- Fallback auf statische Gewichte (ODIN-074) wenn zu wenig Daten vorliegen

**Out of Scope:**

- EXP3 oder andere Bandit-Algorithmen (nur Thompson Sampling)
- Gewichtung einzelner Indikatoren innerhalb der Quant-Seite (nur Quant-vs-LLM)
- Multi-Armed-Bandit ueber mehr als 2 Arms (nur 2: Quant, LLM)
- Automatischer Reset der Verteilungen (manueller Reset ueber Admin-API)
- Online-Visualisierung der Beta-Verteilungen im Frontend (separate Story)

## Akzeptanzkriterien

### Bandit-Infrastruktur

- [ ] Neue Klasse `ThompsonSamplingBandit` in `de.its.odin.brain.arbiter` mit:
  - `BetaDistribution` pro Arm (Quant, LLM) pro Regime (5 Regimes = 10 Verteilungen)
  - Methode `sampleWeights(Regime regime)` → `RegimeWeight(quantWeight, llmWeight)` via Thompson Sampling
  - Methode `updateReward(Regime regime, WinnerSide side, boolean profitable)` → Bayesian Update
- [ ] Prior-Initialisierung: `Beta(alpha0, beta0)` mit konfigurierbarem Prior (Default: alpha=2, beta=2 → uninformativer Prior)
- [ ] Safety Bounds: `quantWeight ∈ [0.20, 0.80]`, `llmWeight ∈ [0.20, 0.80]` — konfigurierbar
- [ ] Unit-Test: Bei uninformativem Prior (2, 2) sind Samples gleichverteilt um 0.5
- [ ] Unit-Test: Nach 10 profitablen Quant-Trades → Quant-Gewicht konsistent > 0.5
- [ ] Unit-Test: Safety Bounds werden eingehalten (Sample wird auf [0.20, 0.80] geclipped)

### Bayesian Update

- [ ] Reward-Signal aus Counterfactual-Daten (ODIN-071): `quantOnlyProfitable`, `llmInfluenceProfitable`
- [ ] Update-Regel: War der Quant-allein Trade profitabel → Quant-Arm: alpha+1. War er unprofitabel → Quant-Arm: beta+1. Analog fuer LLM.
- [ ] Updates erfolgen nach Trade-Close (P&L bekannt), nicht intraday
- [ ] Unit-Test: 5 profitable Quant-Trades + 2 unprofitable → Beta(7, 4) fuer Quant-Arm
- [ ] Unit-Test: Update mit `profitable=true` erhoeht alpha um genau 1

### Integration in DecisionArbiter

- [ ] `DecisionArbiter` akzeptiert wahlweise `RegimeWeightProfile` (statisch, ODIN-074) ODER `ThompsonSamplingBandit` (adaptiv)
- [ ] Konfigurationsschalter `odin.brain.arbiter.adaptive-weights.enabled` (Default: false → statische Gewichte)
- [ ] Wenn adaptiv aktiv: `sampleWeights(fusedRegime)` wird pro Decision-Zyklus aufgerufen
- [ ] `FusionResult` dokumentiert ob statische oder adaptive Gewichte verwendet wurden
- [ ] Fallback: Wenn Bandit weniger als N Trades Daten hat (Default: 20), werden statische Gewichte verwendet
- [ ] Integrationstest: DecisionArbiter mit ThompsonSamplingBandit liefert stochastische aber bounded Gewichte

### Persistierung

- [ ] Beta-Parameter (alpha, beta pro Arm pro Regime) werden nach jedem Update in DB persistiert
- [ ] Flyway-Migration: Neue Tabelle `bandit_state` mit Spalten: `regime VARCHAR`, `arm VARCHAR`, `alpha DOUBLE`, `beta DOUBLE`, `last_updated TIMESTAMP`
- [ ] Beim Startup werden gespeicherte Parameter geladen (falls vorhanden), sonst Prior
- [ ] Unit-Test: Persistierung → Neustart → geladen Parameter stimmen mit gespeicherten ueberein

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `DecisionArbiter` | `odin-brain/.../arbiter/DecisionArbiter.java` | Alternative Gewichtsquelle: Bandit statt statischem Profile |
| `FusionResult` | `odin-brain/.../arbiter/FusionResult.java` | Neues Feld `boolean adaptiveWeightsUsed` |
| `BrainProperties` | `odin-brain/.../config/BrainProperties.java` | Neues `AdaptiveWeightProperties` |
| `PipelineFactory` | `odin-core/.../pipeline/PipelineFactory.java` | Erzeugt/injiziert ThompsonSamplingBandit |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `ThompsonSamplingBandit` | `de.its.odin.brain.arbiter` | Kern-Algorithmus: Beta-Verteilungen, Sampling, Updates |
| `BetaDistribution` | `de.its.odin.brain.arbiter` | Einfache Beta-Verteilung mit `sample()` Methode (Apache Commons Math oder eigene Implementierung) |
| `BanditStateEntity` | `de.its.odin.brain.persistence` | JPA Entity fuer Persistierung der Beta-Parameter |
| `BanditStateRepository` | `de.its.odin.brain.persistence` | JPA Repository fuer BanditStateEntity |

### Algorithmus: Thompson Sampling

```
Pro Decision-Zyklus in Regime R:
1. θ_quant ~ Beta(α_quant_R, β_quant_R)
2. θ_llm   ~ Beta(α_llm_R,   β_llm_R)
3. quantWeight = clip(θ_quant / (θ_quant + θ_llm), 0.20, 0.80)
4. llmWeight   = 1.0 - quantWeight

Nach Trade-Close:
5. Wenn Quant-Only profitabel → α_quant_R += 1, sonst β_quant_R += 1
6. Wenn LLM-Einfluss profitabel → α_llm_R += 1, sonst β_llm_R += 1
7. Persistiere neue (α, β)
```

### Konfiguration

```properties
# Adaptive weighting via Thompson Sampling
odin.brain.arbiter.adaptive-weights.enabled=false
odin.brain.arbiter.adaptive-weights.prior-alpha=2.0
odin.brain.arbiter.adaptive-weights.prior-beta=2.0
odin.brain.arbiter.adaptive-weights.min-trades-for-activation=20
odin.brain.arbiter.adaptive-weights.min-weight=0.20
odin.brain.arbiter.adaptive-weights.max-weight=0.80
```

### Flyway-Migration

```sql
-- V0XX__create_bandit_state.sql
CREATE TABLE bandit_state (
    id          BIGSERIAL PRIMARY KEY,
    regime      VARCHAR(30) NOT NULL,
    arm         VARCHAR(10) NOT NULL,
    alpha       DOUBLE PRECISION NOT NULL,
    beta        DOUBLE PRECISION NOT NULL,
    trade_count INTEGER NOT NULL DEFAULT 0,
    last_updated TIMESTAMP WITH TIME ZONE NOT NULL,
    UNIQUE (regime, arm)
);
```

## Konzept-Referenzen

- `theme-backlog.md` Thema 22: "Adaptive Gewichtung via Bandit-Algorithmen (Thompson Sampling)" — IST-Zustand, SOLL-Zustand, Beta-Prior-Ansatz
- `theme-backlog.md` Thema 12: "Regime-abhaengige Arbiter-Gewichtung" — statische Vorstufe (ODIN-074)
- `theme-backlog.md` Thema 8: "Post-Trade Counterfactual" — Reward-Signal-Quelle (ODIN-071)
- `docs/backend/architecture/05-llm-integration.md` — Fusion-Protokoll, Arbiter
- Research-Hybrid Empfehlung 5: Thompson Sampling fuer adaptive Signal-Gewichtung

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Persistenz in Domain-Modul (DDD)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln (Records fuer DTOs, @ConfigurationProperties, Flyway fuer Migrationen)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api,odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.arbiter.adaptive-weights.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `ThompsonSamplingBandit` (Sampling, Updates, Safety Bounds)
- [ ] Unit-Tests fuer `BetaDistribution` (Sample-Statistik, Edge Cases)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: DecisionArbiter mit ThompsonSamplingBandit
- [ ] Integrationstest: Persistierung → Neustart → korrekte Wiederherstellung
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank
- [ ] Embedded-Postgres-Tests mit Zonky fuer BanditStateRepository
- [ ] Flyway-Migration wird im Test ausgefuehrt
- [ ] Repository-Methoden gegen echte DB getestet

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen, Akzeptanzkriterien, Tests
- [ ] Edge Cases: Identische Verteilungen, Extreme Priors, Overflow bei vielen Updates
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Random-Number-Generation, Numerical Stability)
- [ ] Dimension 2: Konzepttreue-Review (Thompson Sampling nach Literatur)
- [ ] Dimension 3: Praxis-Review (Non-Stationaritaet, Reward-Delay, Regime-Wechsel waehrend Trade)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

### XL-Story: Empfehlung fuer Sub-Story-Aufteilung

Diese Story ist XL und sollte in 3 Sub-Stories aufgeteilt werden:

1. **ODIN-082a: Thompson Sampling Kern** (M) — `ThompsonSamplingBandit`, `BetaDistribution`, Unit-Tests. Reine Mathematik, keine ODIN-Abhaengigkeiten.
2. **ODIN-082b: Arbiter-Integration** (M) — `DecisionArbiter`-Erweiterung, Konfiguration, `FusionResult`-Erweiterung, Fallback-Logik, Integrationstests.
3. **ODIN-082c: Persistierung + Reward-Anbindung** (M) — `BanditStateEntity`, Flyway-Migration, DB-Tests, Anbindung an Counterfactual-Reward-Signal (ODIN-071).

### Technische Hinweise

- **Beta-Verteilung:** Java hat keine Standard-Beta-Distribution. Optionen: (1) Apache Commons Math `BetaDistribution.sample()`, (2) Eigene Implementierung via Gamma-Verteilung: `X ~ Gamma(α, 1)`, `Y ~ Gamma(β, 1)` → `X / (X+Y) ~ Beta(α, β)`. Apache Commons Math ist die robustere Wahl. Pruefen ob die Dependency bereits im POM ist.
- **Thread-Safety:** Der `ThompsonSamplingBandit` wird von `DecisionArbiter` in verschiedenen Pipeline-Threads aufgerufen. Da `sampleWeights()` lesend ist (nur sampling, kein State-Update) und `updateReward()` nach Trade-Close asynchron passiert, muss die Synchronisation beruecksichtigt werden. Empfehlung: `ConcurrentHashMap` fuer Beta-Parameter oder `synchronized`-Bloecke.
- **Non-Stationaritaet:** Maerkte aendern sich — eine Beta-Verteilung mit alpha=500, beta=300 nach 800 Trades ist extrem "sicher" und passt sich kaum noch an. Optionen: (1) Discount-Faktor (alpha *= 0.99 pro Tag), (2) Windowed Counts (nur letzte N Trades). Fuer V1 ist der einfache Ansatz ohne Discounting akzeptabel — das Problem wird erst nach hunderten Trades relevant.
- **Stochastik im Decision-Zyklus:** Thompson Sampling ist per Design stochastisch — zwei aufeinanderfolgende Cycles koennen unterschiedliche Gewichte haben. Das ist gewollt (Exploration) aber macht Backtests nicht-deterministisch. Fuer deterministische Tests: Seed im Random-Generator fixieren.
- **Default: aus.** `adaptive-weights.enabled=false` — die Funktion wird erst aktiviert wenn ausreichend Counterfactual-Daten (ODIN-071) und Erfahrung mit statischen Gewichten (ODIN-074) vorliegen.
