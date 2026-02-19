# ODIN — Kapitel 2: Echtzeit-Datenpipeline

Version: 0.4 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R3 — P1 Nachschaerfungen (Tick-Queue Guardrails, Spool-Durability, TradingCalendar-Einordnung, L2 Best-Effort)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt die Datenpipeline von ODIN: Wie Marktdaten ueber den `MarketDataFeed`-Port empfangen, validiert, zu Bars aggregiert und als immutable Snapshots an die nachgelagerten Schichten (Brain, Execution) verteilt werden. Die Pipeline ist **pro Instrument instanziiert** — jede Pipeline hat eigene Buffer, BarBuilder, DQ-Gates und Aggregationen.

**Modulzuordnung:** `odin-data` (Package: `de.odin.data`)

**Zentrale Abhaengigkeiten (Ports aus odin-api):**
- `MarketDataFeed` — Datenquelle (Live: IbMarketDataFeed, Sim: HistoricalMarketDataFeed)
- `MarketClock` — Zeitreferenz fuer alle Zeitstempel und zeitbasierte Logik. **Stellt auch Session-Informationen bereit** (RTH-Start/End, Half-Days, Feiertage) via integriertem `TradingCalendar` — kein separater Port
- `EventLog` — Persistierung entscheidungsrelevanter Events fuer Record/Replay

> **TradingCalendar ist kein eigener Port.** Session-Zeiten (RTH, Feiertage, Half-Days) werden ueber den `MarketClock`-Port abgefragt (`MarketClock.isWithinRth()`, `MarketClock.getSessionBoundaries()`). Das ist konsistent mit Kap 0 (4+1 Ports) — MarketClock traegt bereits die Session-Semantik (Exchange-TZ, DST). Der TradingCalendar ist ein internes Service-Interface in odin-api, das von der MarketClock-Implementierung (Live und Sim) genutzt wird.

---

## 2. Datenfluesse und Feeds

### Eingehende Feeds (via MarketDataFeed Port)

Die Data Pipeline konsumiert Marktdaten ueber den `MarketDataFeed`-Port. Die konkrete Datenquelle (IB oder historische Daten) ist fuer die Pipeline transparent.

| Feed | Datentyp | Update-Frequenz | Hinweis |
|------|----------|-----------------|---------|
| Ticks | Last, Bid, Ask, Volume | Echtzeit (jede Aenderung) | Primaere Datenquelle fuer BarBuilder |
| L2 (Market Depth) | Bid/Ask-Buch (Top 5) | Echtzeit | Konfigurierbar (`odin.data.l2.enabled`, Default: true) |
| Historical Bars | 1-Min, 5-Min, 30-Min, Daily | Einmalig beim Init + Reconnect-Gap-Fill | Fuer Buffer-Prefill und Kalibrierung |

Jedes eingehende Event traegt `marketTime` (Exchange-TZ), `instrumentId` und `sequenceNumber`. **Die Pipeline vertraut ausschliesslich auf `marketTime` und `sequenceNumber` fuer Ordering — nicht auf Arrival-Time.**

### Sequence-Contract

| Eigenschaft | Spezifikation |
|-------------|---------------|
| Scope | Pro `instrumentId`, **global ueber alle Event-Typen** (Ticks, L2, Bars) |
| Monotonie | Strikt monoton steigend. **Luecken erlaubt** (kein gap-free Versprechen). L2-Coalescing in der L2-Queue kann zusaetzliche Gaps erzeugen — das ist architektonisch gewollt |
| Reconnect | Nach Reconnect setzt der MarketDataFeed eine **neue Sequence-Baseline** (hoeher als letzte bekannte). Die Pipeline erkennt den Gap und loggt ein `DataQualityEvent(WARN, SEQUENCE_GAP)` |
| Duplikate/Reorder | **Strikt Drop** — kein Reorder-Window. Events mit `sequenceNumber <= lastSeen` werden verworfen |
| Verantwortung | Der `MarketDataFeed`-Port (Live: IbMarketDataFeed, Sim: HistoricalMarketDataFeed) garantiert die Monotonie. Die Pipeline validiert via MonotonicityGate |

> **Nicht in odin-data:** P&L-Stream, Order-Status und Account-Data werden von odin-execution bzw. odin-core direkt ueber den BrokerGateway-Port konsumiert.

### Simulation-Modus

Im Simulationsmodus liefert der `HistoricalMarketDataFeed` denselben Event-Stream — nur aus historischen Daten statt live. Zwei Varianten:

