# Sparring-Bericht: Bounce-basierte Level-Erkennung

**Datum:** 2026-02-28
**Anlass:** Failure-Case IREN 2026-02-23 — $40.97-Support wird 4x getestet, nie durchbrochen, aber von der S/R-Engine nicht erkannt
**Sparring-Partner:** ChatGPT (3 Runden), Gemini (3 Runden)

---

## Ausgangssituation: Das Problem

Bei IREN 2026-02-23 gibt es ein offensichtliches Support-Level bei ~$40.97, das der Kurs VIERMAL getestet und nie durchbrochen hat:
- 17:15 CET: 1. Bounce
- 17:58 CET: 2. Bounce
- 18:37 CET: 3. Bounce
- 18:51 CET: 4. Bounce → danach explodiert IREN +4.4% bis Close

Die S/R-Engine gibt stattdessen $41.07 aus (SWING_CLUSTER). Die Distanz zwischen $40.97 und $41.07 beträgt $0.10 — kleiner als der NMS-suppressionRadius von $0.135. Damit wird $40.97 durch $41.07 unterdrückt.

**Parameter-Tuning löst das Problem nicht:** Änderungen an epsilon und NMS-Multiplier verschieben nur das Level-Set, das eigentliche Problem bleibt.

**Schlussfolgerung:** Das Problem ist strukturell, nicht parametrisch.

---

## Der fundamentale Zielkonflikt

Die Engine optimiert auf **"Cluster-Stärke"** (Pivot-Dichte im DBSCAN-Sinne):
> Wie viele Pivot-Punkte liegen nahe beieinander?

Ein starkes Support-Level braucht aber **"Bounce-Qualität"**:
> Wie oft hat der Preis dieses Niveau getestet und respektiert?

Beide Informationstypen sind orthogonal. DBSCAN misst den ersten; den zweiten gibt es nur im BounceDetector — der aber ein reiner Konsument ist und nicht in die Level-Erkennung zurückspeist.

---

## Geminis kritischer Durchbruch: Das Bootstrapping-Problem ist eine Illusion

Gemini stellte bei der Code-Analyse eine entscheidende Beobachtung fest:

> **Das $40.97-Level EXISTIERT in den `activeLevels`!**

NMS filtert nur die `zones`-Liste für den Output (`SrSnapshot`). Es löscht das Level **nicht** aus den `activeLevels` der Engine. Das bedeutet:

1. DBSCAN hat $40.97 erfolgreich geclustert
2. NMS unterdrückt es zugunsten von $41.07 im Output
3. Das Level bleibt aber intern aktiv und sammelt Touches in der TouchRegistry
4. Beim nächsten NMS-Durchlauf steht $41.07 aber weiterhin an erster Stelle, weil die NMS-Sortierung nach `confidence` (nicht nach `touchCount`) erfolgt

**Das eigentliche Problem:** Die NMS-Sortierung priorisiert die theoretische Cluster-Confidence vor der empirisch beobachteten Marktrealität (Touches).

---

## Diskutierte Lösungsansätze

### Ansatz A: Bounce-basierte Level-Erkennung als eigenständiger Pfad
- BounceDetector erkennt Rejections und erzeugt eigenständige "Bounce Level"
- **Problem:** Klassisches Hühner-Ei — BounceDetector braucht existierende Levels
- **Urteil:** Beide Partner lehnen ab — löst das Bootstrapping-Problem nicht

### Ansatz B: Bounce-Count beeinflusst NMS-Entscheidung (Confidence-Dominanz)
- Level mit mehr validierten Touches werden im NMS bevorzugt
- **ChatGPT:** Setzt Touch-Daten voraus, funktioniert nur wenn Level bereits aktiv ist
- **Gemini:** Wenn das Level intern aktiv ist und Touches sammelt, wäre eine TouchCount-erste Sortierung die direkte Lösung
- **Urteil:** Vielversprechend, setzt aber voraus dass das Level intern bereits existiert

### Ansatz C: Separate Source-Kategorie "TESTED_SUPPORT" / "PRICE_ACTION_LEVEL"
- Eigenständige Detection-Logik: "Preisniveau das N-mal in K-Bars respektiert wurde"
- **ChatGPT:** Favorit als Variante E (Price-Action-Heatmap/Buckets)
- **Gemini:** Warnt vor diskreten Schwellenwerten (Overfitting-Risiko bei fester N/K-Kombination)
- **Urteil:** Generalisierbar, aber erhöhter Implementierungsaufwand

### Ansatz D: Dynamische NMS (suppressionRadius sinkt mit TouchCount)
- Je mehr Touches, desto kleiner der effective suppressionRadius
- **ChatGPT:** Riskant — kann zu überdichten Level-Sets führen, schwer zu testen
- **Gemini:** Signaltheoretisch elegant, aber wenn beide Level sichtbar bleiben, ist Redundanz problematisch
- **Urteil:** Sekundäre Option, nicht primäre Lösung

