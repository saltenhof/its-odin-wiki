# ODIN-062: Output Quality Policy

**Modul:** odin-brain (Scoring + Policy) + odin-app (Policy-Anwendung im DTO-Mapping)
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-058 (Spatial Touch Registry), ODIN-059 (Dynamic Scoring -- fuer kalibrierte Confidence-Werte)

---

## Kontext

Die S/R-Engine liefert aktuell bis zu 12 Zonen mit niedriger Confidence (max 0.565 bei IREN 23.02). Zu viele Low-Value-Zonen ueberfluten den Chart und verwirren sowohl den LLM-Analyst als auch den menschlichen Trader. Die Output Quality Policy filtert, kalibriert und begrenzt die Zonenausgabe auf maximal 8 hochwertige, actionable Zonen mit garantierter Support/Resistance-Balance.

Drei Bausteine wirken zusammen: (1) Hybrid-Confidence-Threshold eliminiert Rauschen und marginale Zonen, (2) maxZones=8 mit Spatial Allocation garantiert Support/Resistance-Balance, (3) Confidence-Kalibrierung per Base+Boost und Tightness-Blend hebt starke Level an statt sie in einem gewichteten Durchschnitt zu nivellieren.

## Scope

**In Scope:**
- Neue Klasse `OutputQualityPolicy` in `de.its.odin.brain.sr` (oder `.sr.policy`)
- Confidence-Kalibrierung: Base+Boost-Formel in der bestehenden Scoring-Logik (ersetzt linearen gewichteten Durchschnitt)
- Tightness-Blend: Floor-basierte Daempfung statt multiplikativer Tightness
- Hybrid-Threshold: statischer Floor + dynamischer Cutoff relativ zu Cmax
- Fallback-Mechanismus: Threshold-Relaxierung bis minZones erreicht
- Spatial Allocation: Support/Resistance-Balance mit minPerSide-Reservierung
- maxZones=8 Cap
- Konfiguration als `OutputPolicyProperties` Record mit `@ConfigurationProperties`
- Pipeline-Integration: nach NMS (`suppressOverdenseZones`), vor Final Output
- Playwright-Test: Screenshot-Beweis dass maximal 8 Levels sichtbar sind, keine blassen Levels, mindestens 2 Support + 2 Resistance

**Out of Scope:**
- Aenderungen an der NMS-Logik (`suppressOverdenseZones`)
- Aenderungen an der Touch-Registry oder Touch-Erkennung
- Aenderungen am Zone-Typ (LEVEL/BAND -- das ist ODIN-063)
- Aenderungen am REST-Endpoint oder SSE-Events (keine neuen Felder)
- Aenderungen am Frontend-Rendering (Farben, Linienstaerke -- das ist implizit durch hoehere Confidence sichtbar)

## Akzeptanzkriterien

### Baustein 1: Hybrid Confidence Threshold

- [ ] Statischer Floor: Zonen mit `confidence < minConfFloor` (Default 0.30) werden nie ausgegeben
- [ ] Dynamischer Cutoff: `minConf = max(minConfFloor, dynamicThresholdFactor * Cmax)` mit `dynamicThresholdFactor` Default 0.60
- [ ] `Cmax` ist die hoechste Confidence aller Zonen NACH Kalibrierung (Baustein 3), VOR Threshold-Anwendung
- [ ] Fallback: Wenn nach Threshold weniger als `minZones` (Default 4) Zonen uebrig bleiben, wird der Threshold schrittweise relaxiert bis `minZones` erreicht sind -- aber nie unter `minConfFloor`
- [ ] Threshold-Relaxierung: Absteigend nach Confidence sortieren, die Top-`minZones` Zonen durchlassen wenn ihr Confidence >= minConfFloor

### Baustein 2: maxZones=8 + Spatial Allocation

- [ ] `maxZones` Default 8 (vorher 12)
- [ ] Spatial Allocation: Zonen werden in "Above Price" (Resistance) und "Below Price" (Support) aufgeteilt
- [ ] Mindestens `minPerSide` (Default 2) Slots pro Seite reserviert
- [ ] Restliche Slots werden nach Confidence aufgefuellt (hoechste Confidence zuerst)
- [ ] Wenn eine Seite weniger als `minPerSide` Zonen hat, werden die uebrigen Slots der anderen Seite zugeschlagen
- [ ] Nie mehr als `maxZones` Zonen in der Ausgabe

### Baustein 3: Confidence-Kalibrierung