| Variante | Input | Trade-off |
|----------|-------|-----------|
| **Tick-Replay (Default)** | Historische Ticks + L2 | Volle Validierung aller Pipeline-Pfade (BarBuilder, L2-Gates, Tick-Features). **Volle SoT-Garantie** fuer VWAP, SpreadStats, Extrema |
| **Bar-Replay** | Direkt historische Bars (1-Min) | Schneller, aber Tick/L2-abhaengige Logik nicht validiert. **VWAP, SpreadStats, Intraday-Extrema werden aus Bar-Daten approximiert** und mit `BAR_APPROXIMATED`-Flag markiert |

Die Variante wird per Konfiguration (`odin.simulation.feed-mode=TICK|BAR`) gewaehlt.

> **Determinismus-Garantie nach Feed-Modus:** Im Tick-Replay sind alle Derived Features (VWAP, SpreadStats, Extrema) exakt. Im Bar-Replay werden sie aus Bar-OHLCV approximiert (VWAP ≈ typicalPrice * volume, SpreadStats = nicht verfuegbar, Extrema = Bar-High/Low) und im Snapshot mit `DataFlag.BAR_APPROXIMATED` markiert. Brain und Rules Engine muessen dieses Flag beruecksichtigen — Strategien die auf exakte VWAP/SpreadStats angewiesen sind, muessen im Tick-Replay-Modus laufen.

---

## 3. Architektur der Pipeline

```
                    MarketDataFeed (Port)
                    │ MarketEvent-Stream (Ticks, L2, Bars)
                    v
            ┌───────────────────────┐
            │     BarBuilder        │   ← Baut Decision-Bars (1m) aus Ticks
            └───────────┬───────────┘
                        │ (Bars + validierte Ticks)
                        v
            ┌───────────────────────┐
            │   Data Quality Gates  │   ← Validierungs-Chain (geordnet)
            └───────────┬───────────┘
                        │ (validierte Daten)
                        v
            ┌───────────────────────┐
            │   Rolling Data Buffer │   ← Ringpuffer, Multi-Timeframe
            │   + Aggregation       │
            └───────────┬───────────┘
                        │ (bei Decision-Trigger: Bar-Close)
                        v
            ┌───────────────────────┐
            │  MarketSnapshotFactory│   ← Immutable Snapshot + EventLog
            └───────────┬───────────┘
                        │
               ┌────────┼────────┐
               v        v        v
             Brain   Monitoring  EventLog
```

### Orchestrierung: DataPipelineService

Die zentrale Klasse `DataPipelineService` orchestriert den Datenfluss:

1. **Empfang:** Registriert sich als Listener beim `MarketDataFeed`-Port
2. **Bar-Erzeugung:** Leitet Tick-Events an den `BarBuilder` (1-Minute Decision-Bar)
3. **Validierung:** Leitet Bars und Ticks durch die DQ-Gate-Chain
4. **Pufferung:** Schreibt validierte Daten in den `RollingDataBuffer`
5. **Aggregation:** Triggert Aggregationen bei Bar-Abschluss (1m → 5m → 15m → 30m)
6. **Snapshot:** Erzeugt bei Decision-Trigger (1m-Bar-Close) einen `MarketSnapshot` via `MarketSnapshotFactory`
7. **EventLog:** Schreibt BarClose + Snapshot-Referenz ins EventLog (non-droppable)
8. **Quiescence:** Meldet dem Orchestrator (odin-core) „drained bis sequence X / bar t"
9. **Events:** Publiziert `MarketSnapshotEvent` an registrierte Listener (Brain, Monitoring)

---

## 4. BarBuilder

Der `BarBuilder` erzeugt die **Decision-Bar** (primaerer Decision-Trigger) aus eingehenden Tick-Events.

| Parameter | Default | Beschreibung |
|-----------|---------|-------------|
| Decision-Bar-Timeframe | 1 Minute | Konfigurierbar (`odin.data.decision-bar-timeframe-s=60`) |
| Bar-Boundary | MarketClock-basiert | Bar schliesst bei vollem Minute-Wechsel (MarketClock, Exchange-TZ) |

**Funktionsweise:**
- Sammelt Ticks innerhalb des aktuellen Bar-Fensters
- Bei Bar-Close (MarketClock): erzeugt OHLCV-Bar (Open = erster Tick, High = max, Low = min, Close = letzter Tick, Volume = sum)
- Im **Bar-Replay-Modus** (Simulation): BarBuilder wird umgangen, Bars kommen direkt vom Feed

> **Decision-Trigger = 1-Minute-Bar-Close.** Ticks aktualisieren den Buffer und Trailing-Stops, loesen aber keinen vollstaendigen Decision-Cycle aus. 5-Sekunden-Micro-Bars werden fuer interne Metriken erzeugt (kein Decision-Trigger, keine DQ-Validierung, kein EventLog).

