# ODIN-010 — Subregime-Resolver im Brain-Modul: Implementierungsprotokoll

**Story:** ODIN-010
**Datum:** 2026-02-22
**Status:** BEREIT FÜR QS (R2)
**Implementierer:** Claude Sub-Agent (R1) + Remediation-Agent (R2)

---

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring für Test-Edge-Cases (2 Runden)
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet (R2: NaN-Guards, requireNonNull, LLM-Threshold konfigurierbar)
- [ ] Commit & Push (folgt nach QS PASS)

---

## Zusammenfassung

User Story ODIN-010 erweitert den `RegimeResolver` im `odin-brain`-Modul um die Bestimmung
aller 20 Subregimes (4 pro Hauptregime). Der `ResolvedRegime`-Record wurde von 2 auf 3 Felder
erweitert, `BrainProperties.RegimeProperties` von 3 auf 17 Konfigurationsfelder (inkl. `llmRelevanceThreshold`).

**R2 (Remediation):** Alle QS-R1-Findings wurden behoben. 367 Unit-Tests grün (+2 Regression-Tests).

---

## Subregime-Mapping: Story-AC vs. Implementierung

Die Story-AC enthält konzeptuelle Platzhalter-Namen, die vor der finalen `Subregime`-Enum-Definition
(ODIN-001) erstellt wurden. Die Implementierung verwendet die in ODIN-001 final definierten Enum-Werte.

| Regime | Story-AC (konzeptuell) | Implementiert (`Subregime` Enum) | KPI-Kriterium |
|--------|----------------------|----------------------------------|---------------|
| TREND_UP | EARLY | EARLY | ADX < adxMatureThreshold |
| TREND_UP | MODERATE | MATURE | ADX > adxMatureThreshold |
| TREND_UP | LATE | LATE | ADX > adxLateThreshold + declining |
| TREND_UP | STRONG | EXHAUSTION | ADX > adxLateThreshold + declining + DI dominant + volume climax |
| TREND_DOWN | RELIEF_RALLY | RELIEF_RALLY | ADX < adxMatureThreshold |
| TREND_DOWN | PULLBACK | PULLBACK | ADX > adxMatureThreshold |
| TREND_DOWN | ACCELERATION | ACCELERATION | ADX > adxLateThreshold (not declining) |
| TREND_DOWN | CAPITULATION | CAPITULATION | ADX > adxLateThreshold + declining + DI dominant + volume climax |
| RANGE_BOUND | NARROW/MEAN | MEAN_REVERT | Default (narrow BB, normal vol) |
| RANGE_BOUND | WIDE | RANGE_EXPANSION | bbWidth/ATR >= expansionFactor |
| RANGE_BOUND | UPPER/LOWER | BREAKOUT_ATTEMPT | \|price-bbMiddle\|/halfWidth >= boundaryFactor + vol elevated |
| RANGE_BOUND | CHOP | CHOP | volumeRatio < chopVolumeRatioMax |
| HIGH_VOLATILITY | SPIKE | NEWS_SPIKE | atrDecayRatio > hvNewsSpikeAtrMultiplier |
| HIGH_VOLATILITY | DIRECTIONAL | MOMENTUM_EXPANSION | \|+DI - -DI\| >= hvDirectionalDiDifference |
| HIGH_VOLATILITY | LIQUID | LIQUID_VOL | volumeRatio >= hvElevatedVolumeRatioMin |
| HIGH_VOLATILITY | CHOP | AFTERSHOCK_CHOP | Default |
| UNCERTAIN | NaN | DATA_QUALITY | NaN in critical indicators (wenn enabled) |
| UNCERTAIN | LOW_VOL | LOW_LIQUIDITY | volumeRatio < uncertainLowVolumeRatio |
| UNCERTAIN | TRANSITION | TRANSITION | previousAdx > adxTrendThreshold |
| UNCERTAIN | MIXED | MIXED_SIGNALS | Default |

---

## Geänderte Dateien

### Produktionscode