- [ ] Base+Boost statt gewichteter Durchschnitt: `confidence = structural + (1.0 - structural) * behavioral * boostWeight`
- [ ] `boostWeight` Default 0.80
- [ ] Tightness als Blend: `behavioral = behavioralCore * (tightnessFloor + (1.0 - tightnessFloor) * tightness)` mit `tightnessFloor` Default 0.65
- [ ] Beide Transformationen sind monoton: steigt structural -> steigt confidence; steigt behavioral -> steigt confidence
- [ ] Ranking der Zonen bleibt durch Monotonie erhalten -- keine Invertierungen

### Pipeline-Integration

- [ ] `applyOutputPolicy()` wird aufgerufen NACH `suppressOverdenseZones()` (NMS) und `compressConsolidationZones()`, VOR Final Output
- [ ] Reihenfolge innerhalb `applyOutputPolicy()`: (1) Confidence-Kalibrierung ist bereits im Scoring passiert -> (2) minConf-Threshold anwenden -> (3) Spatial Allocation -> (4) maxZones Cap
- [ ] Die Methode gibt eine neue, gefilterte Liste von Zonen zurueck (keine Mutation der Eingabe)

### Konfiguration

- [ ] `OutputPolicyProperties` als Record mit `@ConfigurationProperties` und `@Validated`
- [ ] Namespace: `odin.brain.sr.output`
- [ ] Alle 7 Parameter konfigurierbar: `minConfFloor`, `dynamicThresholdFactor`, `maxZones`, `minZones`, `minPerSide`, `boostWeight`, `tightnessFloor`
- [ ] Defaults in `application.properties` (nicht in Java)

### UI-Verifikation (Playwright)

- [ ] Screenshot des Charts mit S/R-Levels nach Output Quality Policy
- [ ] Verifikation: maximal 8 Levels sichtbar im Chart
- [ ] Verifikation: keine blassen/kaum sichtbaren Levels mehr (alle ueber Threshold)
- [ ] Verifikation: mindestens 2 Support UND 2 Resistance Levels vorhanden

## Technische Details

**Neue Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/policy/OutputQualityPolicy.java` -- Hauptklasse mit `apply(List<SrZone>, double currentPrice)`
- `odin-brain/src/main/java/de/its/odin/brain/sr/config/OutputPolicyProperties.java` -- Konfiguration als Record

**Aenderungen an bestehenden Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/SrLevelEngine.java` (oder aehnlich) -- Aufruf von `applyOutputPolicy()` in der Pipeline
- Scoring-Logik (wo `confidence` berechnet wird) -- Base+Boost-Formel statt linearem Durchschnitt
- Scoring-Logik (wo `tightness` verrechnet wird) -- Tightness-Blend statt Multiplikator
- `odin-brain/src/main/resources/application.properties` -- Defaults fuer `odin.brain.sr.output.*`

**Konfigurationsrecord:**
```java
@ConfigurationProperties(prefix = "odin.brain.sr.output")
@Validated
public record OutputPolicyProperties(
    double minConfFloor,            // Default: 0.30
    double dynamicThresholdFactor,  // Default: 0.60
    int maxZones,                   // Default: 8
    int minZones,                   // Default: 4
    int minPerSide,                 // Default: 2
    double boostWeight,             // Default: 0.80
    double tightnessFloor           // Default: 0.65
) {}
```

**Base+Boost-Formel (in Scoring-Logik):**
```java
double confidence = structural + (1.0 - structural) * behavioral * boostWeight;
```

**Tightness-Blend (in Scoring-Logik):**
```java
double behavioral = behavioralCore * (tightnessFloor + (1.0 - tightnessFloor) * tightness);
```

**OutputQualityPolicy.apply() Pseudo-Logik:**
```java
public List<SrZone> apply(List<SrZone> zones, double currentPrice) {
    // 1. Threshold berechnen
    double cmax = zones.stream().mapToDouble(SrZone::confidence).max().orElse(0.0);
    double dynamicThreshold = Math.max(minConfFloor, dynamicThresholdFactor * cmax);

    // 2. Threshold anwenden mit Fallback
    List<SrZone> filtered = zones.stream()
        .filter(z -> z.confidence() >= dynamicThreshold)
        .sorted(comparing(SrZone::confidence).reversed())
        .toList();
    if (filtered.size() < minZones) {
        filtered = zones.stream()
            .filter(z -> z.confidence() >= minConfFloor)
            .sorted(comparing(SrZone::confidence).reversed())
            .limit(minZones)
            .toList();
    }

    // 3. Spatial Allocation
    List<SrZone> support = filtered.stream()
        .filter(z -> z.center() < currentPrice).toList();
    List<SrZone> resistance = filtered.stream()
        .filter(z -> z.center() >= currentPrice).toList();

    // Mindestens minPerSide pro Seite, Rest nach Confidence
    // ... (siehe Konzept 5 fuer Details)

    // 4. maxZones Cap
    // ... limit auf maxZones
}
```

