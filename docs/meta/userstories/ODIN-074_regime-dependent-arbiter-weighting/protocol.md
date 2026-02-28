# Protokoll: ODIN-074 — Regime-Dependent Arbiter Weighting

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (17 RegimeWeightTest, 5 RegimeWeightProfileTest, 11 RegimeWeightFusionTest)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (6 RegimeWeightFusionIntegrationTest)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

1. **RegimeWeight als eigenes Record** statt verschachteltem Tuple in RegimeWeightProfile — bessere Validierung (sum=1.0), Wiederverwendbarkeit, saubere JavaDoc.

2. **RegimeWeightProfile erfordert alle 5 Regimes** im Konstruktor (fail-fast) statt graceful fallback fuer fehlende Regimes. Begruendung: ein fehlender Regime in der Konfiguration ist ein Deployment-Fehler, kein Runtime-Szenario. Bei zukuenftigem Enum-Ausbau (neuer Regime) faellt das sofort auf.

3. **fuseConfidence() berechnet gewichteten Durchschnitt** und speichert ihn als `fusedConfidence` in FusionResult. Dual-Key-Regel bleibt davon unberuehrt (binaere Key-Pruefung VOR der Confidence-Fusion).

4. **fuseSizeModifier() respektiert Konservativitaetsprinzip**: LLM-Reduktionen werden immer akzeptiert, unabhaengig von Gewichten. Begruendung: Positionsgroessen-Reduktion ist inhaerent risikoreduzierend. Regime-Weights fuer adaptive Sizing reserviert fuer ODIN-082 (Bandit).

5. **FusionResult.fusedConfidence = 0.0** bei NO_ACTION und QUANT_ONLY Exit-Pfaden (keine sinnvolle Confidence-Fusion ohne beide Seiten).

6. **NaN/Inf Guard** in RegimeWeight-Validierung UND in fuseConfidence(): doppelte Absicherung gegen korrupte Daten.

## Offene Punkte

Keine. Alle Akzeptanzkriterien erfuellt, alle Review-Findings behoben.

## ChatGPT-Sparring

### Durchgefuehrt am 2026-02-28

**Prompt**: Code + Tests zur Review vorgelegt, nach Edge Cases und Test-Luecken gefragt.

**Hoechst-priorisierte Findings (umgesetzt)**:

1. **NaN-Gewichte silently akzeptiert** — `Double.NaN < 0.0` ist false in Java, umging die Validierung. Fix: `!Double.isFinite()` Check als erste Validierung in RegimeWeight-Konstruktor. 4 neue Tests.

2. **Obere Grenze (>1.0) nicht erzwungen** — Wert wie 1.0005 konnte Validierung passieren wenn Summe in Toleranz war. Fix: Explizite `> 1.0` Checks. 2 neue Tests.

3. **Toleranzgrenzen nicht praezise getestet** — Alter Test verwendete exakte Dezimalwerte. Fix: Neue Tests fuer `sum=1.001` (Pass) und `sum=1.0011` (Fail). 2 neue Tests.

4. **fusedConfidence nie numerisch geprueft im Arbiter** — Tests prueften nur Gewichte, nicht die berechnete Confidence. Fix: Assertions fuer fusedConfidence=0.71 (TREND_UP) und fusedConfidence=0.82 (Integration) hinzugefuegt.

**Bewertete aber nicht umgesetzte Vorschlaege**:

- "Regime disagreement test" (Quant=TREND_UP, LLM=HIGH_VOL): Valider Punkt, aber out-of-scope fuer ODIN-074. Die Regime-Fusion-Logik (fuseRegime) ist pre-existing und nicht Teil dieser Story. Kann als separates Verbesserungsticket angelegt werden.
- "QUANT_ONLY with non-null LLM output": Der isQuantOnlyMode()-Check ist pre-existing (LLM wird ignoriert), kein neues ODIN-074-Verhalten. Nicht nachgetestet da pre-existing Logik.

## Gemini-Review

### Durchgefuehrt am 2026-02-28 (3 Dimensionen)

**DIM 1 — Code Review**:
- (+) Records und kompakte Konstruktoren gut genutzt
- (+) NaN/Inf Safety in fuseConfidence bestaetigt
- (Finding) `@DecimalMax("1.0")` fehlt auf WeightPair in BrainProperties — Spring-Validation wuerde einzelne Gewichte > 1.0 zulassen. Akzeptables Risiko: RegimeWeight-Konstruktor faengt das sowieso ab (Runtime-Validierung). Kein Spring-Boot-Startup-Fehler, sondern PipelineFactory-Fehler.

**DIM 2 — Konzepttreue**:
- **(KRITISCH) fuseConfidence() war implementiert aber nicht aufgerufen** — Methode existierte, wurde aber in buildEntryFusion() nicht verwendet. FusionResult hatte kein fusedConfidence-Feld. **Sofort behoben**: fusedConfidence als neues Feld, fuseConfidence() eingewired, Forensik-Payload erweitert, Tests mit numerischen Assertions.
- (+) Default-Gewichte exakt nach Story-Spec
- (+) QUANT_ONLY-Modus korrekt abgesichert

**DIM 3 — Praxis-Review**:
- **(BEHOBEN) JavaDoc/Implementation-Mismatch bei fuseSizeModifier()**: JavaDoc sagte "weight prevails" aber Code akzeptierte immer LLM-Reduktionen. JavaDoc korrigiert um Konservativitaetsprinzip widerzuspiegeln.
- (+) Dual-Key-Regel vollstaendig intakt
- (+) Forensische Traceability durch appliedQuantWeight/appliedLlmWeight/fusedConfidence in Event-Payload

## Test-Ergebnisse

| Test-Suite | Anzahl | Status |
|------------|--------|--------|
| RegimeWeightTest | 17 | PASS |
| RegimeWeightProfileTest | 5 | PASS |
| RegimeWeightFusionTest | 11 | PASS |
| RegimeWeightFusionIntegrationTest | 6 | PASS |
| odin-brain gesamt (Surefire) | 1330 | PASS |
| odin-core gesamt | 296 | PASS |
| odin-backtest gesamt | 369 | PASS |

## Modifizierte Dateien

### Neue Dateien
- `odin-brain/.../arbiter/RegimeWeight.java`
- `odin-brain/.../arbiter/RegimeWeightProfile.java`
- `odin-brain/.../arbiter/RegimeWeightTest.java`
- `odin-brain/.../arbiter/RegimeWeightProfileTest.java`
- `odin-brain/.../arbiter/RegimeWeightFusionTest.java`
- `odin-brain/.../arbiter/RegimeWeightFusionIntegrationTest.java`

### Modifizierte Dateien
- `odin-brain/.../arbiter/DecisionArbiter.java` — Regime-Weight-Fusion, fuseConfidence(), resolveRegimeWeights()
- `odin-brain/.../arbiter/FusionResult.java` — appliedQuantWeight, appliedLlmWeight, fusedConfidence Felder
- `odin-brain/.../config/BrainProperties.java` — ArbiterProperties mit RegimeWeightProperties
- `odin-brain/resources/odin-brain.properties` — 10 neue regime-weight Properties
- `odin-brain/.../TestBrainProperties.java` — arbiter Properties in createDefault()
- `odin-core/.../pipeline/PipelineFactory.java` — createRegimeWeightProfile()
- `odin-backtest/.../ParameterOverrideApplier.java` — arbiter Feld weitergeben
- 10+ Test-Dateien in odin-brain, odin-core, odin-backtest — BrainProperties-Konstruktor-Update