---

## 5. Data Quality Gates

Die DQ-Gates bilden eine geordnete Validierungs-Chain. **Die Reihenfolge ist entscheidend** — Crash-Detection laeuft VOR dem Outlier-Filter.

### Verarbeitungsreihenfolge

```
Eingehende Daten (Bar oder Tick)
  │
  ├─ 1. BarCompletenessGate ── OHLCV vollstaendig? Volume > 0?
  │     → REJECT_EVENT: unvollstaendige Bars verwerfen
  │
  ├─ 2. MonotonicityGate ── sequenceNumber und marketTime monoton steigend?
  │     → REJECT_EVENT: Out-of-order Events verwerfen (Duplikat- und Reorder-Schutz)
  │
  ├─ 3. L2IntegrityGate ── Bid > Ask? Spread > Schwellenwert?
  │     → WARN: L2-Update ignorieren. Persistiert > 5s → REJECT_EVENT
  │     → Gate ist konfigurierbar abschaltbar (`odin.data.dq.l2-integrity-enabled`)
  │
  ├─ 4. StaleQuoteGate ── Kein neuer Tick seit > N Sekunden (MarketClock) waehrend RTH?
  │     → RTH-Zeiten via MarketClock.isWithinRth() (Exchange-spezifisch, Half-Day-aware)
  │     → WARN bei 30s. Bei 60s: ESCALATE → odin-core entscheidet Kill-Switch
  │
  ├─ 5. CrashDetectionGate ── Preis > 5% in < 1 Min UND Vol > 3x Avg?
  │     → MARK: Bar als Crash-Signal markieren (nicht verwerfen!)
  │
  └─ 6. OutlierFilterGate ── Preis > 5% in < 1 Bar UND kein Crash-Signal UND Vol normal?
        → REJECT_EVENT: Bar verwerfen (vermutlich Bad Print)
```

### Gate-Ergebnisse

| Ergebnis | Bedeutung | Event-Wirkung | System-Wirkung |
|----------|-----------|---------------|----------------|
| PASS | Daten ok | Event weiterleiten | Pipeline laeuft normal |
| WARN | Auffaelligkeit, aber verwendbar | Event weiterleiten + Flag | Logging + Alert, Pipeline laeuft weiter |
| MARK | Besonderes Signal erkannt | Event weiterleiten + Flag (z.B. CRASH_SIGNAL) | Pipeline laeuft weiter |
| REJECT_EVENT | Einzelnes Event unbrauchbar | **Event verwerfen** | Pipeline laeuft weiter (Fail-Open auf System-Ebene) |
| ESCALATE | Systemkritisches Problem | DataQualityEvent publizieren | **odin-core entscheidet** (z.B. Kill-Switch). Kein Handel bis geklaert (Fail-Closed) |

> **Terminologie:** `REJECT_EVENT` verwirft ein einzelnes Daten-Event — die Pipeline selbst laeuft weiter (System-Ebene: Fail-Open). `ESCALATE` hingegen ist Fail-Closed auf System-Ebene: kein Handel bis odin-core die Situation bewertet hat.

> **Kill-Switch-Ownership:** DQ-Gates loesen **niemals** selbst den Kill-Switch aus. Sie publizieren ein `DataQualityEvent(ESCALATE)` — die Entscheidung liegt bei odin-core (KillSwitchService).

### DQ-Gates nach Feed-Modus

| Gate | Live | Sim: Tick-Replay | Sim: Bar-Replay |
|------|------|-----------------|-----------------|
| BarCompleteness | Aktiv | Aktiv | Aktiv |
| Monotonicity | Aktiv | Aktiv | Aktiv |
| L2Integrity | Aktiv (wenn L2 enabled) | Aktiv (wenn L2-Daten vorhanden) | **Deaktiviert** (keine L2-Daten) |
| StaleQuote | Aktiv (MarketClock.isWithinRth()) | **Deaktiviert** (Events kontrolliert) | **Deaktiviert** (Events kontrolliert) |
| CrashDetection | Aktiv | Aktiv | Aktiv (Bar-basiert, gleiche Schwellen) |
| OutlierFilter | Aktiv | Aktiv | Aktiv (Bar-basiert, gleiche Schwellen) |

### Signal-Interferenz-Schutz

Der Outlier-Filter (Gate 6) prueft explizit, ob die Crash-Detection (Gate 5) den Bar bereits als reale Extrembewegung klassifiziert hat. Nur Bars die **sowohl** extreme Preisabweichung zeigen **als auch** kein erhoehtes Volumen haben, werden verworfen.

### Degradation vor Kill-Switch

