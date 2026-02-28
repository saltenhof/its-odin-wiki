# ODIN-065: Intraday Seasonality Normalization

## Modul

odin-data, odin-brain, odin-api

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

M

## Kontext

Intraday-Maerkte zeigen robuste periodische Muster: Volumen und Volatilitaet folgen einer U-foermigen Kurve (hoch am Open/Close, tief am Mittag). Ohne Beruecksichtigung dieser Tagesstruktur erzeugen KPIs wie `volumeRatio` und `atrDecayRatio` systematische Fehlsignale — Mittags-Ruhe wird als Anomalie interpretiert, waehrend Opening-Volume faelschlich als Spike gewertet wird. Diese Story fuehrt "Expected Curves" ein, um KPIs tageszeitbereinigt zu berechnen und damit die Gate-Kaskade ueber den gesamten Handelstag stabil zu machen.

## Scope

### In Scope

- Neues Package `de.its.odin.data.seasonality` mit Service zur Berechnung und Verwaltung von Expected Curves
- Expected-Volume-Kurve pro Instrument: `expectedVolume(slot)` fuer 5-Minuten-Slots (78 Slots pro US-RTH-Session)
- Expected-Volatility-Kurve pro Instrument: `expectedVolatility(slot)` analog
- Normalisierte KPIs: `volumeSurprise = volume(t) / expectedVolume(slot(t))` als Ersatz fuer rohen `volumeRatio`
- Integration in `VolumeGate` (`de.its.odin.brain.quant.gates.VolumeGate`): normalisierter volumeSurprise statt absoluter volumeRatio
- Integration in `AtrGate` (`de.its.odin.brain.quant.gates.AtrGate`): normalisierte Volatilitaet
- Konfiguration via `BrainProperties` (Lookback-Tage, Min-Datenpunkte fuer Kurven-Schaetzung)
- Fallback auf aktuelles Verhalten wenn unzureichende historische Daten vorhanden

### Out of Scope

- Expected-Spread-Kurve (spaetere Erweiterung)
- Cluster-basierte Kurven (mehrere Instrumente pro Cluster) — erst in Phase 2
- Automatische Kurven-Aktualisierung waehrend Live-Session (Kurven werden vor RTH-Open berechnet)
- UI-Darstellung der Expected Curves

## Akzeptanzkriterien

- [ ] `IntradaySeasonalityService` berechnet Expected-Volume-Kurve aus historischen 5-Min-Bars (konfigurierbar, Default: 20 Handelstage Lookback)
- [ ] `IntradaySeasonalityService` berechnet Expected-Volatility-Kurve analog
- [ ] Pro 5-Min-Slot (78 Slots fuer US-RTH 09:30-16:00 ET) wird der Median des historischen Volumens/der Volatilitaet als Expected-Wert verwendet
- [ ] `volumeSurprise(t) = volume(t) / expectedVolume(slot(t))` wird als normalisierter KPI berechnet
- [ ] `VolumeGate` verwendet `volumeSurprise` statt `volumeRatio` wenn Expected Curves verfuegbar sind
- [ ] `AtrGate` verwendet normalisierte Volatilitaet wenn Expected Curves verfuegbar sind
- [ ] Wenn weniger als `minDaysForCurve` (Default: 10) historische Tage vorhanden sind, wird auf das bisherige Verhalten zurueckgefallen (roher volumeRatio)
- [ ] Unit-Test verifiziert: `volumeSurprise(t) = volume(t) / expectedVolume(slot(t))` fuer bekannte Testdaten
- [ ] Unit-Test verifiziert: Mittags-Slot (12:00-12:05 ET) mit niedrigem absolutem Volumen aber normalem Verhaeltnis zur Expected Curve erzeugt PASS (statt bisherigem FAIL)
- [ ] Unit-Test verifiziert: Opening-Slot (09:30-09:35 ET) mit hohem absolutem Volumen aber normalem Verhaeltnis zur Expected Curve erzeugt kein falsches Spike-Signal

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `IntradaySeasonalityService` | odin-data | `de.its.odin.data.seasonality` | Berechnet Expected Curves aus historischen Bars. Input: Liste von 5-Min-Bars ueber N Tage. Output: Map<Integer, Double> (Slot-Index -> Expected-Wert) |
| `ExpectedCurve` | odin-api | `de.its.odin.api.dto` | Record mit `Map<Integer, Double> expectedVolume`, `Map<Integer, Double> expectedVolatility`, `int lookbackDays`, `LocalDate computedDate` |
| `SeasonalitySlot` | odin-data | `de.its.odin.data.seasonality` | Utility: berechnet Slot-Index aus Zeitstempel (z.B. 09:30 ET = Slot 0, 09:35 ET = Slot 1, ..., 15:55 ET = Slot 77) |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `VolumeGate` (`de.its.odin.brain.quant.gates.VolumeGate`) | Neuer Konstruktor-Parameter `ExpectedCurve`. Wenn vorhanden, `volumeSurprise` statt `volumeRatio` verwenden |
| `AtrGate` (`de.its.odin.brain.quant.gates.AtrGate`) | Analog: normalisierte Volatilitaet wenn ExpectedCurve vorhanden |
| `KpiEngine` (`de.its.odin.brain.kpi.KpiEngine`) | Berechnet `volumeSurprise` als zusaetzlichen Indikator wenn ExpectedCurve injiziert |
| `IndicatorResult` (`de.its.odin.api.dto.IndicatorResult`) | Neues Feld `volumeSurprise` (Optional, Double.NaN als Default fuer Rueckwaertskompatibilitaet) |
| `BrainProperties` (`de.its.odin.brain.config.BrainProperties`) | Neue Nested-Record `SeasonalityProperties` mit `lookbackDays` (Default 20), `minDaysForCurve` (Default 10) |
| `PipelineFactory` (`de.its.odin.core.pipeline.PipelineFactory`) | ExpectedCurve an KpiEngine und Gates durchreichen |