### Ansatz E (ChatGPT): PriceActionTouchMap als eigenständiger Kandidaten-Generator
- Preis-Grid (Buckets) die direkt Bars auswerten
- Bucket akkumuliert touchCount, rejectCount mit exponentieller Recency-Gewichtung
- Promotion nach rejectScore >= 1.8 (ca. 2 frische Rejections)
- Neue Source-Kategorie PRICE_ACTION_LEVEL, die parallel zu DBSCAN in den Level-Pool eingeht
- **ChatGPT-Vorteil:** Löst Bootstrapping, unabhängig von existierenden Levels, kein Hühner-Ei
- **Gemini-Kritik:** Unnötige architektonische Komplexität wenn das Level bereits intern existiert; erzeugt konkurrierende "Wahrheiten" in der Pipeline

---

## Wo sich die Sparring-Partner einig sind

1. **DBSCAN ist nicht das Hauptproblem** — Parameter-Tuning hilft nicht, strukturelles Defizit
2. **Bounce-Qualität ist ein eigenständiger Informationstyp** der in der aktuellen Architektur zu wenig Gewicht hat
3. **TouchCount muss die NMS-Entscheidung beeinflussen** — ein empirisch bestätigtes Level muss ein theoretisches Level übertrumpfen können
4. **TouchRegistry darf nicht mit unvalidierten Pivot-Rohdaten verschmutzt werden** — Evidenztypen müssen getrennt bleiben
5. **Anti-Overfitting-Kernprinzip:** Jede Lösung muss auf "N-mal getestet und respektiert" als generisches Signal aufbauen, nicht auf IREN-spezifische Parameter

---

## Wo die Sparring-Partner widersprechen

| Aspekt | ChatGPT | Gemini |
|--------|---------|--------|
| Bootstrapping-Problem | Existiert — Level wird nie erzeugt wenn kein DBSCAN-Cluster | Illusion — Level existiert intern, NMS-Sortierung ist das Problem |
| Primäre Lösung | Neuer PriceActionLevelDetector (Ansatz E) | NMS-Sortierung umkehren (1 Codezeile) |
| Architekturansatz | Neue Datenstruktur (PivotHeatmap/Buckets) als separate Evidenzquelle | Vorhandene activeLevels-Infrastruktur nutzen |
| Code-Footprint | Mittel (neue Klasse, neue Source-Kategorie) | Minimal (eine Sortierregel geändert) |
| Vollständigkeit | Löst auch Fälle wo DBSCAN scheitert | Setzt voraus dass DBSCAN das Level intern erkennt |

---

## Tiefenanalyse: Wer hat recht?

**Geminis architektonische Diagnose ist präziser für den bekannten Failure-Case:** Die Diagnose "NMS-Suppression" impliziert, dass $40.97 als Cluster existiert. Wenn das stimmt, löst die NMS-Umkehrung das Problem minimal-invasiv und korrekt.

**ChatGPTs Ansatz E ist allgemeingültiger für unbekannte Failure-Cases:** Wenn ein Level so stark fragmentiert ist, dass DBSCAN es nicht erkennt (epsilon zu eng, Streuung zu groß), hilft die NMS-Sortierungs-Änderung nicht. Der PriceActionLevelDetector wäre dann der einzige Weg.

**Der entscheidende Unterschied:** IREN 23.02 ist dokumentiert als NMS-Suppression-Fall. Andere Fälle könnten DBSCAN-Failures sein. Eine robuste Lösung muss beide abdecken.

---

## Empfohlene Lösung: Zweistufige Strategie

Basierend auf der Synthese beider Sparring-Partner ergibt sich eine zweistufige Empfehlung:

### Stufe 1: Sofortiger Fix — NMS-Priorität korrigieren

**In `SrLevelEngine.java`, Methode `suppressOverdenseZones`:**

Aktuelle Sortierung:
```java
candidates.sort(Comparator.comparingDouble(SrZone::confidence).reversed()
        .thenComparingInt(SrZone::totalValidatedTouchCount).reversed()
        .thenComparingDouble(SrZone::center));
```

Geänderte Sortierung:
```java
candidates.sort(Comparator.comparingInt(SrZone::totalValidatedTouchCount).reversed()
        .thenComparingDouble(SrZone::confidence).reversed()
        .thenComparingDouble(SrZone::center));
```

**Wirkung:** Ein Level mit 2 validierten Touches überlebt gegen ein Level mit 0 Touches, auch wenn das Null-Touch-Level eine höhere Base-Confidence hat. Empirische Marktrealität schlägt theoretische Cluster-Priorität.

**Voraussetzung für Wirksamkeit:** Das Level muss intern als `SrLevel` in `activeLevels` vorhanden sein (auch wenn es im Output unterdrückt ist). Touches werden in der TouchRegistry auch für nicht-output-sichtbare Level gesammelt.

**Risiko:** Wenn TouchCount 0 ist für alle Kandidaten (früh am Tag, noch kein Touch), wird Confidence zum Tie-Breaker — keine Änderung zum heutigen Verhalten. Risikofrei.