**Pipeline-Einbindung (Reihenfolge):**
```
scoreZones()                    -> Confidence berechnet (mit Base+Boost + Tightness-Blend)
suppressOverdenseZones()        -> NMS
compressConsolidationZones()    -> Konsolidierung komprimiert (Konzept 2)
                                   |
applyOutputPolicy()             -> NEU:
    1. minConf-Threshold anwenden (hybrid: Floor + dynamisch)
    2. Spatial Allocation (Support/Resistance-Balance)
    3. maxZones=8 Cap
                                   |
Final Output
```

## Konzept-Referenzen

- `specs/sr-engine-v2/05-concept-output-quality.md` -- **Gesamtes Dokument** (primaere Referenz)
  - Baustein 1: Abschnitt "Baustein 1: Minimum-Confidence-Threshold (Hybrid: statisch + dynamisch)"
  - Baustein 2: Abschnitt "Baustein 2: maxZones auf 8 + raeumliche Verteilung" inkl. Spatial-Allocation-Pseudocode
  - Baustein 3: Abschnitt "Baustein 3: Confidence-Kalibrierung", Fix 3a (Base+Boost) + Fix 3b (Tightness-Blend)
  - Konfiguration: Abschnitt "Konfigurationsparameter" mit `OutputPolicyProperties` Record
  - Pipeline-Integration: Abschnitt "Pipeline-Integration" (Reihenfolge-Diagramm)
  - Testfaelle: Abschnitt "Tests" (8 Testfaelle)
  - Erwartetes Ergebnis: Abschnitt "Erwartetes Ergebnis fuer IREN 23.02" (12 -> 5-7 Zonen)
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` -- Abschnitt "Keine Sonder-Quota": ACTIVE OR-Level bekommen niedrige Base Confidence und werden natuerlich von Konzept 5 reguliert
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` -- Abschnitt "Pipeline-Flow", Schritt 10: `applyOutputPolicy()` kommt NACH Konsolidierungskompression
- `specs/sr-engine-v2/03-concept-lookahead-asymmetry.md` -- Abschnitt "Dynamic Quality Scoring": Die dynamisch berechnete Quality fliesst als `behavioral` in die Base+Boost-Formel ein
- `specs/sr-engine-v2/04-concept-reconciliation-drift.md` -- Abschnitt "Auswirkungen auf andere Komponenten": `SupportLevelConfidence.behavioral.touchCount` fragt Registry ab -- dieser touchCount-basierte behavioral Score wird durch Base+Boost kalibriert

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen (`de.its.odin.brain.sr.policy`), Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT", Risikoeinstufung, ChatGPT-Pool-Review
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13), Konfiguration (CSpec), Port-Abstraktion

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain` und `mvn compile -pl odin-app`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (alle Defaults ueber Properties)
- [ ] Records fuer DTOs und Konfiguration (`OutputPolicyProperties`)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.sr.output.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Zone mit confidence 0.25 -> unter Floor 0.30 -> nicht in Ausgabe
- [ ] Unit-Test: Dynamischer Threshold mit Cmax=0.80 -> Threshold=0.48 -> alles unter 0.48 entfernt
- [ ] Unit-Test: minZones-Fallback: Nur 3 Zonen ueber Threshold -> Threshold relaxiert bis 4 vorhanden
- [ ] Unit-Test: minZones-Fallback nie unter minConfFloor: Wenn < 4 Zonen ueber Floor -> nur die ueber Floor
- [ ] Unit-Test: Spatial Allocation: 6 Resistance, 2 Support -> mindestens 2 Support in Ausgabe
- [ ] Unit-Test: Spatial Allocation: 1 Support verfuegbar, minPerSide=2 -> 1 Support + 7 Resistance (kein Phantom-Support)
- [ ] Unit-Test: maxZones=8: 10 Zonen ueber Threshold -> nur Top-8 in Ausgabe
- [ ] Unit-Test: Base+Boost Monotonie: structural steigt -> confidence steigt (bei konstantem behavioral)
- [ ] Unit-Test: Base+Boost Monotonie: behavioral steigt -> confidence steigt (bei konstantem structural)
- [ ] Unit-Test: Base+Boost Beispielwerte aus Konzept: VWAP (structural=0.82, behavioral=0.55) -> confidence ~ 0.90
- [ ] Unit-Test: Tightness-Blend: tightness=0.3 -> behavioral wird um ~25% gedaempft (nicht 70%)
- [ ] Unit-Test: Tightness-Floor: tightness=1.0 -> behavioral unveraendert
- [ ] Unit-Test: Tightness-Floor: tightness=0.0 -> behavioral * tightnessFloor (nicht 0)
- [ ] Unit-Test: Leere Zonenliste -> leere Ausgabe (kein NPE)
- [ ] Unit-Test: Alle Zonen unter minConfFloor -> leere Ausgabe
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer aeussere Abhaengigkeiten (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: Komplette Pipeline-Strecke `scoreZones -> NMS -> applyOutputPolicy` mit realen Zonen-Daten
- [ ] Integrationstest: IREN-23.02-aehnliches Szenario: 12 Zonen rein, 5-7 Zonen raus (Plausibilitaetscheck)
- [ ] Integrationstest: Scoring mit Base+Boost + OutputPolicy zusammengeschaltet (kein Mock fuer Scoring)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: OutputQualityPolicy-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei identischen Confidences? Nur Support-Zonen vorhanden? Alle Zonen exakt auf Threshold? Cmax = 0? minPerSide > maxZones/2?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, Off-by-One bei Spatial Allocation, Sortier-Stabilitaet, Edge Cases bei Threshold-Relaxierung"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `specs/sr-engine-v2/05-concept-output-quality.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche: Base+Boost-Formel, Tightness-Blend-Formel, Threshold-Berechnung, Spatial-Allocation-Algorithmus, Fallback-Mechanismus, Konfigurationsparameter und Defaults"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Was passiert wenn alle Zonen auf derselben Seite (nur Support oder nur Resistance) liegen? Was wenn Cmax extrem niedrig ist (z.B. 0.32)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Reihenfolge der drei Schritte, Threshold-Relaxierungsstrategie, Behandlung von Edge Cases bei Spatial Allocation)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