Bei einzelnen Gate-Verletzungen wird zunaechst degradiert (groessere Intervalle, konservativere Regeln), bevor der Kill-Switch gezogen wird. Nur bei persistierendem Datenausfall (> 60s keine validen Ticks, MarketClock-Zeit, waehrend RTH laut `MarketClock.isWithinRth()`) eskaliert odin-data via ESCALATE-Event.

---

## 6. Rolling Data Buffer

### Speicherstruktur

Der Buffer ist ein **In-Memory Ringpuffer** mit mehreren Granularitaetsstufen:

| Stufe | Granularitaet | Kapazitaet | Quelle | Prefill |
|-------|--------------|------------|--------|---------|
| Decision-Bar | 1-Minute-Bars | Letzte 6 Stunden | BarBuilder (aus Ticks) | **Kein Prefill** — entsteht ab Live-/Replay-Start |
| 5-Min | 5-Minuten-Bars | Gesamter Handelstag | Aggregation aus Decision-Bar | **Kein Prefill** — entsteht ab Live-/Replay-Start |
| 15-Min | 15-Minuten-Bars | Gesamter Handelstag + 5 Vortage | Aggregation aus 5-Min | **Kein Prefill** — entsteht ab Live-/Replay-Start (Vortage via Historical Init) |
| 30-Min | 30-Minuten-Bars | Letzte 5 Handelstage | Init via Historical Data | Prefill beim Init |
| Daily | Tages-Bars | Letzte 20 Handelstage | Init via Historical Data | Prefill beim Init |

> **Keine Disaggregation:** 30-Min-Bars werden **nicht** in 15m/5m/1m zurueckgerechnet. Die feingranularen Buffer (1m, 5m, 15m) fuer den aktuellen Tag werden ausschliesslich aus Live-/Replay-Daten aufgebaut. Brain-Warmup (15 Min nach RTH-Start) deckt die initiale Luecke ab.

### Aggregationslogik

Aggregation erfolgt **bottom-up bei Bar-Abschluss**:

```
1-Min-Bar eingetroffen (Decision-Bar)
  → Decision-Bar-Buffer schreiben
  → Ist 5-Min abgeschlossen? → 5-Min-Bar aggregieren
    → Ist 15-Min abgeschlossen? → 15-Min-Bar aggregieren
      → Ist 30-Min abgeschlossen? → 30-Min-Bar aggregieren
```

Aggregation berechnet OHLCV: Open = erster Open, High = max(Highs), Low = min(Lows), Close = letzter Close, Volume = sum(Volumes). **Zeitgrenzen basieren auf MarketClock** (Exchange-TZ, DST-aware).

### Zusaetzliche Buffer-Daten

| Datentyp | Struktur | Verwendung |
|----------|----------|-----------|
| Tick-Buffer | Last N Ticks (FIFO, konfigurierbar) | Spread-Berechnung, Trailing-Stop-Updates |
| Spread-Tracker | Min/Max/Avg aus L2 + Ticks | Quant-Validierung (Spread-Check) |
| Intraday-Extrema | High-of-Day, Low-of-Day (laufend) | Rules Engine, Pattern-Detection |
| VWAP-Akkumulator | Kumulatives (Price x Volume) / Volume | VWAP-Berechnung (Source of Truth, Kap 0) |

### Thread-Safety

Der Buffer wird von **einem Writer-Thread** (Data-Pipeline-Thread) beschrieben und von **mehreren Reader-Threads** (KPI, Rules, LLM-Prompt-Builder) gelesen via Snapshot-Prinzip.

**Mechanismus:** Der Writer-Thread erzeugt bei jedem Decision-Trigger einen neuen immutable Snapshot und publiziert ihn via `AtomicReference`-Swap. Reader holen sich die aktuelle Referenz — kein Locking. Der Snapshot ist ein flaches Record mit kopierten Werten (keine Buffer-Referenzen).

---

## 7. MarketSnapshot (Immutable)

### Zweck

Bei jedem Decision-Cycle (1m-Bar-Close) erzeugt die `MarketSnapshotFactory` einen **immutable MarketSnapshot**. Alle nachgelagerten Berechnungen (KPI, Rules, Quant, Arbiter) arbeiten auf demselben Snapshot. Determinismus garantiert.

### Inhalt eines Snapshots

