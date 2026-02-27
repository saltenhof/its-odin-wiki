# ODIN-063: Band API + Hybrid UI Rendering

**Modul:** odin-brain (ZoneType Enum) + odin-app (SrZoneDto Erweiterung) + odin-frontend (Chart-Rendering)
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-060 (Consolidation Bands -- liefert ConsolidationBand-Daten), ODIN-062 (Output Quality -- fuer kalibrierte Confidence)

---

## Kontext

Konsolidierungszonen (Konzept 2 / ODIN-060) werden aktuell als einzelne Linien dargestellt -- identisch zu scharfen S/R-Levels. Das ist semantisch falsch: Konsolidierungszonen sind Preisbereiche ("Chop-Zonen"), keine diskreten Preisbarrieren. Die UI muss den Unterschied visuell kommunizieren.

Diese Story fuehrt eine Hybrid-Rendering-Strategie ein: Scharfe S/R-Levels bekommen eine Linie mit dezentem Halo (Toleranzband), Konsolidierungszonen werden als halbtransparente gefuellte Rechtecke dargestellt. Dazu wird das Backend-API um ein `zoneType`-Feld erweitert (additiv, kein Breaking Change) und das Frontend implementiert das differentielle Rendering.

## Scope

**In Scope:**
- `ZoneType` Enum in odin-brain: `LEVEL`, `BAND`
- `SrZone` bekommt `zoneType()`-Methode (Default: `LEVEL`)
- ConsolidationBands werden mit `ZoneType.BAND` erstellt
- `SrZoneDto` Erweiterung um `String zoneType` (additives Feld)
- SSE-Events enthalten `zoneType`
- Frontend: LEVEL-Rendering (Solid Line + dezenter Halo)
- Frontend: BAND-Rendering (gefuelltes halbtransparentes Rechteck + gestrichelte Raender + POC-Linie + Label)
- Frontend: Farbschema (Support gruen, Resistance rot, Konsolidierung orange)
- Frontend: Graceful Degradation (unbekannter zoneType -> LEVEL-Rendering)
- TypeScript-Interface-Erweiterung (`SrZoneType`)
- Playwright E2E-Tests mit Screenshot-Beweisen fuer alle Rendering-Varianten

**Out of Scope:**
- Aenderungen an der Konsolidierungserkennung (das ist ODIN-060)
- Aenderungen an der Output Quality Policy (das ist ODIN-062)
- Aenderungen am REST-Endpoint-Pfad oder der HTTP-Methode
- Neue Endpoints
- LLM/Rules-Engine-Interpretation von BAND vs. LEVEL (spaetere Story)
- Mobile-Responsive Rendering

## Akzeptanzkriterien

### Backend: ZoneType Enum + SrZone-Erweiterung

- [ ] `ZoneType` Enum in `de.its.odin.brain.sr` mit Werten `LEVEL` und `BAND`
- [ ] `SrZone` bekommt Methode `ZoneType zoneType()` mit Default-Rueckgabe `LEVEL`
- [ ] Bestehende Zonen (OR, VWAP, Cluster) haben `zoneType = LEVEL`
- [ ] ConsolidationBands (aus ODIN-060) haben `zoneType = BAND`
- [ ] Bei BAND: `center` = POC, `band` = halbe Breite des Konsolidierungsbereichs

### Backend: SrZoneDto + API

- [ ] `SrZoneDto` erweitert um Feld `String zoneType` (`"LEVEL"` oder `"BAND"`)
- [ ] REST-Response `GET /api/v1/charts/{symbol}/sr-levels` enthaelt `zoneType` in jedem Zonen-Objekt
- [ ] SSE-Events mit S/R-Daten enthalten ebenfalls `zoneType`
- [ ] Aenderung ist additiv: bestehende Felder (center, band, confidence, touchCount, sources) unveraendert
- [ ] Kein Breaking Change fuer bestehende API-Consumer

### Frontend: LEVEL-Rendering (Linie + Halo)