### Algorithmus: Expected-Volume-Berechnung

```
fuer jeden Slot s (0..77):
    sammle Volumen-Werte aus den letzten N Handelstagen fuer Slot s
    expectedVolume[s] = median(gesammelte Werte)
```

Median statt Mean zur Robustheit gegen Outlier (z.B. Earnings-Tage).

### Konfiguration

```properties
odin.brain.seasonality.lookback-days=20
odin.brain.seasonality.min-days-for-curve=10
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 1: Intraday-Seasonality-Normalisierung, Abschnitt IST-Zustand und SOLL-Zustand
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine-Architektur, Indikator-Berechnung
- `docs/backend/architecture/02-realtime-pipeline.md` — Datenpipeline, Snapshot-Erzeugung

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln (kein var, explizite Typen, Records fuer DTOs, JavaDoc-Pflicht)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data,odin-brain,odin-api`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.seasonality.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `IntradaySeasonalityService`, `SeasonalitySlot`, `ExpectedCurve`
- [ ] Unit-Tests fuer modifizierte `VolumeGate` und `AtrGate` mit Expected Curves
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstests mit realen Klassen (nicht alles weggemockt)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Gate-Evaluierung mit Expected Curves End-to-End durchspielt

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session gestartet mit: Klasse(n), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Race Conditions, Null-Safety, Performance)
- [ ] Dimension 2: Konzepttreue-Review (Implementierung vs. theme-backlog.md Thema 1)
- [ ] Dimension 3: Praxis-Review (unbehandelte Szenarien, reale Marktbedingungen)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten (Working State, Design-Entscheidungen, Offene Punkte, ChatGPT-Sparring, Gemini-Review)

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Median statt Mean:** Fuer Expected-Werte den Median verwenden. Der Mean wird von Outlier-Tagen (Earnings, FDA-Entscheidungen, FOMC) stark verzerrt.
- **Slot-Berechnung:** US-RTH ist 09:30-16:00 ET = 390 Minuten = 78 Slots a 5 Minuten. Die Slot-Berechnung muss mit der US-Eastern-Timezone arbeiten, nicht UTC.
- **Historische Daten:** Die Expected Curves werden VOR dem RTH-Open berechnet. Im Backtest stehen die Daten direkt zur Verfuegung (HistoricalMarketDataFeed). Im Live-Betrieb muessen die Bars aus der Datenbank oder einem Cache stammen.
- **Rueckwaertskompatibilitaet:** `volumeSurprise` wird als optionaler Zusatz in `IndicatorResult` eingefuegt. Bestehende Logik, die `volumeRatio` nutzt, bleibt funktionsfaehig wenn keine Expected Curve vorhanden ist.
- **Testdaten:** Fuer Unit-Tests synthetische 5-Min-Bars generieren, die das U-Muster reproduzieren (hohes Volumen am Open/Close, niedriges Volumen um 12:00 ET). Dann verifizieren, dass der normalisierte Wert stabil bleibt.
- **Abhaengigkeit auf odin-data:** `IntradaySeasonalityService` lebt in odin-data, weil er historische Bars als Input braucht. Das `ExpectedCurve`-DTO lebt in odin-api, damit odin-brain es verwenden kann ohne direkte Abhaengigkeit auf odin-data.