```
MarketSnapshot (Record, immutable)
├── marketTime          : Instant (von MarketClock, Exchange-TZ)
├── instrumentId        : String
├── currentPrice        : double (Last)
├── currentBid          : double
├── currentAsk          : double
├── currentSpread       : double (in %)
├── decisionBars        : List<Bar>  (letzte N 1-Min-Bars)
├── fiveMinBars         : List<Bar>  (letzte N 5-Min-Bars)
├── intradayHigh        : double
├── intradayLow         : double
├── vwap                : double (Source of Truth)
├── cumulativeVolume    : long
├── spreadStats         : SpreadStats (min, max, avg)
├── dailyBars           : List<Bar>  (letzte 20 Tage)
├── flags               : Set<DataFlag>  (z.B. CRASH_SIGNAL, STALE_WARNING, BAR_APPROXIMATED)
├── sequenceNumber      : long (monoton steigend pro Pipeline)
└── runId               : String (Join-Key zum RunContext)
```

> **marketTime statt Instant.now():** Der Timestamp kommt von der `MarketClock`. In der Simulation ist das die vom Runner gesteuerte SimClock. Kein `Instant.now()` im Snapshot.

> **BAR_APPROXIMATED:** Im Bar-Replay-Modus sind `vwap`, `spreadStats`, `currentBid`, `currentAsk` und `currentSpread` aus Bar-Daten approximiert (VWAP ≈ typicalPrice * volume, Spread = nicht verfuegbar → 0.0). Das Flag signalisiert nachgelagerten Konsumenten, dass diese Felder keine exakte SoT-Qualitaet haben.

> **L2-Daten im Snapshot (Best-Effort):** Da Tick-Queue und L2-Queue getrennt sind und L2 coalesced wird, koennen L2-basierte Felder (`currentBid`, `currentAsk`, `currentSpread`, `spreadStats`) im Snapshot gegenueber dem letzten Tick leicht verzoegert sein. Dies ist **Best-Effort** und architektonisch akzeptabel — L2 dient der Spread-Validierung, nicht der Preisfindung. Das `L2IntegrityGate` arbeitet auf dem L2-Stream direkt und ist von Queue-Gaps nicht betroffen (Gate prueft nur empfangene Updates, nicht fehlende).

### Erzeugungstrigger

Ein Snapshot wird erzeugt bei:
1. **Decision-Bar-Close** (1-Minute-Bar abgeschlossen, MarketClock)
2. **Expliziter Anforderung** (z.B. beim State-Transition der Pipeline)

### Source of Truth

| Datum | Source of Truth | Anmerkung |
|-------|----------------|-----------|
| VWAP | odin-data (VWAP-Akkumulator) | Einzige VWAP-Quelle. Im Bar-Replay-Modus: Approximation, mit Flag |
| Preis/Bid/Ask/Spread | odin-data (Tick-Buffer) | Letzter bekannter Stand. Im Bar-Replay: nur Close verfuegbar |
| KPI-Werte (RSI, ATR, EMA) | odin-brain (KpiEngine) | Nicht im Snapshot, werden von Brain berechnet |

### Lifecycle

```
Buffer-Update → Snapshot erzeugen → EventLog (non-droppable) → Event publizieren → Brain konsumiert → Snapshot wird GC'd
```

Snapshots werden nicht dauerhaft in-memory gehalten. Fuer Post-Mortem-Analyse haelt der `DataPipelineService` die letzten N Snapshots in einem Diagnostics-Ringpuffer (konfigurierbar, Default: 100), abrufbar via REST. **Bei ESCALATE-Events wird der gesamte Ringpuffer-Inhalt automatisch ins EventLog persistiert** (einmalig, fuer Forensik).

---

## 8. Derived Features (odin-data)

Die Data Pipeline berechnet **deterministische, zustandsarme Ableitungen** aus dem Marktdatenstream. Diese dienen als rohe Features fuer die nachgelagerten Schichten (Brain, Rules Engine).

| Feature | Berechnung | Tick-Replay | Bar-Replay |
|---------|-----------|-------------|------------|
| VWAP | Kumulatives (Price x Volume) / Volume | Exakt (Tick-basiert) | Approximiert (typicalPrice * barVolume), `BAR_APPROXIMATED` |
| Spread-Statistiken | Min/Max/Avg aus L2 + Ticks | Exakt | Nicht verfuegbar (0.0), `BAR_APPROXIMATED` |
| Intraday-Extrema | High-of-Day, Low-of-Day | Exakt (Tick-basiert) | Aus Bar-High/Low (kann Micro-Extrema verpassen), `BAR_APPROXIMATED` |
| Volume-Profile | Volumen-Verteilung pro Preislevel (optional, v2) | Verfuegbar | Nicht verfuegbar |

> **Abgrenzung zu odin-brain:** Alles was **Interpretation** ist (Regime-Klassifikation, Pattern-Erkennung, Setup-Signale, Entry/Exit-Bewertung) gehoert in odin-brain. odin-data liefert nur Rohdaten und einfache Ableitungen — keine Signale, keine State-Machines, keine Strategie-Logik.

