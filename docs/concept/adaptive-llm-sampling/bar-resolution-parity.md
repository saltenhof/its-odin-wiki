# Bar-Resolution-Paritaet: Live ↔ Backtest

> **Status:** ENTWURF
> **Datum:** 2026-03-02
> **Kontext:** Analyse der Bar-Resolution-Architektur zeigt fundamentale Diskrepanz zwischen Live- und Backtest-Modus. Der Backtest nutzt einen komplett anderen Datenpfad als Live — Backtestergebnisse sind damit nur eingeschraenkt aussagekraeftig fuer das Live-Verhalten.
> **Bezug:** concept/adaptive-llm-sampling/concept.md (Voraussetzung fuer sinnvolle Sampling-Optimierung)

---

## 1. Problemstellung

### 1.1 Designprinzip

Die Bar-Aufloesung (1m, 3m, 5m) ist eine **algorithmische Entscheidung** pro Indikator/KPI, nicht eine Frage des Betriebsmodus. Welche Aufloesung fuer RSI, EMA, VolumeRatio, S/R-Erkennung oder Pattern-Detection am besten funktioniert, haengt an der Natur des Algorithmus — nicht daran, ob Live oder Backtest laeuft.

**Konsequenz:** Live und Backtest MUESSEN denselben Datenpfad nutzen, sonst ist der Backtest wertlos als Praediktor fuer Live-Verhalten.

### 1.2 Ist-Zustand (kritisch)

Der Backtest nutzt einen **komplett anderen Datenpfad** als der Live-Modus:

```
LIVE:   Ticks → BarBuilder(1m) → BarAggregator(3m/5m/15m/30m) → RollingDataBuffer → Snapshot
BACKTEST: DB(5m) → BAR_APPROXIMATED → direkt in 5m-Buffer → Snapshot (ohne Aggregation)
```

**Konkrete Diskrepanzen:**

| Komponente | Live-Modus | Backtest-Modus | Impact |
|------------|-----------|----------------|--------|
| **1m-Bars** | Vorhanden (aus BarBuilder) | **LEER** — nie produziert | S/R Engine, VPIN, Monitor Events, Opening Pattern Detector funktionieren nicht |
| **3m-Bars** | Vorhanden (aus BarAggregator) | **LEER** — nie produziert | EMA9/21, VolumeRatio, Pattern FSMs nutzen 5m-Fallback statt 3m |
| **5m-Bars** | Aus Aggregation (1m→5m) | Direkt aus DB | Korrekt, aber einziger befuellter Buffer |
| **BarAggregator** | Aktiv (1m→3m→5m→15m→30m) | **Komplett umgangen** | Gesamte Aggregationslogik untested im Backtest |
| **Decision-Bar** | 3m (konfiguriert) | 5m (hardcoded via BAR_APPROXIMATED) | Alle decision-bar-abhaengigen Algorithmen laufen mit anderer Aufloesung |
| **S/R Level Engine** | Nutzt `snapshot.oneMinBars()` | Erhaelt leere Liste | **S/R-Levels werden im Backtest nicht berechnet** |
| **VPIN** | Nutzt `oneMinBars` | Erhaelt leere Liste | **VPIN im Backtest nicht berechnet** |
| **Monitor Events** | Layer 1 — jeder 1m-Bar | **Inaktiv** (keine echten 1m-Bars) | Crash/Spike-Detection, Exhaustion-Events fehlen |
| **Opening Pattern Detector** | Nutzt `snapshot.oneMinBars()` | Erhaelt leere Liste | **Opening Patterns im Backtest nicht erkannt** |
| **EMA9, EMA21** | 3m-Series (decision) | 5m-Fallback | Kurzfristige EMAs glaetten staerker als gewollt |
| **VolumeRatio** | 3m-Series (decision) | 5m-Fallback | Volume-Spikes werden gedaempft |
| **Pattern FSMs** | Arbeiten auf 3m-Bars | Arbeiten auf 5m-Bars | Pattern-Timing und -Erkennung weichen ab |
| **LLM-Sampling** | Alle 5 Bars × 3m = 15 Min | Alle 5 Bars × 5m = 25 Min | Fundamentaler Frequenzunterschied |

### 1.3 Code-Stellen der Diskrepanz

**Ursache 1 — BAR_APPROXIMATED-Bypass** (`DataPipelineService.java`, Zeilen 603-605):