**Anti-Overfitting:** Keine feste Schwelle. Das Prinzip "Touch schlägt Theorie" ist universal.

### Stufe 2: Mittelfristig — PriceActionTouchMap als zusätzliche Evidenzquelle

Für Fälle wo DBSCAN das Level nicht erkennt (Streuung > epsilon, minSamples-Versagen):

Eine separate, level-unabhängige **PivotHeatmap** (nicht die ValidatedTouchRegistry):
- Preis-Buckets die direkt Pivot-Koordinaten akkumulieren
- Separat von der ValidatedTouchRegistry (keine Evidenz-Vermischung)
- Peaks in der Heatmap werden als PRICE_ACTION_LEVEL-Kandidaten promoted
- Einspeisung in die normale Scoring/NMS/OutputPolicy-Pipeline

**Parameter (robust, ATR-skaliert):**
- `bucketSize = max(tickSize, min(price * 0.0005, 0.10 * ATR_5m))`
- Exponentieller Decay statt festes K-Fenster: `weight = exp(-ageBars / tau)` mit `tau = 45 Bars`
- Promotion-Schwelle: `rejectScore >= 1.8` (ca. 2 frische Rejections mit vollem Gewicht)
- Fusion vor NMS: wenn PRICE_ACTION_LEVEL und DBSCAN_CLUSTER innerhalb fusionRadius liegen → zu einem Composite-Level mergen, kein Dominanz-Konflikt

---

## Anti-Overfitting-Check

**Frage:** Wäre diese Lösung auf einem anderen Instrument oder einem anderen Tag genauso sinnvoll?

**Stufe 1 (NMS-Priorität):**
- Range-Day (SPY, seitwärts): Levels mit tatsächlichen Bounces werden bevorzugt — korrekt
- Trend-Day (starker Move nach oben): Support-Levels mit frühen Bounces bleiben sichtbar — korrekt
- News-Spike-Day (plötzlicher Move): Level-Set wird ohnehin durchgebrochen, neue Levels entstehen — Sortierungs-Änderung neutral
- Large-Cap mit breiter ATR: Touch-Count-Logik ist ATR-unabhängig — korrekt
- **Fazit: Vollständig generalisierbar. Kein IREN-spezifischer Parameter.**

**Stufe 2 (PriceActionTouchMap):**
- ATR-skalierte bucketSize und touchBand garantieren Volatilitätsinvarianz
- Exponentieller Decay statt festes Fenster macht es zeitraum-agnostisch
- Reject-Definition (Close wieder weg vom Level) ist universelles Bounce-Kriterium
- **Fazit: Generalisierbar, aber Parametrisierung muss breit getestet werden (kein Single-Day-Fit)**

---

## Risiken und Mitigationen

| Risiko | Mitigation |
|--------|------------|
| NMS-Umkehr bevorzugt spurious Touch-Noise | TouchRegistry-Quality-Gate bleibt aktiv (validatedTouchQualityMin); nur validierte Touches zählen |
| Level mit 1 Touch vs. Level mit 0 Touches: falsches Resultat früh am Tag | Kaum Auswirkung: 1 Touch > 0 Touches ist korrekt — wenn Level einmal getestet wurde, ist es relevanter |
| PriceActionTouchMap konkurriert mit DBSCAN | Fusion vor NMS löst Konflikt; kein Dominanzproblem |
| Exponentieller Decay macht alte Level "blind" | Korrekt — alte Level aus anderem Marktkontext sollen an Gewicht verlieren |
| PivotHeatmap wächst unbegrenzt | Nur relevanter Preisbereich gescannt (currentPrice ± scanAtrMult * ATR_5m) |

---

## Implementierungsreihenfolge

1. **Stufe 1 sofort:** NMS-Sortierung ändern (1 Zeile), Unit-Test schreiben, Backtest laufen lassen
2. **Validierung:** Prüfen ob $40.97 bei IREN 23.02 jetzt korrekt erkannt wird
3. **Weitere Regression-Tests:** Prüfen ob Level-Set auf anderen Backtests stabil bleibt (kein Overfitting-Artefakt)
4. **Stufe 2 später:** PriceActionTouchMap als separates ODIN-Issue planen, wenn Stufe 1 validiert

---

## Fazit

Das Kernproblem ist nicht DBSCAN-Epsilon oder NMS-Radius — es ist eine Priorisierungsphilosophie: Die Engine stellt theoretische Pivot-Mathematik über empirisch beobachtetes Marktverhalten.

**Die minimale, risikoarme und generalisierbare Lösung:** TouchCount als primäres NMS-Kriterium vor Confidence. Ein Level das der Markt einmal getestet hat, ist real. Ein Level das nur im Pivot-Clustering erscheint, aber nie berührt wurde, ist Theorie. Realität schlägt Theorie.

ChatGPTs Ansatz E (PriceActionTouchMap) ist der richtige mittelfristige Weg für Fälle wo DBSCAN komplett versagt — aber er ist kein Ersatz für die NMS-Priorisierungs-Korrektur, die der einfachere, sicherere erste Schritt ist.
