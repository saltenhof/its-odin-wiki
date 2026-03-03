# ODIN-103: Re-Entry-Mechanismus — Wiederaufstocken nach Teilverkaeufen in laufender Position

**GitHub Issue:** saltenhof/its-odin-backend#102
**Size:** L | **Module:** odin-brain, odin-core, odin-execution | **Epic:** Brain Pipeline
**Erstellt:** 2026-03-03

---

## Kontext

Im Backtest QS-IREN-20260223-1m-r3 wurde ein signifikantes P&L-Potenzial identifiziert: Nach 3 Teilverkaeufen (50% der Position liquidiert) trat um 18:00-18:39 UTC ein klassisches Pullback-Recovery-Setup auf (TREND_UP conf=0.84, RSI 58-66, Preis ueber VWAP). Die State-Machine konnte keinen Re-Entry durchfuehren, weil der POSITIONED-State ausschliesslich den Uebergang zu OBSERVING kennt — 94 REJECT-Events belegen dies. Geschaetzter entgangener Gewinn: ~$510. Alle drei LLMs (Claude, ChatGPT, Gemini) identifizieren Re-Entry als groessten P&L-Hebel. Die Story umfasst zusaetzlich eine Vereinfachung der Tranchenstruktur (User-Praeferenz: max 3-4 Tranchen statt 5).

## Modul

odin-brain, odin-core, odin-execution

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

L

---

## Scope

### In Scope

- **State-Machine-Erweiterung:** POSITIONED-State erhaelt Re-Entry-Pfad unter strengen Bedingungen (kein neuer State, sondern bedingte Aktion innerhalb POSITIONED)
- **Exposure-Controller:** Pruefung `currentShares <= 60% peakShares` als Kapazitaetsbedingung fuer Re-Entry
- **Regime-Filter fuer Re-Entry:** TREND_UP mit confidence >= 0.80, 2x hintereinander (Hysterese gegen Regime-Flipping)
- **Markt-Filter fuer Re-Entry:** RSI 55-70, Preis ueber VWAP oder EMA20 steigend
- **Risk-Budget-Cap:** `totalOpenRisk <= 1.0 * initialRisk` (Risk-Free-Pyramiding — nur mit bereits gesichertem Gewinn)
- **Cooldown-Mechanismus:** Mindestens 10-15 Minuten seit letztem Exit, konfigurierbar
- **Re-Entry-Limit:** Max. 1-2 Re-Entries pro Trade-Cycle, konfigurierbar
- **Re-Entry-Sizing:** Max 50% der bereits verkauften Shares, als einzelner Block. Eigener Stop (ATR-basiert) und eigene Target-Levels
- **Tranchenstruktur-Vereinfachung:** COMPACT_3T (25/25/50) als neue Standard-Option. FULL_5T bleibt verfuegbar aber nicht mehr Default. STANDARD_4T (20/20/20/40) als Alternative

### Out of Scope

- Micro-Pyramiding (mehr als 2 Re-Entries pro Cycle)
- Sub-Tranchen innerhalb eines Re-Entry-Adds (der Add ist ein einzelner Block)
- Aenderung der initialen Entry-Logik (OBSERVING -> POSITIONED bleibt unveraendert)
- Short-Selling / Reverse-Entry
- Multi-Ticker Re-Entry Koordination

## Akzeptanzkriterien