- [ ] Horizontale Solid Line bei `center`-Preis ueber die gesamte Chart-Breite
- [ ] Farbe: Support `#0EB35B` (Bullish Green), Resistance `#FC3243` (Bearish Red), VWAP `#2196F3`
- [ ] Confidence-abhaengige Opacity der Linie (35-80%)
- [ ] Dezenter Halo: halbtransparentes Rechteck [center-band, center+band]
- [ ] Halo-Opacity: 8-12% (deutlich dezenter als die Linie selbst)
- [ ] Halo-Farbe: gleiche Farbe wie Linie
- [ ] Label: Preis + Source-Abkuerzung rechts am Chart-Rand

### Frontend: BAND-Rendering (Gefuellter Bereich)

- [ ] Gefuelltes halbtransparentes Rechteck [center-band, center+band]
- [ ] Farbe: `#FFA726` (Orange/Amber)
- [ ] Fuellungs-Opacity: 12-18%
- [ ] Gestrichelte Randlinie oben (center+band) mit Opacity 40%
- [ ] Gestrichelte Randlinie unten (center-band) mit Opacity 40%
- [ ] Gestrichelte POC-Linie bei `center` mit Opacity 60%
- [ ] Label: `"Consol. $X.XX--$Y.YY"` Format (Preisspanne von center-band bis center+band)
- [ ] Band ist visuell klar unterscheidbar von LEVEL-Rendering

### Frontend: Graceful Degradation

- [ ] Unbekannter `zoneType`-Wert (weder `"LEVEL"` noch `"BAND"`) wird als LEVEL gerendert
- [ ] Kein JavaScript-Fehler oder Crash bei unbekanntem zoneType
- [ ] Fehlender `zoneType` (undefined/null) wird als LEVEL gerendert

### Frontend: TypeScript

- [ ] `SrZoneType = 'LEVEL' | 'BAND'` als Union Type (kein enum-Keyword)
- [ ] `SrZone` Interface erweitert um `zoneType: SrZoneType`
- [ ] Strikte TypeScript-Checks bestehen

## Technische Details

**Backend -- Neue Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/sr/ZoneType.java` -- Enum mit `LEVEL`, `BAND`

**Backend -- Aenderungen an bestehenden Dateien:**
- `SrZone` (odin-brain) -- neue Methode `zoneType()`, Default `LEVEL`
- ConsolidationBand-Erstellung (odin-brain, aus ODIN-060) -- `zoneType = BAND`
- `SrZoneDto` (odin-app) -- neues Feld `String zoneType`
- DTO-Mapping (odin-app) -- `zone.zoneType().name()` in DTO uebernehmen
- SSE-Event-Builder (odin-app) -- `zoneType` in Event-Payload aufnehmen

**Backend -- DTO:**
```java
public record SrZoneDto(
    double center,          // Level-Preis (bei Band: POC)
    double band,            // Toleranzband (bei Band: halbe Breite)
    double confidence,
    int touchCount,
    String sources,         // "OR", "CLUSTER", "CLUSTER,VWAP", "CONSOLIDATION"
    String zoneType         // "LEVEL" oder "BAND"
) {}
```

**Backend -- Mapping:**
```java
new SrZoneDto(
    zone.center(),
    zone.band(),
    zone.confidence(),
    zone.touchCount(),
    zone.sourceLabel(),
    zone.zoneType().name()   // "LEVEL" oder "BAND"
)
```

**Frontend -- TypeScript-Interface:**
```typescript
type SrZoneType = 'LEVEL' | 'BAND';

interface SrZone {
  center: number;
  band: number;
  confidence: number;
  touchCount: number;
  sources: string;
  zoneType: SrZoneType;
}
```

**Frontend -- Farb-Konstanten:**
```typescript
const SR_COLORS = {
  SUPPORT: '#0EB35B',
  RESISTANCE: '#FC3243',
  VWAP: '#2196F3',
  CONSOLIDATION: '#FFA726',
} as const;