---

## 9. Bar-Close-Lifecycle und Quiescence

Der Bar-Close ist der zentrale Taktgeber der Pipeline. Folgender Ablauf gilt pro Pipeline bei jedem Decision-Bar-Close:

### Ablauf (pro Pipeline)

1. `MarketEvent`-Stream trifft ein → BarBuilder sammelt Ticks
2. MarketClock erreicht Bar-Boundary → BarBuilder erzeugt Decision-Bar (OHLCV)
3. DQ-Gates validieren die Bar
4. Validierte Bar → RollingDataBuffer (+ Aggregation hoeherer Timeframes)
5. `MarketSnapshotFactory` erzeugt immutable `MarketSnapshot`
6. Snapshot + BarClose werden ins **EventLog geschrieben (non-droppable)**
7. Pipeline meldet **Quiescence** an den Orchestrator (odin-core): „drained bis sequence X / bar t"
8. Orchestrator (Live: sofort weiter; Sim: Bar-Close-Barrier, wartet auf Brain/Execution)
9. `MarketSnapshotEvent` wird an Brain publiziert → Decision-Cycle startet

### Quiescence-Semantik

| Feld | Bedeutung |
|------|-----------|
| `sequenceNumber` | Letztes **verarbeitetes** MarketEvent (nicht nur im Snapshot enthaltenes) |
| `barTime` | Zeitstempel der abgeschlossenen Decision-Bar (MarketClock) |
| `instrumentId` | Instrument dieser Pipeline |
| `runId` | Join-Key zum RunContext |

> **Idempotenz-Anforderung:** Nachgelagerte Konsumenten (Brain, Execution) muessen auf dem Tupel `(runId, instrumentId, barTime)` idempotent sein. Bei Replay oder Recovery darf derselbe Bar-Close nicht zu doppelten Aktionen fuehren.

> **Simulation:** Im Decision-synced-Modus gibt der SimulationRunner erst nach vollstaendigem Abschluss des Decision-Cycle (Brain → Execution → OMS → Global Risk) das naechste Event frei. Die Quiescence-Meldung der Pipeline ist der erste Schritt in dieser Barrier-Kette.

---

## 10. EventLog-Integration (Record/Replay)

Die Data Pipeline schreibt folgende Events ins EventLog:

| Event-Klasse | Volume | Drop-Policy |
|-------------|--------|-------------|
| MarketEvent (Ticks, L2-Updates) | Hoch | **Best-effort** — darf bei Ueberlast gesampelt/gedroppt werden. sequenceNumber bleibt erhalten |
| BarClose (Decision-Bar) | Niedrig | **Non-droppable** — Pflicht fuer Replay |
| MarketSnapshot-Referenz (Snapshot-ID + Key-Felder: marketTime, instrumentId, sequenceNumber, vwap, currentPrice) | Niedrig | **Non-droppable** — Pflicht fuer Reproduzierbarkeit |
| DataQualityEvent (WARN, REJECT_EVENT, ESCALATE) | Niedrig | **Non-droppable** |

> **Replay-Minimum:** Fuer einen vollstaendigen Replay-Lauf werden mindestens die BarClose-Events + DQ-Events benoetigt. Tick-Level-Replay ist optional und abhaengig von der Feed-Variante (Tick-Replay vs. Bar-Replay).

### EventLog-Failure-Strategie (Live-Modus)

Im Live-Modus schreibt die Pipeline **asynchron** ins EventLog (Kap 1). Fuer non-droppable Events gilt folgende Failure-Policy.

> **Spool-Durability: In-Memory.** Der Spool-Buffer ist ein In-Memory-Ringpuffer im Prozess. Bei Prozess-Crash gehen ungespoolte Events verloren. Das ist akzeptabel, weil Crash-Recovery v1 (Kap 0) ohnehin nicht intraday restartbar ist — GTC-Stops schuetzen Positionen, und Post-Mortem-Forensik nutzt das bereits persistierte EventLog (alles vor dem Crash).

| Situation | Reaktion |
|-----------|---------|
| EventLog erreichbar, Queue OK | Normalbetrieb — async write, non-blocking |
| EventLog temporaer langsam | **Lokaler Spool-Buffer** (Bounded, Default: 10.000 Events). Events werden gepuffert und nachgeliefert |
| Spool-Buffer > 80% | `DataQualityEvent(WARN, EVENTLOG_BACKPRESSURE)` an odin-core |
| Spool-Buffer voll | `DataQualityEvent(ESCALATE, EVENTLOG_OVERFLOW)` → odin-core entscheidet (Kill-Switch oder Degraded-Continue ohne EventLog) |

