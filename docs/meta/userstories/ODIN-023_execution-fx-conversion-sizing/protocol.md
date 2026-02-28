# Protokoll: ODIN-023 — FX-Conversion in Position Sizing

**Erstellt:** 2026-02-22 (QS-Agent, Runde 1 — erstmalig, da kein Implementierungsprotokoll vorhanden)

---

## Working State

- [x] Initiale Implementierung (durch Implementierungs-Agent, kein Protokoll erstellt)
- [x] Unit-Tests geschrieben (PositionSizerTest, RiskGateTest)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases (QS-Agent Runde 1)
- [x] Integrationstests geschrieben (RiskGatePositionSizerIntegrationTest)
- [x] Gemini-Review Dimension 1 (Code) — QS-Agent Runde 1
- [x] Gemini-Review Dimension 2 (Konzepttreue) — QS-Agent Runde 1
- [x] Gemini-Review Dimension 3 (Praxis) — QS-Agent Runde 1
- [x] Review-Findings eingearbeitet (Minor-Fixes durch QS-Agent)
- [ ] Commit & Push (nach QS-PASS)

---

## Design-Entscheidungen

### FX-Fallback-Strategie

- Entscheidung: Ein einziger konfigurierbarer Default `odin.execution.sizing.default-fx-rate-eur-usd=1.08`
- Begruendung: Story schreibt "statischer FX-Rate aus Config reicht" (V1). EUR-Aktien sollen FX=1.0 explizit uebergeben. Der konfigurierbare Default ist fuer den USD-Normalfall (1.08).
- Abweichung zum AC-Wortlaut: AC sagt "(1.0 fuer EUR-Aktien, 1.08 fuer USD-Aktien)" — dies ist als Erklaerung zu verstehen, welche Werte die Caller uebergeben sollen, NICHT als Anforderung an zwei separate Config-Keys.
- Langfristig (Phase 2): Instrumenten-Waehrungs-aware Fallback, wenn EUR-Aktien tatsaechlich 0.0 uebergeben.

### FX-Guard-Logik

- Implementiert: `Double.isFinite(fxRate) && fxRate > 0` in beiden PositionSizer und RiskGate
- Schutzt vor NaN und Infinity aus fehlerhafter AccountRiskState-Befuellung
- Auf Empfehlung von Gemini-Review (Dimension 1) und ChatGPT nachgeruesten

### LOG.warn bei FX-Fallback

- Auf Empfehlung Gemini-Review Dimension 3 nachgetragen
- RiskGate loggt WARN wenn live FX-Rate ungueltig ist und Fallback greift
- Wichtig fuer Observability in Prod

### Redundante FX-Fallback-Logik (PositionSizer + RiskGate)

- Beide Klassen enthalten die Fallback-Logik. Dies ist bewusst behalten als Defense-in-Depth.
- PositionSizer ist stateless POJO und wird theoretisch auch ohne RiskGate verwendet (testbar).
- RiskGate uebernimmt die Primaer-Aufloesung, PositionSizer hat den Guard als Fallback.

---

## Offene Punkte (nach Gemini Dimension 3)

1. **EUR-Aktien + FX=0.0 Szenario**: Wenn GlobalRiskManager fuer EUR-Aktien 0.0 uebergibt statt 1.0, wird 1.08 als Fallback genommen → Uebersizing um 8%. Wird jetzt durch LOG.warn sichtbar. Korrekte Loesung erfordert instrumentenspezifische FX-Konfiguration (Phase 2, Out of Scope fuer V1).

2. **Stark schwankender EUR/USD waehrend der Sitzung**: Wenn FX-Rate von 1.05 auf 1.15 steigt, werden neue Positionen entsprechend groesser. Der Live-Rate-Mechanismus (AccountRiskState.fxRate) ist bereits vorhanden — der Aufrufer (GlobalRiskManager) muss den Rate aktuell halten. ODIN-023 macht keine Annahmen darueber.

3. **Stale FX-Rate**: Wenn GlobalRiskManager den Kurs nicht aktualisiert, wird derselbe Wert den ganzen Tag verwendet. LOG.warn gibt eine Benachrichtigung wenn 0.0 durchkommt. Langfristig: Staleness-Monitoring in GlobalRiskManager.

---

## ChatGPT-Sparring (Runde 1 + 2)

### Vorgeschlagene Testszenarien (ChatGPT)

| Szenario | Umsetzung | Entscheidung |
|----------|-----------|-------------|
| Risk-budget path isoliert testen (ohne Cap-Dominanz) | Test `appliesHighVolStressFactor` nutzt kleines remainingBudget=1000 → Risk-dominiert | Abgedeckt, kein neuer Test noetig |
| EUR-Aktien Fallback 1.0 vs USD 1.08 differenziert | Geht ueber V1-Scope | Verworfen, Open Point dokumentiert |
| Flooring auf 0 bei minimaler Position | Testfall `returnsZeroForTinyBudget` abgedeckt | Bestehendes Test genuegt |
| Low FX Rate (0.5) Test | Abgedeckt durch `positionValueCapScalesWithFxRate` implizit | Aufwand gering, aber nicht kritisch fuer V1 |
| Negative FX Rate bei RiskGate | NEU: `usesFallbackFxRateWhenLiveRateIsNegative()` hinzugefuegt | Umgesetzt |
| NaN FX Rate | Gemini und ChatGPT empfohlen | Double.isFinite() Guard implementiert; NaN-Test als HINT eingestuft |

### Verworfene Vorschlaege

- "Separation fallback logic" (FxRateResolver class): Over-Engineering fuer V1. Verworfen.
- Event-Payload-Rounding-Tests mit exakter Dezimalstelle: HINT-Level, keine Zeit-Investition noetig.

---

## Gemini-Review

### Dimension 1: Code-Bugs

| Finding | Bewertung | Massnahme |
|---------|-----------|----------|
| Small Position Zeroing durch slippageReductionFactor | Vorher existierendes Verhalten, nicht in ODIN-023 Scope | Verworfen (Out of Scope) |
| Transaction Costs bypassen Risk-Limits | Vorher existierendes Verhalten | Verworfen (Out of Scope) |
| Fehlende Infinity-Guard bei FX Rate | Berechtigt (IMPORTANT) | Double.isFinite() in RiskGate und PositionSizer implementiert |
| Direction-Blind R/R (Long mit Target unter Entry) | Vorher existierend, OMS-Upstream-Problem | Verworfen (Out of Scope fuer ODIN-023) |
| Hardcoded R/R Fallback Trap (DEFAULT_RR_MULTIPLIER=2.0) | Konfigurationsrisiko, aber vorher existierend | Verworfen (Out of Scope) |
| Redundante FX-Fallback-Logik | HINT | Dokumentiert, bewusst behalten als Defense-in-Depth |

### Dimension 2: Konzepttreue

- Schritt 1 (max_risk) + 1b (FX): KORREKT implementiert
- Schritt 2/2b (stop_dist + stress_factor): KORREKT
- Schritt 3 (shares = floor(...)): KORREKT
- Schritt 4 (Cap 50% Tageskapital × fxRate): KORREKT mit FX-Skalierung
- Schritt 5 (Confidence-Scaling): KORREKT (min(regimeConfidence, quantScore) × slippageReductionFactor)
- ENTRY_SIZED Event mit fxRate: KORREKT

### Dimension 3: Praxis-Gaps

Alle als Open Points dokumentiert (siehe oben). Keine kritischen Fehler in der Implementierung gefunden. Die EUR-Aktien-Fallback-Problematik ist V1-Scope-Grenze, durch LOG.warn abgesichert.
