# ODIN-069: Order Flow Imbalance (OFI)

## Modul

odin-data, odin-brain, odin-api, odin-broker

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

L

## Kontext

OFI quantifiziert das Ungleichgewicht zwischen Bid- und Ask-Seite im Orderbuch auf Best-Level. Statt nur den Spread zu pruefen (ODINs aktuelles Spread-Gate), misst OFI den tatsaechlichen Kauf-/Verkaufsdruck. Damit wird das Gate-System "execution-aware" — es erkennt Adverse-Selection-Situationen bevor sie im Preis sichtbar werden. Cont/Kukanov/Stoikov zeigen eine lineare Beziehung zwischen OFI und Preisaenderungen. OFI ist die natuerliche Erweiterung des existierenden Spread-Gates zu einem echten Liquidity-Gate.

## Scope

### In Scope

- **OFI-Berechnung auf Best-Bid/Best-Ask-Level** (Top of Book): Netto-Aenderung in Bid-Size minus Netto-Aenderung in Ask-Size pro Zeitintervall
- **Z-Score-Normalisierung** mit rollierendem Fenster (konfigurierbar, Default 20 Bars)
- **OFI als zusaetzlicher KPI** in `IndicatorResult`
- **Integration als neues Gate** in der Gate-Kaskade (`OfiGate`) oder als Modifier im bestehenden `SpreadGate`
- **Tick-Daten-Erfassung:** Best Bid/Ask/Size aus IB TWS API (reqMktData) verarbeiten
- **Fallback wenn keine Top-of-Book-Daten verfuegbar:** Gate neutral (PASS) mit Warning

### Out of Scope

- Multi-Level OFI (MLOFI) mit L2-Depth (Phase 2, erfordert IB L2-Subscription)
- OFI-basiertes Signal fuer Entry-Timing (nur Gate/Filter, nicht Signal)
- VPIN (Volume-Synchronized Probability of Informed Trading) — separates Thema 20
- OFI-Visualisierung im Frontend

## Akzeptanzkriterien

- [ ] `MarketSnapshot` (`de.its.odin.api.dto.MarketSnapshot`) enthaelt neue Felder: `bestBidPrice`, `bestAskPrice`, `bestBidSize`, `bestAskSize` (alle double, NaN wenn nicht verfuegbar)
- [ ] `OfiCalculator` berechnet OFI pro Bar: `OFI(t) = deltaBidSize(t) - deltaAskSize(t)` wobei `deltaBidSize = bidSize(t) - bidSize(t-1)` wenn bidPrice unveraendert, sonst `+bidSize(t)` bei bidPrice hoch, `-bidSize(t-1)` bei bidPrice runter (analog fuer Ask)
- [ ] `OfiCalculator` berechnet Z-Score: `ofiZScore = (OFI - rollingMean) / rollingStdDev` mit konfigurierbarem Fenster
- [ ] `OfiGate` implementiert `EntryGate` Interface: FAIL wenn `ofiZScore < -2.0` (starker Verkaufsdruck, konfigurierbar)
- [ ] `IndicatorResult` enthaelt neue Felder: `ofiRaw` (double), `ofiZScore` (double)
- [ ] `IbMarketDataFeed` leitet Best-Bid/Ask-Daten aus `tickPrice`/`tickSize` Callbacks an `MarketSnapshot` weiter
- [ ] Wenn Best-Bid/Ask nicht verfuegbar (NaN): OfiGate liefert PASS mit Warnung "OFI data unavailable"
- [ ] Unit-Test verifiziert: OFI-Berechnung fuer bekannte Bid/Ask-Sequenz liefert korrekte Werte
- [ ] Unit-Test verifiziert: Z-Score-Normalisierung konvergiert fuer stationaere Daten gegen N(0,1)
- [ ] Unit-Test verifiziert: OfiGate FAIL bei ofiZScore < -2.0

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `OfiCalculator` | odin-brain | `de.its.odin.brain.kpi` | Berechnet OFI und Z-Score. Haelt rollierenden State (letzte N OFI-Werte). Pro-Pipeline-POJO |
| `OfiGate` | odin-brain | `de.its.odin.brain.quant.gates` | Implementiert `EntryGate`. Prueft ofiZScore gegen Schwelle |
| `TopOfBookSnapshot` | odin-api | `de.its.odin.api.dto` | Record: bestBidPrice, bestAskPrice, bestBidSize, bestAskSize, timestamp |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `MarketSnapshot` (`de.its.odin.api.dto.MarketSnapshot`) | Neue Felder: `bestBidPrice`, `bestAskPrice`, `bestBidSize`, `bestAskSize` (double, NaN-Default) |
| `IbMarketDataFeed` (`de.its.odin.broker.ib.IbMarketDataFeed`) | Tick-Price/Tick-Size Callbacks fuer BID/ASK/BID_SIZE/ASK_SIZE in MarketSnapshot schreiben |
| `OdinEWrapper` (`de.its.odin.broker.ib.OdinEWrapper`) | Bid/Ask-Ticks an DataPipelineService weiterleiten |
| `IndicatorResult` (`de.its.odin.api.dto.IndicatorResult`) | Neue Felder: `ofiRaw`, `ofiZScore` (double, NaN-Default) |
| `KpiEngine` (`de.its.odin.brain.kpi.KpiEngine`) | OfiCalculator aufrufen wenn Top-of-Book-Daten vorhanden |
| `GateCascadeEvaluator` (`de.its.odin.brain.quant.GateCascadeEvaluator`) | OfiGate in Gate-Kaskade aufnehmen (nach SpreadGate, vor VolumeGate) |
| `BrainProperties` (`de.its.odin.brain.config.BrainProperties`) | Nested-Record `OfiProperties`: `enabled`, `zScoreWindow`, `failThreshold` |
| `GateType` (`de.its.odin.api.model.GateType`) | Neuer Enum-Wert `OFI` |