- [x] State-Machine erlaubt Re-Entry im POSITIONED-State wenn ALLE Bedingungen erfuellt sind: Exposure <= 60% Peak, TREND_UP conf >= 0.80 (2x konsekutiv), RSI 55-70, Preis > VWAP oder EMA20 steigend, totalOpenRisk <= 1.0 * initialRisk, Cooldown abgelaufen, Re-Entry-Limit nicht erreicht
- [x] Re-Entry wird NICHT ausgefuehrt wenn mindestens eine Bedingung verletzt ist — jede Bedingung einzeln getestet
- [x] Re-Entry-Sizing betraegt max 50% der bereits verkauften Shares, nie mehr als die Differenz zwischen peakShares und currentShares
- [x] Re-Entry-Add erhaelt eigenen ATR-basierten Stop und eigene Target-Levels (unabhaengig von der Restposition)
- [x] Cooldown-Timer verhindert Re-Entry innerhalb von 10-15 Minuten nach letztem Teilverkauf
- [x] Maximal 1-2 Re-Entries pro Trade-Cycle (konfigurierbar)
- [x] Neue Tranchenstruktur COMPACT_3T (25/25/50) ist verfuegbar und als Default konfigurierbar
- [x] STANDARD_4T (20/20/20/40) ist als Alternative konfigurierbar
- [x] FULL_5T (20/20/10/10/40) bleibt verfuegbar aber nicht mehr Default
- [x] Auf Range-/Mean-Reversion-Tagen (kein persistenter TREND_UP) feuert Re-Entry NICHT
- [x] Alle Bedingungen, Schwellenwerte und Limits sind per `@ConfigurationProperties` konfigurierbar (keine hardcodierten Werte)
- [ ] Backtest IREN 2026-02-23 mit Re-Entry ON zeigt den erwarteten Re-Entry um ~18:00-18:39 UTC
- [ ] Backtest IREN 2026-02-23 mit Re-Entry OFF zeigt identisches Verhalten zum aktuellen Stand (kein Regressionsbruch)
- [x] Events: Re-Entry-Attempt und Re-Entry-Execution werden als Events geloggt (analog zu Entry/Exit-Events)

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `ReEntryCondition` | odin-brain | `de.its.odin.brain.rules` | Record: aggregiert alle Re-Entry-Bedingungen (exposure, regime, rsi, vwap, riskBudget, cooldown, reEntryCount). Methode `boolean isSatisfied()` |
| `ReEntryEvaluator` | odin-brain | `de.its.odin.brain.rules` | Evaluiert Re-Entry-Bedingungen gegen aktuellen MarketContext. Gibt `Optional<ReEntrySignal>` zurueck |
| `ReEntrySignal` | odin-brain | `de.its.odin.brain.rules` | Record: shares, stopPrice, targetLevels, reason |
| `ReEntryConfig` | odin-brain | `de.its.odin.brain.config` | `@ConfigurationProperties(prefix = "odin.brain.reentry")` Record mit allen Schwellenwerten |
| `ExposureController` | odin-core | `de.its.odin.core.pipeline` | Trackt peakShares, currentShares, verkaufte Shares. Methode `boolean hasCapacity(double threshold)` |
| `TranchenProfile` | odin-execution | `de.its.odin.execution.oms` | ENUM: COMPACT_3T(25,25,50), STANDARD_4T(20,20,20,40), FULL_5T(20,20,10,10,40). Ersetzt/ergaenzt bestehende Tranchen-Logik |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `PipelineStateMachine` | POSITIONED-State erhaelt bedingten Re-Entry-Pfad. Neue Methode `evaluateReEntry(MarketContext)` die den ReEntryEvaluator aufruft |
| `EntryRules` | Neue Methode oder Erweiterung fuer Re-Entry-spezifische Entry-Evaluation (getrennt von Initial-Entry) |
| `DecisionArbiter` | Muss Re-Entry-Signale erkennen und weiterleiten (analog zu Initial-Entry-Signalen, aber mit eigenem Pfad) |
| `TranchenCalculator` | Integration von `TranchenProfile`. Re-Entry-Sizing-Logik: `min(50% * soldShares, peakShares - currentShares)` |
| `OrderManagementService` | Re-Entry-Orders entgegennehmen und ausfuehren. Re-Entry-Add als eigener Order-Block mit eigenem Stop |

### Konfiguration

```properties
# Re-Entry conditions
odin.brain.reentry.enabled=false
odin.brain.reentry.min-exposure-ratio=0.60
odin.brain.reentry.min-regime-confidence=0.80
odin.brain.reentry.regime-confirmation-count=2
odin.brain.reentry.rsi-min=55.0
odin.brain.reentry.rsi-max=70.0
odin.brain.reentry.require-above-vwap=true
odin.brain.reentry.max-risk-ratio=1.0
odin.brain.reentry.cooldown-minutes=12
odin.brain.reentry.max-per-cycle=2
odin.brain.reentry.sizing-max-ratio=0.50

# Tranchen profile
odin.execution.oms.tranchen-profile=COMPACT_3T
```