| Datei | Änderungstyp | Beschreibung |
|-------|-------------|--------------|
| `odin-brain/.../rules/RegimeResolver.java` | R1: Vollständig neu; R2: NaN-Guards, requireNonNull | 5 Subregime-Methoden, `previousAdx` nur bei isFinite() updaten, volumeRatio NaN-Guard |
| `odin-brain/.../config/BrainProperties.java` | R2: Erweitert | `llmRelevanceThreshold` als konfigurierbares Feld (war hardcoded 0.5) |
| `odin-brain/.../resources/odin-brain.properties` | R2: Erweitert | `regime.llm-relevance-threshold=0.5` hinzugefügt |
| `odin-app/.../service/ParameterOverrideApplier.java` | R2: Bugfix | `applyRegimeOverrides` auf alle 17 Felder erweitert; `RulesProperties` + `ExecutionProperties` fehlende Argumente ergänzt (pre-existing bugs) |

### Testcode

| Datei | Änderungstyp | Beschreibung |
|-------|-------------|--------------|
| `TestBrainProperties.java` | R2: Erweitert | `llmRelevanceThreshold=0.5` in `RegimeProperties`-Konstruktor |
| `RegimeResolverTest.java` | R2: Erweitert | +2 Regression-Tests für NaN-volumeRatio und NaN-ADX-State-Korruption |
| `KpiEngineTest.java` | R2: Bugfix | `llmRelevanceThreshold=0.5` in `RegimeProperties`-Konstruktor |
| `PipelineFactoryTest.java` | R2: Bugfix | `RegimeProperties` auf 17 Argumente; `ExhaustionProperties` ergänzt |
| `ParameterOverrideApplierTest.java` | R2: Bugfix | `RegimeProperties` auf 17 Argumente; `ExhaustionProperties` ergänzt |
| `RegimeResolverIntegrationTest.java` | Neu (R1) | 3 Integrationstests (Consistency, Hysteresis, alle 20 reachable) |

---

## Designentscheidungen

### Subregime gegen bestätigtes Regime auflösen
Subregimes werden immer gegen das **bestätigte** Regime aufgelöst (nach `applyConfirmationLag`),
nicht gegen den Rohkandidaten. Dies stellt sicher, dass `subregime.parentRegime() == result.regime()`
immer gilt. Subregime-Wechsel innerhalb desselben Regimes sind sofort wirksam (keine Bestätigungs-Lag).

### UNCERTAIN TRANSITION: ADX-Proxy
Der TRANSITION-Subregime nutzt `previousAdx > adxTrendThreshold` als Proxy dafür, dass zuvor ein
Trend aktiv war. Eine vollständige Kopplung an confirmed-regime-changes würde State-Erweiterungen
erfordern, die über den Scope von ODIN-010 hinausgehen. Geplante Verbesserung für ODIN-011.

### RANGE_BOUND Boundary-Erkennung
`|currentPrice - bbMiddle| / bbHalfWidth >= boundaryFactor` erkennt, ob der aktuelle Preis im
äußeren Bereich der BB-Spanne liegt (nahe dem oberen oder unteren Band). Der Wert ist 0 bei
bbMiddle und ~1 an den Bändern.

### Epsilon-Guard für DI-Division
`Math.max(di, DI_EPSILON)` mit `DI_EPSILON = 1e-9` schützt vor Division durch 0, wenn ein DI-Wert
exakt 0 ist. Semantisch bedeutet DI=0 maximale Dominanz der Gegenrichtung.

### R2: previousAdx — nur finiten Wert speichern
`previousAdx` wird nur dann aktualisiert, wenn `Double.isFinite(currentAdx)` true ist. NaN-ADX
(Provider-Fehler, Sessionsübergänge) darf den State nicht korrumpieren, da `previousAdx` für
LATE/EXHAUSTION-Erkennung im Folgebar verwendet wird.

### R2: LLM_RELEVANCE_THRESHOLD als konfigurierbares Feld
Statt hardcoded `private static final double LLM_RELEVANCE_THRESHOLD = 0.5` ist der Schwellenwert
nun als `config.llmRelevanceThreshold()` konfigurierbar. Tuning ohne Recompilierung möglich.

### R2: volumeRatio NaN-Guard in resolveRangeBoundSubregime
`!Double.isFinite(volumeRatio)` (statt `Double.isNaN`) schützt auch vor Infinity-Werten
(z.B. extremer Volume-Spike / sehr kleiner gleitender Durchschnitt → Overflow). Bei nicht-finitem
volumeRatio früher CHOP-Fallback statt stiller MEAN_REVERT-Fehlklassifikation.