### OFI-Algorithmus (Cont/Kukanov/Stoikov 2014)

```
Wenn bidPrice(t) > bidPrice(t-1):
    deltaBid = +bidSize(t)
Wenn bidPrice(t) < bidPrice(t-1):
    deltaBid = -bidSize(t-1)
Wenn bidPrice(t) == bidPrice(t-1):
    deltaBid = bidSize(t) - bidSize(t-1)

Analog fuer Ask (mit umgekehrtem Vorzeichen):
Wenn askPrice(t) < askPrice(t-1):
    deltaAsk = +askSize(t)
Wenn askPrice(t) > askPrice(t-1):
    deltaAsk = -askSize(t-1)
Wenn askPrice(t) == askPrice(t-1):
    deltaAsk = askSize(t) - askSize(t-1)

OFI(t) = deltaBid - deltaAsk
```

### Konfiguration

```properties
odin.brain.ofi.enabled=false
odin.brain.ofi.z-score-window=20
odin.brain.ofi.fail-threshold=-2.0
odin.brain.ofi.elevated-fail-threshold=-1.5
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 6: Order Flow Imbalance (OFI), Abschnitte IST-Zustand, SOLL-Zustand, Cont/Kukanov/Stoikov-Referenz
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine, Indikator-Berechnung
- `docs/backend/architecture/03-broker-integration.md` — IB TWS API, OdinEWrapper, Tick-Daten

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Abhaengigkeitsregeln (odin-broker -> odin-api nur)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Port-Abstraktion, ENUM statt String

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data,odin-brain,odin-api,odin-broker`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (GateType.OFI)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.ofi.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `OfiCalculator` (bekannte Bid/Ask-Sequenzen)
- [ ] Unit-Tests fuer `OfiGate` (Schwellen-Logik)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer Port-Interfaces

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: GateCascadeEvaluator mit OfiGate End-to-End
- [ ] Integrationstest: KpiEngine mit OfiCalculator
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases diskutiert (kein Top-of-Book, Spread-Crossing, Size=0)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Numerik, NaN-Handling, State-Management)
- [ ] Dimension 2: Konzepttreue-Review (Cont et al. 2014 vs. Implementierung)
- [ ] Dimension 3: Praxis-Review (IB-Datenqualitaet, Tick-Frequenz, Markt-Oeffnungszeiten)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **KRITISCH: IB TWS API Datenverfuegbarkeit pruefen.** Erster Schritt: Verifizieren ob `reqMktData()` mit Generic Tick Types Best-Bid/Ask-Price UND Best-Bid/Ask-Size liefert. Die Standard-Tick-Types 1 (Bid), 2 (Ask), 0 (BidSize), 3 (AskSize) sollten das abdecken. Im `OdinEWrapper.tickPrice()`/`tickSize()` pruefen ob diese Callbacks bereits ankommen und nur nicht weitergeleitet werden.
- **Backtest-Kompatibilitaet:** Im Backtest sind typischerweise keine Top-of-Book-Daten vorhanden (nur OHLCV). OfiGate muss in diesem Fall PASS zurueckgeben mit klarem Log-Hinweis. Alternativ: synthetische Bid/Ask aus OHLCV schaetzen (midPrice +/- spreadEstimate/2).
- **Z-Score-Warmup:** Die ersten N Bars (=z-score-window) haben keinen validen Z-Score. In dieser Phase PASS zurueckgeben.
- **Gate-Reihenfolge:** OfiGate sollte NACH SpreadGate in der Kaskade stehen (SpreadGate ist billiger zu berechnen). Wenn Spread bereits FAIL, braucht OFI nicht evaluiert zu werden.
- **Feature-Flag:** OFI ist Default-off (`odin.brain.ofi.enabled=false`). Erlaubt schrittweise Aktivierung und A/B-Vergleich.
- **OFI-Vorzeichen-Konvention:** Positiver OFI = Kaufdruck ueberwiegt (bullish). Negativer OFI = Verkaufsdruck ueberwiegt (bearish). Das Gate soll bei stark negativem OFI (Z-Score < -2.0) Entries blockieren, weil Adverse Selection droht.