const HALO_OPACITY = 0.10;           // 8-12%
const BAND_FILL_OPACITY = 0.15;      // 12-18%
const BAND_EDGE_OPACITY = 0.40;
const BAND_POC_OPACITY = 0.60;
```

**Frontend -- Rendering-Logik (Pseudocode):**
```typescript
function renderSrZones(zones: SrZone[], currentPrice: number) {
  for (const zone of zones) {
    const side = zone.center < currentPrice ? 'support' : 'resistance';

    if (zone.zoneType === 'BAND') {
      // Konsolidierungsband
      renderFilledRect(zone.center - zone.band, zone.center + zone.band,
                       SR_COLORS.CONSOLIDATION, BAND_FILL_OPACITY);
      renderDashedLine(zone.center - zone.band, SR_COLORS.CONSOLIDATION, BAND_EDGE_OPACITY);
      renderDashedLine(zone.center + zone.band, SR_COLORS.CONSOLIDATION, BAND_EDGE_OPACITY);
      renderDashedLine(zone.center, SR_COLORS.CONSOLIDATION, BAND_POC_OPACITY);
      renderLabel(`Consol. ${formatPrice(zone.center - zone.band)}--${formatPrice(zone.center + zone.band)}`);
    } else {
      // Default: LEVEL (auch fuer unbekannte zoneTypes)
      const color = zone.sources.includes('VWAP') ? SR_COLORS.VWAP
                  : side === 'support' ? SR_COLORS.SUPPORT
                  : SR_COLORS.RESISTANCE;
      const opacity = mapConfidenceToOpacity(zone.confidence);
      renderSolidLine(zone.center, color, opacity);
      renderFilledRect(zone.center - zone.band, zone.center + zone.band,
                       color, HALO_OPACITY);
      renderLabel(`${formatPrice(zone.center)} ${zone.sources}`);
    }
  }
}
```

**REST-Response-Beispiel:**
```json
[
  {
    "center": 41.067,
    "band": 0.034,
    "confidence": 0.71,
    "touchCount": 12,
    "sources": "CLUSTER",
    "zoneType": "LEVEL"
  },
  {
    "center": 41.337,
    "band": 0.042,
    "confidence": 0.90,
    "touchCount": 9,
    "sources": "CLUSTER,VWAP",
    "zoneType": "LEVEL"
  },
  {
    "center": 42.04,
    "band": 0.225,
    "confidence": 0.65,
    "touchCount": 60,
    "sources": "CONSOLIDATION",
    "zoneType": "BAND"
  }
]
```

## Konzept-Referenzen

- `specs/sr-engine-v2/06-concept-band-api-and-ui.md` -- **Gesamtes Dokument** (primaere Referenz)
  - API-Aenderung: Abschnitt "API-Aenderung: SrZoneDto erweitern" mit aktuellem und neuem DTO, Response-Beispiel
  - LEVEL-Rendering: Abschnitt "LEVEL-Rendering (Linie + Halo)" mit Farben, Opacity, Label-Format
  - BAND-Rendering: Abschnitt "BAND-Rendering (Gefuellter Bereich)" mit Farben, Opacity, POC, Label-Format
  - Farb-Tabelle: Abschnitt "Farben (basierend auf User-Vorgaben)" -- vollstaendige Farb- und Opacity-Zuordnung
  - TypeScript: Abschnitt "TypeScript-Interface" mit `SrZoneType` und `SrZone`
  - Rendering-Logik: Abschnitt "Chart-Rendering-Logik (Pseudocode)"
  - Backend-Aenderungen: Abschnitt "Backend-Aenderungen" mit ZoneType Enum, SrZoneDto-Mapping, SSE-Events
  - Risiken: Abschnitt "Risiken und Mitigationen" -- Graceful Degradation, Breaking Change, Band-Breite
  - Tests: Abschnitt "Tests" -- 4 Backend-Tests + 5 Frontend-Tests
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` -- Abschnitt "Architektur: ConsolidationBand (neuer Zone-Typ)": Record-Definition mit bottom, top, poc, totalTouches. ConsolidationBands werden mit `type = CONSOLIDATION` erstellt
- `specs/sr-engine-v2/02-concept-overdense-consolidation.md` -- Abschnitt "Downstream-Signale": Frontend soll "Band als gefuelltes Rechteck rendern (nicht 4 einzelne Linien)"
- `specs/sr-engine-v2/01-concept-or-level-flooding.md` -- Abschnitt "Interaktion mit anderen Konzepten": "Konzept 6: OR-Level sind immer `zoneType=LEVEL` (nie BAND)"
- `specs/sr-engine-v2/05-concept-output-quality.md` -- Abschnitt "Pipeline-Integration": Output Quality kommt VOR dieser Story (Abhaengigkeit)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen (`de.its.odin.brain.sr`), Namensregeln, Abhaengigkeitsregeln (brain -> api, app -> core -> brain)
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT", Risikoeinstufung, ChatGPT-Pool-Review
- `docs/frontend/guardrails/frontend.md` -- Feature-basierte Ordnerstruktur, TypeScript strict, Union-Types statt enum, CSS Modules, Dark Theme
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13), Frontend-Architektur (SSE + REST), Chart-Farben (User-Vorgaben)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`, `mvn compile -pl odin-app`)
- [ ] Frontend baut fehlerfrei (`npm run build` in odin-frontend)
- [ ] Kein `var` -- explizite Typen (Java)
- [ ] TypeScript strict-Mode bestanden (Frontend)
- [ ] Keine Magic Numbers -- `private static final` Konstanten (Java), `as const` Objekte (TypeScript)
- [ ] Records fuer DTOs (`SrZoneDto`)
- [ ] ENUM statt String fuer endliche Mengen (`ZoneType` Enum in Java, Union Type in TypeScript)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc + TSDoc). Kommentare: Englisch
- [ ] CSS Modules fuer neue Styles (kein Inline-CSS)
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

**Backend:**
- [ ] Unit-Test: ConsolidationBand wird als SrZoneDto mit `zoneType="BAND"` serialisiert
- [ ] Unit-Test: Regulaeres Level wird als SrZoneDto mit `zoneType="LEVEL"` serialisiert
- [ ] Unit-Test: API-Response enthaelt gemischte LEVEL- und BAND-Zonen
- [ ] Unit-Test: BAND-center ist POC, BAND-band ist halbe Breite
- [ ] Unit-Test: ZoneType.name() liefert "LEVEL" bzw. "BAND"
- [ ] Unit-Test: Default-zoneType fuer SrZone ist LEVEL

**Frontend:**
- [ ] Unit-Test: LEVEL-Zone erzeugt Linie + Halo (DOM/Canvas Assertions)
- [ ] Unit-Test: BAND-Zone erzeugt gefuelltes Rechteck + Raender + POC
- [ ] Unit-Test: Unbekannter zoneType -> Fallback auf LEVEL-Rendering (kein Crash)
- [ ] Unit-Test: Konsolidierungsfarbe (`#FFA726`) unterscheidet sich von Support/Resistance
- [ ] Unit-Test: Label-Format fuer BAND: `"Consol. $X.XX--$Y.YY"`
- [ ] Unit-Test: Label-Format fuer LEVEL: `"$X.XX SOURCE"`
- [ ] Testklassen-Namenskonvention: `*Test` (Java), `*.test.ts` / `*.test.tsx` (TypeScript)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Backend-Integrationstest: REST-Endpoint liefert gemischte LEVEL- und BAND-Zonen im JSON
- [ ] Backend-Integrationstest: SSE-Event enthaelt `zoneType` im Payload
- [ ] Frontend-Integrationstest: Chart-Komponente rendert gemischte Zonen korrekt (LEVEL als Linie, BAND als Rechteck)
- [ ] Mindestens 1 Integrationstest pro Seite (Backend + Frontend) der die Hauptfunktionalitaet End-to-End abdeckt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Java Failsafe), `*.integration.test.ts` (TypeScript)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: ZoneType Enum, SrZoneDto, Frontend-Rendering-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei band=0 (kein Halo/kein Rechteck)? Sehr breite Bands (> 5% Preis)? Ueberlappende Bands? VWAP innerhalb eines Bands? Confidence=0?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Backend-Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Null-Safety, fehlendes zoneType-Mapping, inkonsistente Serialisierung"
- [ ] Frontend-Code an Gemini uebergeben mit Auftrag: "Pruefe auf Rendering-Bugs, fehlende Fallbacks, falsche Farb/Opacity-Werte, Canvas/SVG-Fehler"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `specs/sr-engine-v2/06-concept-band-api-and-ui.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche: DTO-Felder, ZoneType-Werte, Farben, Opacity-Werte, Label-Formate, Rendering-Logik (Linie vs. Rechteck), Graceful Degradation"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Accessibility (Farbenblindheit?), Performance bei vielen Bands, Touch/Hover-Interaktion auf Bands, Dark-Theme-Kontrast?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Canvas vs. SVG fuer Rendering, Opacity-Feintuning, Label-Positionierung)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

### 2.9 Playwright E2E-Tests mit Screenshot-Beweisen (PFLICHT, ZENTRAL)

**Diese Tests sind der KERN der Abnahme fuer diese Story. Kein "ich habe das implementiert" -- nur Screenshot-Evidenz zaehlt.**

- [ ] **Screenshot 1: BAND-Rendering** -- Konsolidierungsband ist als halbtransparentes oranges Rechteck im Chart sichtbar
- [ ] **Screenshot 2: LEVEL-Rendering** -- Regulaere S/R-Level haben scharfe Linie + dezenten Halo
- [ ] **Screenshot 3: Farben** -- Konsolidierungsband ist orange (`#FFA726`), Support gruen (`#0EB35B`), Resistance rot (`#FC3243`)
- [ ] **Screenshot 4: Band-Label** -- Label zeigt `"Consol. $X.XX--$Y.YY"` Format
- [ ] **Screenshot 5: POC** -- Gestrichelte Linie bei POC innerhalb des Bands sichtbar
- [ ] **Screenshot 6: Graceful Degradation** -- Unbekannter zoneType wird als LEVEL gerendert (kein Crash). Test: Backend-Mock mit `zoneType: "UNKNOWN"` -> kein Fehler, LEVEL-Rendering
- [ ] Screenshots im Verzeichnis `e2e/screenshots/` gespeichert
- [ ] JEDER Screenshot im `protocol.md` mit Dateiname + Beschreibung referenziert
- [ ] Screenshots werden mit `page.screenshot()` erstellt (echte Pixel-Evidenz)
- [ ] Test-Setup: Backtest mit bekannten Daten (z.B. IREN 23.02) die sowohl LEVEL als auch BAND-Zonen erzeugen