---

## ChatGPT Review — Zusammenfassung

**Runde 1 — Befunde:**
- CRITICAL: `volumeClimaxPresent = volumeRatio >= diDominanceRatio` war Bug (Kopplung zweier unabhängiger Parameter) → **behoben**: neues `trendVolumeClimaxRatioMin`-Feld
- CRITICAL: `resolveRangeBoundSubregime` nutzte `|bbMiddle - vwap|` statt `|currentPrice - bbMiddle|` → **behoben**: `currentPrice` wird durchgereicht
- WICHTIG: Magic Number `1.0` für volume elevated → **behoben**: `rangeElevatedVolumeRatioMin`, `hvElevatedVolumeRatioMin`
- WICHTIG: DI-Division durch 0 möglich → **behoben**: `DI_EPSILON = 1e-9`, `Math.max(di, DI_EPSILON)`
- WICHTIG: `TestBrainProperties` JavaDoc falsch (behauptete "All values match default") → **behoben**: auf "representative test defaults" korrigiert
- HINWEIS: Subregime-Namen in Story.md vs. Enum: dokumentiert (siehe Mapping-Tabelle oben)

**Runde 2 — Befunde:**
- CRITICAL: `@DecimalMin("0.0")` bei `adxTrendThreshold` erlaubt 0 als Divisor → **behoben**: auf `@DecimalMin("1.0")` geändert
- CRITICAL: Veraltetes JavaDoc in TREND_UP/TREND_DOWN (volume climax reference) → **behoben**: auf `trendVolumeClimaxRatioMin` aktualisiert
- CRITICAL: Epsilon-Literal `1e-9` inline als Magic Number → **behoben**: `private static final double DI_EPSILON = 1e-9`
- IMPORTANT: `bbHalfWidth > 0` doppelt geprüft (inline + guard) → **behoben**: Early-return bei `bbHalfWidth <= 0`, redundante Bedingung entfernt
- Bestätigt: RANGE_BOUND-Logik mit `currentPrice vs bbMiddle` ist semantisch korrekt
- Bestätigt: Epsilon-Guard für DI ist ausreichend

---

## Gemini-Review — Zusammenfassung (R2, 2026-02-22)

### Dimension 1: Code-Review (Bugs & Implementierungsfehler)

**Session-State-Leakage:** Gemini identifizierte zunächst das Risiko, dass `previousAdx` über Tagesgrenzen
hinweg korrumpiert werden könnte. Nach Klärung der ODIN-Architektur (jede Trading-Session instanziiert
einen frischen `RegimeResolver`) wurde dies als **kein Problem** eingestuft. Scope korrekt.

**NaN in DI-Werten:** `Math.max(NaN, 1e-9) = NaN` → `NaN >= 2.5 = false` — EXHAUSTION fällt konservativ
auf EARLY/RELIEF_RALLY zurück. Dieses implizite Verhalten ist **gewollt und korrekt**.

**Null-Safety:** `Objects.requireNonNull(indicators)` + `Objects.requireNonNull(config)` + `llmAnalysis`
null-safe via explizite Prüfung + `currentPrice` als primitiver `double` → **vollständig abgedeckt**.

**NaN-Guard in resolveRangeBoundSubregime:** Explizit bestätigt als **vollständig und exceptionally robust**:
deckt bbUpper/bbMiddle/bbLower/atr/volumeRatio ab, schützt vor Division durch 0, nutzt `!Double.isFinite`
statt nur `isNaN` für Infinity-Schutz.

**Gesamturteil Dimension 1: Keine P0-Bugs verbleibend. Implementation als production-ready eingestuft.**

### Dimension 2: Konzepttreue-Review

Gemini identifizierte die Abweichung der Enum-Namen zwischen story.md und Implementierung. Nach Klarstellung,
dass story.md konzeptuelle Platzhalter enthält und die Implementierung die ODIN-001-finalisierten Enum-Namen
verwendet (dokumentiert im protocol.md Mapping-Table), wurde dies als **kein Konzept-Fehler** eingestuft.

**Hysterese-Konzept:** Korrekt implementiert. Regime-Bestätigung auf Makro-Ebene, sofortige Subregime-
Wechsel auf Mikro-Ebene. Gemini bestätigt: "This is an excellent architectural choice."