## Konzept-Referenzen

- Backtest-Analyse QS-IREN-20260223-1m-r3: identifiziert Re-Entry als groessten P&L-Hebel, entgangener Gewinn ~$510 bei IREN am 2026-02-23
- State-Machine: `odin-core/src/main/java/de/its/odin/core/pipeline/PipelineStateMachine.java` — aktuelle States OBSERVING/POSITIONED und Uebergaenge
- Entry-Rules: `odin-brain/src/main/java/de/its/odin/brain/rules/EntryRules.java` — bestehende Entry-Evaluation als Vorlage fuer Re-Entry-Conditions
- Decision-Arbiter: `odin-brain/src/main/java/de/its/odin/brain/arbiter/DecisionArbiter.java` — Signal-Routing und Entscheidungslogik
- TranchenCalculator: `odin-execution/src/main/java/de/its/odin/execution/oms/TranchenCalculator.java` — bestehende Tranchen-Berechnung
- OrderManagementService: `odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java` — Order-Execution-Pfad

## Guardrail-Referenzen

- `CLAUDE.md` — Coding-Regeln (R1-R13): kein `var`, explizite Typen, Records fuer DTOs, ENUM fuer endliche Mengen, `@ConfigurationProperties` als Record + `@Validated`
- `CLAUDE.md` — Architecture: gegen Interfaces in `de.its.odin.api.port` programmieren, Events sind immutable, MarketClock verwenden (kein `Instant.now()`)
- `CLAUDE.md` — Tests: `*Test` (Surefire), `*IntegrationTest` (Failsafe), neue Geschaeftslogik -> Unit-Test PFLICHT
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln zwischen Modulen
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht

## Notizen fuer den Implementierer

1. **Re-Entry ist NICHT ein zweiter Entry.** Es ist eine Aufstockung einer bestehenden Position. Die State-Machine bleibt in POSITIONED — es gibt keinen Uebergang zu OBSERVING und zurueck. Der Re-Entry-Add wird als Sub-Position innerhalb des bestehenden Trade-Cycles behandelt.

2. **Risk-Free-Pyramiding ist das Kernprinzip:** Re-Entry darf nur stattfinden, wenn bereits genuegend Gewinn gesichert wurde (`totalOpenRisk <= initialRisk`). Das bedeutet: die Teilverkaeufe haben den Break-Even-Punkt der Restposition so weit gesenkt, dass der Re-Entry-Add das Gesamtrisiko nicht ueber das initiale Risiko hinaus erhoeht.

3. **Hysterese gegen Regime-Flipping:** Nicht auf einen einzelnen TREND_UP-Tick reagieren. Zwei konsekutive Bars mit TREND_UP conf >= 0.80 sind Pflicht. Dies verhindert Fehlsignale in choppy Phasen.

4. **Tranchenstruktur:** Die User-Praeferenz ist klar: wenige, bedeutsame Tranchen. Der User findet 5 Tranchen zu granular. COMPACT_3T (25/25/50) ist der bevorzugte Default. Die letzte Tranche (50%) als groesster Block wird erst am Ende liquidiert — das maximiert den Gewinn bei starken Trends.

5. **Cooldown-Timer:** Verwendet MarketClock, NICHT Instant.now(). Der Timer startet beim letzten Exit-Event und wird bei jedem Bar-Update geprueft.

6. **Re-Entry-Add bekommt eigene Stops/Targets:** Der Add wird wie eine eigenstaendige Mini-Position verwaltet — eigener ATR-basierter Stop, eigene Targets. Bei Stop-Hit des Adds wird nur der Add liquidiert, nicht die Restposition.

7. **Default: Re-Entry DISABLED.** `odin.brain.reentry.enabled=false` als Default. Feature muss explizit aktiviert werden. So bleibt der bestehende Betrieb unveraendert bis der User das Feature nach Validierung freischaltet.

8. **Backtest-Validierung ist kritisch:** Nicht nur den IREN-Tag testen. Multi-Day-Backtest (20+ Tage) fuer Expectancy, Max Drawdown, Sharpe. Besonders auf Range-/Mean-Reversion-Tagen pruefen, dass Re-Entry NICHT feuert (weil kein persistenter TREND_UP vorliegt).