> **Simulation:** Im Sim-Modus schreibt die Pipeline **synchron** ins EventLog (innerhalb der Bar-Close-Barrier). EventLog-Failure ist hier ein harter Fehler — Simulation wird abgebrochen.

---

## 11. Initialisierung (Pre-Market)

Beim Tagesstart laedt die Data Pipeline historische Daten, bevor der Echtzeit-Feed aktiv wird:

### Ablauf

1. **Session-Info abfragen:** RTH-Start/End, Half-Day-Status, Feiertag-Check via `MarketClock.getSessionBoundaries()` fuer das aktuelle Datum und die konfigurierte Exchange
2. **Historical Data Request:** 20 Tage Daily Bars + 5 Tage Intraday-Bars (30-Min) via `MarketDataFeed`
3. **Buffer Prefill:** Daily- und 30-Min-Buffer werden mit historischen Daten gefuellt. **1m/5m/15m-Buffer bleiben leer** — werden ab Live-/Replay-Start bottom-up aufgebaut
4. **VWAP-Reset:** VWAP-Akkumulator auf 0 setzen (Intraday-Berechnung startet bei erster RTH-Bar)
5. **Extrema-Reset:** Intraday-High/Low zuruecksetzen
6. **DQ-Gate Kalibrierung:** Durchschnittliches Volumen und ATR aus historischen Daten berechnen (Referenz fuer Stale-Detection, Crash-Detection)
7. **Feed aktivieren:** Echtzeit-Subscription (Live) oder Replay-Start (Sim)
8. **Warm-Up:** Erste 15 Minuten nach RTH-Start (via `MarketClock.getSessionBoundaries()`, z.B. 09:30–09:45 ET) dienen der Kalibrierung — Intraday-ATR und Volumen-Baseline werden berechnet. **Kein Decision-Cycle waehrend Warm-Up** — Snapshots werden erzeugt, aber Brain ignoriert sie (via Warm-Up-Flag im Snapshot oder Pipeline-State)

### Simulation-Init

Im Simulationsmodus laedt `HistoricalMarketDataFeed` die Daten aus DB oder CSV. Der Init-Ablauf ist identisch — nur die Datenquelle unterscheidet sich. Die MarketClock wird vom SimulationRunner auf den Tagesstart gesetzt (z.B. 07:00 ET fuer Pre-Market).

### Session-Boundaries

| Event | Aktion |
|-------|--------|
| Start-of-Day | Leere Buffers, Prefill mit Historical (30m/Daily), DQ-Kalibrierung, Session-Query via MarketClock |
| End-of-Day | Pipeline stop, letzter Snapshot erzeugen, Flush aller Aggregationen, Quiescence melden |

### Fehlerbehandlung

| Fehler | Reaktion |
|--------|---------|
| Historical Data nicht verfuegbar | Retry 3x mit Backoff. Ohne Historical Data kein Start → Pipeline bleibt in INITIALIZING |
| Feed-Subscription fehlgeschlagen | Retry ueber odin-broker Reconnect. Nach Reconnect: Subscription-Rehydration (Kap 1) |
| Pre-Market-Daten lueckenhaft | Akzeptabel — Pre-Market-Volumen ist oft duenn. DQ-Gates tolerieren Luecken vor RTH |
| MarketClock Session-Info nicht verfuegbar | Pipeline bleibt in INITIALIZING — ohne Session-Zeiten kein sicherer Betrieb |

---

## 12. Thread-Modell (Data Layer)

| Thread | Aufgabe | Scope |
|--------|---------|-------|
| Data-Pipeline-Thread | Feed-Events konsumieren, BarBuilder, DQ-Check, Buffer-Update, Snapshot-Erzeugung | Pro Pipeline |
| DQ-Monitor-Thread | Periodische Stale-Detection (Watchdog, alle 5s MarketClock) | Pro Pipeline (nur Live-Modus) |

> **Trennung von Empfang und Verarbeitung:** Der IB-Receiver-Thread (global, in odin-broker) dispatcht Events auf pipeline-spezifische Queues via IbDispatcher. Der Data-Pipeline-Thread konsumiert aus seiner Queue und verarbeitet sequentiell. Keine Contention auf dem Buffer.

### Queue-Architektur und Backpressure

Die Pipeline verwendet **zwei getrennte Queues** pro Pipeline:

