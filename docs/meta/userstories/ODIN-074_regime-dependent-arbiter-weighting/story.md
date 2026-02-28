# ODIN-074: Regime-Dependent Arbiter Weighting

**Modul:** odin-brain, odin-api
**Phase:** 1
**Abhaengigkeiten:** ODIN-071 (Post-Trade Counterfactual — liefert die empirische Basis fuer die Gewichtskalibrierung)
**Geschaetzter Umfang:** M

---

## Kontext

Der DecisionArbiter behandelt Quant- und LLM-Votes in allen Regimes symmetrisch: "Konservativster Wert gewinnt." Das ist fuer den Start korrekt, laesst aber Potenzial liegen. In klaren Trends liefern deterministische KPIs (EMA-Kreuzung, ADX, Volume) zuverlaessigere Signale, waehrend der LLM bei unklaren oder news-getriebenen Situationen mehr Kontextverstaendnis einbringt. Regime-abhaengige Gewichte ermoeglichen eine differenziertere Fusion, ohne die Dual-Key-Regel (beide muessen zustimmen) aufzuweichen.

## Scope

**In Scope:**

- Neues Record `RegimeWeightProfile` in `de.its.odin.brain.arbiter` mit Quant/LLM-Gewichten pro Regime
- Konfiguration der Default-Gewichte via `BrainProperties` (Namespace `odin.brain.arbiter.regime-weights.*`)
- Anpassung der Confidence-Fusion in `DecisionArbiter`: gewichtete Confidence statt gleichwertigem Min/Max
- Anpassung der Parameter-Fusion: Gewichte beeinflussen SizeModifier-Auswahl bei Quant/LLM-Divergenz
- Logging der angewandten Gewichte im `FusionResult` fuer Forensik
- Unit-Tests fuer alle Regime-Gewichtskombinationen

**Out of Scope:**

- Adaptive/dynamische Gewichtsanpassung basierend auf Live-Performance (das ist ODIN-082 Bandit)
- Aenderung der Dual-Key-Regel (Entry erfordert weiterhin BEIDE Keys)
- Kalibrierung der Gewichte anhand realer Counterfactual-Daten (erfordert ODIN-071 Datengrundlage)
- Aenderungen am QuantVote- oder LlmVote-Record selbst

## Akzeptanzkriterien

- [ ] Neues Record `RegimeWeightProfile` existiert mit `double quantWeight()` und `double llmWeight()` pro `Regime` (5 Regimes)
- [ ] Gewichte summieren sich zu 1.0 — Validierung in der Properties-Klasse
- [ ] Default-Gewichte konfiguriert: TREND_UP (0.70/0.30), TREND_DOWN (0.70/0.30), RANGE_BOUND (0.50/0.50), HIGH_VOLATILITY (0.40/0.60), UNCERTAIN (0.60/0.40)
- [ ] `DecisionArbiter.fuseConfidence()` berechnet `fusedConfidence = quantWeight * quantConf + llmWeight * llmConf` statt bisherigem symmetrischen Vergleich
- [ ] Dual-Key-Regel bleibt unberuehrt: Entry erfordert weiterhin BEIDE VoteActions >= ALLOW_ENTRY
- [ ] `FusionResult` enthaelt neues Feld `appliedQuantWeight` und `appliedLlmWeight` fuer Forensik
- [ ] Bei QUANT_ONLY-Modus werden Gewichte ignoriert (quantWeight=1.0, llmWeight=0.0)
- [ ] Unit-Test verifiziert: TREND_UP mit Quant-Confidence=0.8 und LLM-Confidence=0.5 ergibt fusedConfidence = 0.7*0.8 + 0.3*0.5 = 0.71
- [ ] Unit-Test verifiziert: HIGH_VOLATILITY mit Quant-Confidence=0.5 und LLM-Confidence=0.9 ergibt fusedConfidence = 0.4*0.5 + 0.6*0.9 = 0.74
- [ ] Unit-Test verifiziert: Gewichte mit Summe != 1.0 in Properties werden bei Validierung rejected
- [ ] Integrationstest: Kompletter Decision-Zyklus durch DecisionArbiter mit Regime-Gewichten, QuantVote und LlmVote liefert korrekte FusionResult-Werte

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `DecisionArbiter` | `odin-brain/.../arbiter/DecisionArbiter.java` | Neue Methode `fuseConfidence(QuantVote, LlmVote, RegimeWeightProfile)`, Gewichte in Fusion-Logik einbeziehen |
| `FusionResult` | `odin-brain/.../arbiter/FusionResult.java` | Neue Felder `appliedQuantWeight`, `appliedLlmWeight` |
| `BrainProperties` | `odin-brain/.../config/BrainProperties.java` | Neues Nested Record `ArbiterProperties` mit `RegimeWeightProperties` |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `RegimeWeightProfile` | `de.its.odin.brain.arbiter` | Immutable Record: `Map<Regime, RegimeWeight>` mit Lookup-Methode |
| `RegimeWeight` | `de.its.odin.brain.arbiter` | Record: `double quantWeight, double llmWeight` mit Summe-1.0-Validierung |