```java
if (activeFlags.contains(DataFlag.BAR_APPROXIMATED)) {
    buffer.addFiveMinBar(bar);          // direkt in 5m-Buffer
    decisionBarProduced = true;         // als Decision-Bar markiert
    // BarAggregator wird NICHT aufgerufen
    // 1m-Buffer wird NICHT befuellt
    // 3m-Buffer wird NICHT befuellt
}
```

**Ursache 2 — BarInterval-Enum** (`BarInterval.java`):

```java
public enum BarInterval {
    ONE_MINUTE(60),
    FIVE_MINUTES(300);
    // THREE_MINUTES fehlt — aber das ist nicht das Hauptproblem
}
```

**Ursache 3 — RollingDataBuffer.getDecisionBars() hardcoded** (Zeile 159):

```java
public List<Bar> getDecisionBars() {
    return threeMinBars.toList();  // IMMER 3m, unabhaengig von Konfiguration
}
```

**Ursache 4 — BarSeriesManager.resolveDuration() hardcoded** (Zeilen 157-164):

```java
case TIMEFRAME_DECISION -> Duration.ofSeconds(DURATION_3M_SECONDS); // 180s hardcoded
```

### 1.4 Schweregrad

Dies ist kein kosmetisches Problem. Der Backtest simuliert ein **fundamental anderes System** als Live:

- **S/R-Levels fehlen komplett** — die gesamte Support/Resistance-Logik existiert im Backtest nicht
- **VPIN fehlt** — Orderflow-Analyse nicht vorhanden
- **Opening Patterns fehlen** — Gap-and-Go, Opening Consolidation etc. werden nicht erkannt
- **Monitor Events fehlen** — Crash-Down, Spike-Up Events werden nicht detektiert
- **Kurzfristige Indikatoren sind zu traege** — EMA9 auf 5m statt 3m reagiert deutlich langsamer auf Kursbewegungen

Ein Backtest-Ergebnis, das auf diesen Luecken basiert, kann **keine valide Aussage** ueber das Live-Verhalten treffen.

---

## 2. Loesung: Einheitlicher Datenpfad

### 2.1 Designprinzip

```
Beide Modi:  1m-Bars → BarAggregator(3m/5m/15m/30m) → RollingDataBuffer → Snapshot
```

Der Backtest replayed **1-Minuten-Bars** und nutzt **denselben BarAggregator** wie der Live-Modus. Die Konfiguration (`decision-bar-timeframe-s`) bestimmt einheitlich, welcher Timeframe die Decision-Cycle-Frequenz steuert.

### 2.2 Anforderung: 1-Minuten-Bars als Eingangsdaten fuer Backtests

Der Backtest muss 1-Minuten-Bars aus der Datenbank laden. Welcher Datenprovider diese geliefert hat (Yahoo Finance, Databento, oder kuenftige Provider) ist irrelevant — das `HistoricalDataProvider`-Interface abstrahiert die Quelle bereits. Alle aktuell angebundenen Provider unterstuetzen 1-Minuten-Bars.

**Anforderung:** Der Backtest-Datenpfad laedt Bars mit `interval_seconds=60` aus `odin.market_bar`. Sind fuer einen angefragten Tag keine 1m-Bars vorhanden, schlaegt der Backtest mit einer klaren Fehlermeldung fehl — kein stiller Fallback auf 5m.

### 2.3 Anforderung: Einheitlicher Aggregationspfad

Der `BAR_APPROXIMATED`-Bypass in `DataPipelineService` muss entfallen. Stattdessen durchlaufen Backtest-Bars denselben `BarAggregator`-Pfad wie Live-Bars:

1. 1m-Bar wird in den 1m-Buffer gelegt
2. `BarAggregator.onOneMinBar()` produziert 3m, 5m, 15m, 30m-Bars (identisch zum Live-Modus)
3. Der konfigurierte `decision-bar-timeframe-s` bestimmt, welche Aggregation den Decision-Cycle ausloest

Das `BAR_APPROXIMATED`-Flag behält seine semantische Bedeutung ("kein Intra-Bar-Tick-Level vorhanden"), steuert aber nicht mehr den Datenpfad. Es dient nur noch als Hinweis fuer Komponenten, die Tick-Level-Daten benoetigen (z.B. OFI-Berechnung → wird bei APPROXIMATED uebersprungen).