### 2.9 Playwright E2E-Tests (PFLICHT)

- [ ] Screenshot des Charts mit S/R-Levels nach Output Quality Policy
- [ ] Screenshot-Beweis: maximal 8 Levels sichtbar
- [ ] Screenshot-Beweis: keine blassen/kaum sichtbaren Levels (alle ueber Threshold)
- [ ] Screenshot-Beweis: mindestens 2 Support UND 2 Resistance Levels vorhanden
- [ ] Screenshots im Verzeichnis `e2e/screenshots/` gespeichert
- [ ] Jeder Screenshot im `protocol.md` mit Beschreibung referenziert

## Notizen fuer den Implementierer

- **Baustein 3 (Kalibrierung) aendert den Scoring-Code, nicht die Policy.** Die Base+Boost-Formel und der Tightness-Blend greifen dort, wo die Confidence berechnet wird (vermutlich in `SupportLevelConfidence` oder aehnlich). Die `OutputQualityPolicy` arbeitet mit den bereits kalibrierten Werten.
- **Reihenfolge beachten:** Kalibrierung im Scoring -> NMS -> Konsolidierung -> OutputPolicy. Die Policy sieht nur die NMS-ueberlebenden, konsolidierten Zonen.
- **Spatial Allocation: currentPrice muss zur Verfuegung stehen.** Die Policy braucht den aktuellen Preis, um Support vs. Resistance zu unterscheiden. Das muss als Parameter durchgereicht werden.
- **minPerSide ist eine Reservierung, kein Minimum.** Wenn nur 1 Support-Zone existiert, kann minPerSide=2 keine zweite erfinden. Die Reservierung stellt nur sicher, dass vorhandene Support-Zonen nicht von Resistance-Zonen verdraengt werden.
- **Monotonie-Invariante der Kalibrierung ist kritisch.** Wenn Base+Boost oder Tightness-Blend das Ranking invertieren wuerden, koennten schwache Zonen staerkere verdraengen. Die mathematische Monotonie (bewiesen im Konzept) muss durch Unit-Tests abgesichert werden.
- **Fallback-Mechanismus genau implementieren:** Wenn nach dynamischem Threshold nur 3 Zonen uebrig sind, NICHT den Threshold auf minConfFloor setzen, sondern die naechstbeste Zone (4. in Confidence-Reihenfolge) dazunehmen -- sofern sie >= minConfFloor ist.
- **Testdaten aus dem Konzept verwenden:** Die IREN-23.02-Beispielwerte im Konzept (12 Zonen mit konkreten Confidences) sind ideale Testdaten fuer den Integrationstest.