### Konfiguration

```properties
# Namespace: odin.brain.arbiter.regime-weights
odin.brain.arbiter.regime-weights.trend-up.quant-weight=0.70
odin.brain.arbiter.regime-weights.trend-up.llm-weight=0.30
odin.brain.arbiter.regime-weights.trend-down.quant-weight=0.70
odin.brain.arbiter.regime-weights.trend-down.llm-weight=0.30
odin.brain.arbiter.regime-weights.range-bound.quant-weight=0.50
odin.brain.arbiter.regime-weights.range-bound.llm-weight=0.50
odin.brain.arbiter.regime-weights.high-volatility.quant-weight=0.40
odin.brain.arbiter.regime-weights.high-volatility.llm-weight=0.60
odin.brain.arbiter.regime-weights.uncertain.quant-weight=0.60
odin.brain.arbiter.regime-weights.uncertain.llm-weight=0.40
```

### Design-Pattern

- `RegimeWeightProfile` wird in `PipelineFactory` aus `BrainProperties` erzeugt und als Konstruktor-Parameter an `DecisionArbiter` uebergeben
- Gewichte beeinflussen nur die **Confidence-Fusion** und **Parameter-Auswahl bei Divergenz**, NICHT die Dual-Key Entry-Regel
- Das `fusedRegime` im FusionResult bleibt vom bisherigen Konservatismus-Prinzip bestimmt (Quant-Regime bei Divergenz)

## Konzept-Referenzen

- `theme-backlog.md` Thema 12: "Regime-abhaengige Arbiter-Gewichtung" — IST-Zustand, SOLL-Zustand, Gewichtstabelle
- `theme-backlog.md` Thema 8: "Post-Trade Counterfactual" — liefert die empirische Basis (Abhaengigkeit)
- `docs/backend/architecture/05-llm-integration.md` — LLM-Integration und Fusion-Protokoll
- `docs/concept/overall-research-hybrid-tradingsystem-20262802/` — Research-Hybrid Empfehlung 2 ("Prioritaet Hoch")

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln (brain haengt nur von api ab)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln (Records fuer DTOs, ENUM statt String, @ConfigurationProperties als Record)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.arbiter.regime-weights.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `RegimeWeightProfile`, `RegimeWeight`, und erweiterte `DecisionArbiter`-Fusion
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstests mit realen Klassen (DecisionArbiter + RegimeWeightProfile + QuantVote + LlmVote)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest: Kompletter Decision-Zyklus mit verschiedenen Regimes

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klasse(n), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Race Conditions, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (Vergleich mit theme-backlog.md Thema 12)
- [ ] Dimension 3: Praxis-Review (unbehandelte reale Szenarien)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis mit Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Die Default-Gewichte (0.70/0.30 fuer TREND_UP etc.) sind Startwerte aus der Research-Synthese. Sie muessen NICHT empirisch validiert sein — die empirische Kalibrierung erfolgt erst nach ODIN-071 (Counterfactual). Die Story erstellt die **Infrastruktur** fuer regime-abhaengige Gewichtung.
- `FusionResult` ist ein Record — das Hinzufuegen neuer Felder erfordert Anpassung aller `new FusionResult(...)` Aufrufe inkl. `FusionResult.noAction()`. Alle existierenden Tests muessen aktualisiert werden.
- Die Confidence-Fusion sollte den gewichteten Durchschnitt verwenden, ABER die Dual-Key Entry-Logik (`quantVote.vote() >= ALLOW_ENTRY && llmVote.vote() >= ALLOW_ENTRY`) darf NICHT durch Gewichte beeinflusst werden. Gewichte wirken erst NACH der binaeren Key-Pruefung.
- `PipelineFactory` muss `RegimeWeightProfile` erzeugen und an `DecisionArbiter` uebergeben (odin-core Aenderung).