**LLM-Konservatismus-Filter:** Ranking UNCERTAIN > TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > TREND_UP
ist für Long-Only korrekt. Gemini-Hinweis: Bei Short-Unterstützung (Out-of-Scope für V1) müsste die
Hierarchie dynamisch werden.

### Dimension 3: Praxis-Gaps

**Warmup-Periode:** NaN-Werte propagieren korrekt bis UNCERTAIN > DATA_QUALITY. Gemini empfahl expliziten
NaN-Check am Anfang von `resolveFromKpi`. Entscheidung: **Verworfen**. Das bestehende Kaskaden-Verhalten
ist korrekt und der explizite Check wäre redundant — DATA_QUALITY wird bereits über
`hasCriticalNanIndicators` in `resolveUncertainSubregime` abgedeckt.

**Priority-Masking:** Strenge Hierarchie ist intentional — entspricht dem Konzept-Design.

**Thin Markets/Stale Prices:** ODIN DataPipeline hat upstream DQ-Gates (Data Quality Gates), die veraltete
Quotes ablehnen bevor sie den RegimeResolver erreichen. Damit kein operatives Problem.

**Neues Finding aus Dimension 3 (dokumentiert als Offener Punkt):** Gemini bestätigte, dass
`resolveFromKpi` bei bestimmten Indikator-Konstellationen am Session-Anfang mehrere `false`-Evaluierungen
durch NaN-Vergleiche kaskadiert. Dies ist akzeptabel, da der Endeffekt korrekt ist (UNCERTAIN > DATA_QUALITY).
Ein expliziter früher NaN-Check als Qualitätsmerkmal für zukünftige Code-Reviews dokumentiert.

---

## Neue Konfigurationsfelder (odin-brain.properties)

| Property | Default | Beschreibung |
|----------|---------|--------------|
| `regime.llm-relevance-threshold` | 0.5 | Min. LLM-Confidence für Sekundärfilter (neu in R2) |
| `regime.adx-mature-threshold` | 25 | ADX für MATURE/ACCELERATION |
| `regime.adx-late-threshold` | 40 | ADX-Peak für LATE/CAPITULATION |
| `regime.di-dominance-ratio` | 2.5 | +DI/-DI Mindestverhältnis |
| `regime.trend-volume-climax-ratio-min` | 2.5 | volumeRatio für EXHAUSTION/CAPITULATION |
| `regime.range-bb-width-expansion-factor` | 2.0 | bbWidth/ATR für RANGE_EXPANSION |
| `regime.range-bb-boundary-factor` | 0.8 | \|price-bbMiddle\|/halfWidth für BREAKOUT_ATTEMPT |
| `regime.range-chop-volume-ratio-max` | 0.7 | Max volumeRatio für CHOP |
| `regime.range-elevated-volume-ratio-min` | 1.0 | Min volumeRatio für BREAKOUT_ATTEMPT |
| `regime.hv-news-spike-atr-multiplier` | 3.0 | atrDecayRatio für NEWS_SPIKE |
| `regime.hv-directional-di-difference` | 15.0 | \|+DI - -DI\| für MOMENTUM_EXPANSION |
| `regime.hv-elevated-volume-ratio-min` | 1.0 | Min volumeRatio für LIQUID_VOL |
| `regime.uncertain-low-volume-ratio` | 0.5 | Max volumeRatio für LOW_LIQUIDITY |
| `regime.uncertain-data-quality-nan-check` | true | NaN-Check für DATA_QUALITY aktivieren |

---

## Offene Punkte / Backlog

- **UNCERTAIN TRANSITION verbessern**: An confirmed-regime-change koppeln statt ADX-Proxy (ODIN-011)
- **Expliziter NaN-Early-Return in resolveFromKpi**: Als Qualitätsmerkmal für zukünftige Reviews vorgemerkt (Gemini Dimension 3 Finding) — aktuell kein P0
- **Boundary-Tests erweitern**: `adx == adxMatureThreshold`, `volumeRatio == trendVolumeClimaxRatioMin` usw. (ODIN-010 Erweiterung oder separater Tech-Task)
- **TestBrainProperties vollständig angleichen**: Kapazitäten (960/192 statt 390/78) evtl. angleichen oder separaten "fast-test" Modus einführen