### 2.4 Anforderung: Dynamische Decision-Bar-Aufloesung

Zwei Hardcodings muessen aufgeloest werden:

1. `RollingDataBuffer.getDecisionBars()` gibt aktuell immer den 3m-Buffer zurueck, unabhaengig von der Konfiguration. Diese Methode muss den konfigurierten `decision-bar-timeframe-s` beruecksichtigen.

2. `BarSeriesManager.resolveDuration()` hat die Decision-Duration auf 180s hardcoded. Dieser Wert muss aus der Konfiguration kommen.

### 2.5 Anforderung: SimClock-Stepping auf 1m

Der `SimulationRunner` stepped die SimClock aktuell in `barIntervalSeconds`-Schritten (300s bei 5m). Mit 1m-Bars wird in 60s-Schritten gestepped. Die reine Sim-Logik laeuft in Millisekunden pro Iteration — der limitierende Faktor bleibt die LLM-Call-Latenz, die sich durch die Resolution nicht aendert.

---

## 3. Aenderungskatalog

### 3.1 Anforderungskatalog (Prio 1 — Backtest-Live-Paritaet)

| # | Anforderung | Betroffene Stelle | Aufwand |
|---|-------------|-------------------|---------|
| 1 | `BarInterval`-Enum um `THREE_MINUTES` erweitern | `BarInterval.java` | Trivial |
| 2 | Backtest-Default auf 1-Minuten-Bars umstellen | Backtest-Konfiguration | Trivial |
| 3 | BAR_APPROXIMATED-Bypass entfernen — alle Bars durch BarAggregator leiten | `DataPipelineService` | Mittel |
| 4 | `getDecisionBars()` dynamisch anhand konfiguriertem Timeframe | `RollingDataBuffer` | Klein |
| 5 | Decision-Duration konfigurierbar statt hardcoded 180s | `BarSeriesManager` | Klein |
| 6 | SimClock-Stepping auf 1-Minuten-Intervall | `SimulationRunner` | Klein |
| 7 | Backtest-Datendownload mit 1m-Aufloesung (provider-agnostisch ueber `HistoricalDataProvider`) | Backtest-Datenpfad | Mittel |
| 8 | `decision-bar-timeframe-s` als einheitlicher Default fuer Live und Backtest | Properties | Trivial |

### 3.2 Validierung (Prio 1 — parallel zur Implementierung)

| # | Anforderung | Akzeptanzkriterium |
|---|-------------|-------------------|
| 1 | 1m-Bars im Snapshot vorhanden | `snapshot.oneMinBars()` nicht leer |
| 2 | 3m-Bars im Buffer vorhanden | `buffer.getThreeMinBarCount() > 0` |
| 3 | EMA9/21 auf Decision-Series (nicht 5m-Fallback) | KpiEngine nutzt `seriesDecision`, nicht `series5m` |
| 4 | Monitor Events werden detektiert | Log enthaelt MonitorEvent-Eintraege |
| 5 | VPIN wird berechnet | `snapshot.vpin() != NaN` |
| 6 | LLM-Sampling-Frequenz konsistent mit Live | Bei `intervalBars=5` und 3m-Decision-Bars: ~15 Min |
| 7 | S/R-Levels werden berechnet | S/R Engine erhaelt 1m-Bars, produziert Levels |
| 8 | A/B-Vergleich: gleicher Tag, 1m vs. alter 5m | Abweichungen dokumentiert und erklaert |

### 3.3 Datenanforderung (Voraussetzung)

1-Minuten-Bars muessen in der Datenbank (`odin.market_bar` mit `interval_seconds=60`) vorliegen, bevor ein Backtest gestartet werden kann. Die Beschaffung erfolgt provider-agnostisch ueber das `HistoricalDataProvider`-Interface — welcher Provider die Daten liefert, ist eine Konfigurationsfrage und nicht Gegenstand dieses Konzepts.

Falls fuer einen angefragten Tag keine 1m-Bars in der DB vorhanden sind, muss der Backtest mit einer klaren Fehlermeldung abbrechen: `"No 1-minute bars available for {symbol} on {date}. Download required."` Kein stiller Fallback auf 5m.

### 3.4 Nicht-Aenderungen (bewusste Entscheidungen)

