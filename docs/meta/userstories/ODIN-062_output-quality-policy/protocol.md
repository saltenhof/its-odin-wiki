# Protokoll: ODIN-062 -- Output Quality Policy

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [ ] Playwright E2E-Tests mit Screenshots
- [ ] Commit & Push

## Design-Entscheidungen

### 1. Package-Struktur
- `de.its.odin.brain.sr.config.OutputPolicyProperties` — Config-Record separiert von Policy-Logik
- `de.its.odin.brain.sr.policy.OutputQualityPolicy` — Policy als eigene Klasse, kein Teil von SrLevelEngine
- Begruendung: Sauberere Separation of Concerns; Policy ist wiederverwendbar und testbar ohne Engine

### 2. Base+Boost in SupportLevelConfidence als neuer Overload
- Neuer `compute(..., OutputPolicyProperties)` Overload statt Modifikation bestehender Methode
- Bestehende 4- und 8-parametrige `compute()` Methoden bleiben erhalten (Legacy-Pfad)
- Begruendung: Keine Regressions-Risiken bei bestehenden Tests; klare Feature-Flag-Semantik

### 3. SrProperties-Erweiterung (11. Parameter)
- `OutputPolicyProperties output` als 11. Record-Component in `SrProperties`
- Konsequenz: Alle Tests die `SrProperties` direkt konstruieren mussten aktualisiert werden (7 Dateien)
- Default-Werte in `odin-brain.properties` unter `odin.brain.sr.output.*`

### 4. Fallback-Semantik (nach ChatGPT-Sparring)
- Urspruenglich: Fallback = top-N by confidence (pre-select)
- Nach ChatGPT-Review geaendert: Fallback = widening zum gesamten floor-filtered Pool
- Begruendung: Verhindert Trend-Day-Bias (wo Fallback alle Resistance-Zonen haette selektiert)
- Spatial Allocation in Stage 2 uebernimmt dann die finale Auswahl mit Side-Balance

### 5. Deterministische Tie-Breaking (nach Gemini-Review)
- Alle confidence-descending Comparators verwenden jetzt `BY_CONFIDENCE_DESC` Konstante
- Secondary key: center ascending (deterministische Sortierung bei gleichen Confidence-Werten)
- Verhindert non-deterministische Ausgaben bei identischen Scores

### 6. `computeRecencyScore` Fix (Semantik-Korrektur)
- Problem: Zonen ohne Touches hatten `recencyScore = 1.0` weil `lastActivityTime()` = `createdAt` = NOW
- Fix: Wenn `totalValidatedTouchCount() == 0`, direkt `0.0` zurueckgeben
- Begruendung: Recency misst Touch-Aktivitaet, nicht Zonenalter

### 7. Cross-Field Validation in OutputPolicyProperties
- `@AssertTrue isMinZonesWithinCap()`: minZones <= maxZones
- `@AssertTrue isMinPerSideWithinCap()`: minPerSide <= maxZones
- Spatial Allocation claempt zusaetzlich: supportReserved + resistanceReserved <= totalSlots

### 8. maxZones-Koexistenz
- Alter `maxZones=12` in `ConfluenceProperties` kontrolliert Zone-Fusion-Limit (andere semantische Stage)
- Neuer `maxZones=8` in `OutputPolicyProperties` kontrolliert finalen Output-Limit
- Bewusste Beibehaltung beider um Fusion-Stage nicht zu brechen

## Offene Punkte
- Playwright E2E-Test: S/R-Level-Anzeige im Frontend soll nicht mehr als 8 Zonen zeigen
  (wird in separatem QS-Agent geprueft)

## ChatGPT-Sparring

### Wichtigstes Finding: Fallback undermines Spatial Allocation
- Problem: Top-N Fallback selektiert bei Trend-Day alle High-Confidence Resistance-Zonen
- Danach hat Spatial Allocation keinen Support mehr zum Reservieren
- Fix: Fallback widened den Kandidaten-Pool, laesst Spatial Allocation entscheiden
- ChatGPT-Empfehlung: "Use the fallback to relax the threshold, not to pre-decide the selection"

### Weitere Findings (alle umgesetzt):
- P0: NaN/Infinity Guards in filterByHybridThreshold und applySpatialAllocation
- P1: Cross-field Config-Validation (@AssertTrue)
- P1: Clamping von Reservations in applySpatialAllocation

### Verworfene Empfehlungen:
- "Side-aware fallback" (top-K mit pre-allocated sides): zu komplex, ChatGPT bestaetigte
  dass "widen to floor pool" einfacher und aequivalent ist

## Gemini-Review

### Dimension 1 — Bugs und Null-Safety
- Spatial allocation Math.round Bias bei ungeraden Konfigurationen: bekannt, akzeptabel, dokumentiert
- NaN/Infinity-Safety: korrekt implementiert mit `Double::isFinite` Guards
- computeRecencyScore Fix: korrekt (+1 von Gemini)
- `@AssertTrue` Validation erlaubt 2*minPerSide > maxZones (wird von Spatial Allocation geclaempt): akzeptabel

### Dimension 2 — Konzepttreue
- Alle drei Formeln korrekt implementiert: Base+Boost, Tightness-Blend, Hybrid Threshold (+1)
- Fallback Logic respektiert static floor (+1)
- Stage 3 (maxZones cap) ist nach Spatial Allocation technisch redundant aber als Safety-Net beibehalten

### Dimension 3 — Praxis-Edge-Cases
- Single-sided Input: korrekt (kein Phantom-Zone-Filling)
- Identische Confidences: Tie-Breaking durch sekundaeren Sort-Key (center asc) implementiert
- center >= currentPrice = Resistance: bewusste Design-Entscheidung (Preis testet Level von unten)
- Sparse Fallback (1 Zone, minZones=4): returns single valid zone (minZones = Ziel, nicht Pflicht)

## Playwright E2E-Tests
Offen — wird in separatem QS-Agent nach Commit geprueft.