## Notizen fuer den Implementierer

- **Additives API-Feld:** `zoneType` ist ein neues Feld im DTO. Bestehende Consumer die das Feld nicht kennen, ignorieren es (standard JSON-Deserialisierung). Kein Breaking Change.
- **Frontend-Charting-Library pruefen:** Die Rendering-Implementierung haengt davon ab, ob lightweight-charts, TradingView, Canvas oder SVG verwendet wird. Die Story gibt die VISUELLE Spezifikation vor, die technische Umsetzung muss zur bestehenden Chart-Architektur passen.
- **Orange fuer Konsolidierung:** `#FFA726` signalisiert "Warnung/Neutral" -- weder bullish (gruen) noch bearish (rot). Das ist bewusst gewaehlt (siehe Konzept 6, Abschnitt "Farben").
- **Halo vs. Band Opacity:** Halo (8-12%) ist bewusst dezenter als Band-Fill (12-18%). Der Halo soll das Toleranzband nur andeuten, das Band soll als Zone erkennbar sein.
- **POC-Linie innerhalb des Bands:** Die gestrichelte Linie bei `center` (POC) gibt dem Trader einen Orientierungspunkt innerhalb der Chop-Zone. Opacity 60% macht sie sichtbar aber nicht dominant.
- **Label-Format:** BAND-Labels zeigen die Preisspanne (`Consol. $41.65--$42.10`), LEVEL-Labels zeigen den Einzelpreis + Source (`$41.34 VWAP`). Das ist im Konzept so spezifiziert.
- **Graceful Degradation ist kritisch:** Wenn das Backend in Zukunft neue ZoneTypes einfuehrt (z.B. `DYNAMIC`), darf das Frontend nicht crashen. Der Fallback auf LEVEL-Rendering ist die sichere Default-Strategie.
- **VWAP-Farbe:** VWAP-Levels haben eine eigene Farbe (`#2196F3`) unabhaengig von Support/Resistance. Die Rendering-Logik muss `sources.includes('VWAP')` pruefen BEVOR die Support/Resistance-Farbe angewendet wird.
- **Playwright-Test-Setup:** Fuer die Screenshot-Tests braucht es einen Backtest-Lauf mit Daten die sowohl LEVEL- als auch BAND-Zonen erzeugen. IREN 23.02 ist der ideale Kandidat (hat beides). Alternativ: Mock-Backend mit festen Zonen-Daten fuer deterministische Screenshots.
- **CSS Modules:** Neue Styles fuer Bands (Opacity, Dash-Pattern) in CSS Modules definieren, nicht inline. Dark Theme ist der einzige Theme -- kein Light-Theme-Fallback noetig.