| Aspekt | Entscheidung | Begruendung |
|--------|-------------|-------------|
| `decision-bar-timeframe-s` aendern | NEIN — bleibt bei 180 (3m) | 3m ist der bewusst gewaehlte Kompromiss aus Aufloesung und Rauschen |
| 5m komplett abschaffen | NEIN | 5m bleibt fuer feste KPIs (RSI, ATR, ADX, Bollinger, EMA50/100) |
| 15m/30m-Bars im Backtest | Automatisch — BarAggregator produziert sie aus 5m-Cascade | Keine explizite Aenderung noetig |
| OFI-Berechnung im Backtest | Weiterhin deaktiviert (BAR_APPROXIMATED = kein Tick-Level) | Erfordert L1/L2-Daten, nicht nur OHLCV |

---

## 4. Laufzeit-Impact

### 4.1 Backtest-Laufzeit mit 1m-Bars

| Metrik | 5m-Bars (aktuell) | 1m-Bars (neu) |
|--------|-------------------|---------------|
| Bars pro Handelstag | ~78 | ~390 |
| Decision-Bars pro Tag | ~78 (alle 5m-Bars) | ~130 (alle 3m-Bars) |
| Sim-Iterationen | 78 | 390 |
| Sim-Logik-Zeit pro Bar | ~1-2ms | ~1-2ms |
| Sim-Logik gesamt | ~0.1s | ~0.5s |
| LLM-Calls (bei intervalBars=5) | ~16 (78/5) | ~26 (130/5) |
| LLM-Call-Zeit (90s/Call) | ~24 Min | ~39 Min |
| **Gesamtlaufzeit** | **~25 Min** | **~40 Min** |

Die Laufzeitsteigerung (~60%) wird fast ausschliesslich durch die hoeheren LLM-Call-Anzahl verursacht (26 statt 16 Decision-Bar-Trigger). Die reine Sim-Logik bleibt vernachlaessigbar.

### 4.2 Mitigation: CachedAnalyst

Fuer wiederholte Backtests ueber denselben Tag kann der `CachedAnalyst` verwendet werden (04-llm-integration.md §16). Beim ersten Lauf werden alle LLM-Responses gecacht; bei Replay-Laeufen entfaellt die LLM-Latenz komplett. Die Sim-Laufzeit sinkt dann auf unter 1 Sekunde.

---

## 5. Risiken

| Risiko | Wahrscheinlichkeit | Mitigation |
|--------|---------------------|-----------|
| Bestehende Backtestergebnisse nicht mehr vergleichbar | Hoch (gewollt) | A/B-Dokumentation erstellen. Alte Ergebnisse als "v1 (5m-only)" kennzeichnen |
| 1m-Daten nicht fuer alle historischen Tage vorhanden | Mittel | Klare Fehlermeldung bei fehlenden Daten. Download vor Backtest-Start (provider-agnostisch) |
| Regressions in S/R Engine bei neuem Datenpfad | Niedrig | S/R Engine hat bereits Unit-Tests. Zusaetzlich: Backtest-Vergleichslauf |
| BarAggregator-Bugs bei hohem Durchsatz | Niedrig | BarAggregator hat eigene Unit-Tests. 390 Bars/Tag ist kein hoher Durchsatz |

---

## 6. Zusammenfassung

**Kern-Erkenntnis:** Der aktuelle Backtest simuliert ein fundamental anderes System als das Live-System. Kritische Komponenten (S/R Engine, VPIN, Monitor Events, Opening Patterns) sind im Backtest **komplett inaktiv**. Kurzfristige Indikatoren (EMA9/21, VolumeRatio) arbeiten mit falscher Aufloesung. Die Backtestergebnisse sind damit nur eingeschraenkt aussagekraeftig.

**Loesung:** Der Backtest replayed 1-Minuten-Bars und nutzt denselben `BarAggregator`-Datenpfad wie der Live-Modus. Der `BAR_APPROXIMATED`-Bypass wird entfernt. Alle Konfigurationsparameter (insbesondere `decision-bar-timeframe-s`) gelten einheitlich fuer beide Modi. Die Datenquelle fuer 1m-Bars ist provider-agnostisch ueber das bestehende `HistoricalDataProvider`-Interface abstrahiert.

**Aufwand:** ~8 Anforderungen, vorwiegend klein bis mittel. Groesster Aufwand: BAR_APPROXIMATED-Pfad in der DataPipelineService vereinheitlichen.