| Queue | Typ | Kapazitaet | Drop-Policy | Begruendung |
|-------|-----|-----------|-------------|-------------|
| **Tick-Queue** | Unbounded (LinkedBlockingQueue) | Kein Limit | **Kein Drop** — Ticks sind Grundlage fuer OHLCV-Korrektheit des BarBuilder | BarBuilder-Integritaet hat Vorrang. Bei 2–3 Instrumenten und 1-Min-Bars ist die Tick-Rate beherrschbar |
| **L2-Queue** | Bounded (ArrayBlockingQueue) | Konfigurierbar (`odin.data.l2-queue-capacity`, Default: 500) | **Coalesced** — nur letzter Stand pro Level behalten. Bei Queue-Voll: aelteste L2-Updates verwerfen | L2 ist supplementaer; Verlust einzelner Updates akzeptabel |

> **Warum Tick-Queue unbounded:** Bei 2–3 gleichzeitigen Instrumenten mit typischen US-Equity-Tick-Raten (100–500 Ticks/s pro Instrument) und 1-Minute-Bar-Zyklen ist die Verarbeitung schnell genug. Wenn dennoch ein Stau entsteht (z.B. durch GC-Pause), wird die Queue temporaer tiefer — der naechste Bar-Close arbeitet den Rueckstand ab.

### Tick-Queue Guardrails

Obwohl die Tick-Queue unbounded ist, werden Monitoring-Metriken und Schwellen definiert, um RAM-Runaway fruehzeitig zu erkennen:

| Metrik | Beschreibung | Schwelle |
|--------|-------------|----------|
| `tickQueueSize` | Aktuelle Anzahl Events in der Queue | WARN bei > 5.000, ESCALATE bei > 20.000 |
| `tickQueueMaxAge` | MarketClock.now() − marketTime des aeltesten Elements | WARN bei > 5s, ESCALATE bei > 15s |

Bei ESCALATE publiziert die Pipeline ein `DataQualityEvent(ESCALATE, TICK_QUEUE_OVERLOAD)` → odin-core entscheidet (Kill-Switch oder Degraded-Continue). **Die Pipeline droppt keine Ticks** — die Entscheidung ueber Degrade/Kill liegt bei odin-core.

---

## 13. Konfiguration

```properties
# odin-data.properties

# BarBuilder
odin.data.decision-bar-timeframe-s=60

# Buffer-Groessen
odin.data.buffer.tick-capacity=500
odin.data.buffer.historical-daily-days=20
odin.data.buffer.historical-intraday-days=5

# DQ-Gate-Schwellenwerte
odin.data.dq.stale-warn-threshold-s=30
odin.data.dq.stale-kill-threshold-s=60
odin.data.dq.crash-price-deviation-percent=5.0
odin.data.dq.crash-volume-multiplier=3.0
odin.data.dq.outlier-price-deviation-percent=5.0
odin.data.dq.l2-integrity-enabled=true
odin.data.dq.l2-max-spread-percent=10.0

# L2
odin.data.l2.enabled=true
odin.data.l2-queue-capacity=500

# Tick-Queue Guardrails
odin.data.tick-queue-warn-size=5000
odin.data.tick-queue-escalate-size=20000
odin.data.tick-queue-warn-age-s=5
odin.data.tick-queue-escalate-age-s=15

# EventLog Spool (Live-Modus, in-memory)
odin.data.eventlog-spool-capacity=10000
odin.data.eventlog-spool-alert-threshold-percent=80

# Diagnostics
odin.data.diagnostics-ringbuffer-size=100

# Warm-Up
odin.data.warmup-duration-min=15
```

---

## 14. Abhaengigkeiten und Schnittstellen

### Konsumiert (Ports aus odin-api)

- `MarketDataFeed` — Marktdaten-Stream (Ticks, L2, Historical Bars)
- `MarketClock` — Zeitreferenz fuer BarBuilder, DQ-Gates, Aggregationen + Session-Info (RTH-Zeiten, Feiertage, Half-Days via integriertem TradingCalendar)
- `EventLog` — Persistierung von BarClose, Snapshot-Referenz, DQ-Events

### Produziert (fuer odin-brain, odin-core)

- `MarketSnapshotEvent` — Immutable Snapshot bei jedem Decision-Bar-Close
- `DataQualityEvent` — DQ-Gate-Verletzungen (WARN, REJECT_EVENT, ESCALATE) + EventLog-Backpressure
- `PipelineQuiescence` — Signal an Orchestrator: Pipeline ist drained bis (sequenceNumber, barTime, instrumentId, runId)

### Nicht in Scope

- KPI-Berechnung (RSI, EMA, ATR) → `odin-brain` (KpiEngine)
- Pattern-State-Machines (Setup A/B) → `odin-brain` (RulesEngine)
- Order-Status-Events → `odin-execution`
- Account-Data → `odin-core`
