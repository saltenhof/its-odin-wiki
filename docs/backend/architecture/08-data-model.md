# ODIN — Kapitel 8: Datenmodell & Persistenz (PostgreSQL)

Version: 0.6 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2. Cross-Review R1+R2. DDD-Modulschnitt: Entities in Domaenen-Module, Drain-Mechanismus neu (Domaenen persistieren direkt, AuditEventDrainer nur fuer EventRecord)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt das Datenmodell und die Persistenz-Schicht von ODIN. Alle handelsrelevanten Daten (Runs, Trades, Fills, LLM-Calls, Decisions, Events) werden in PostgreSQL persistiert — fuer Forensik, Recovery und Langzeit-Analyse.

**Modulzuordnung:** Entities und Repositories leben in ihren fachlichen Domaenen-Modulen (DDD-Modulschnitt, Kap 1):

| Entity-Gruppe | Modul | Package |
|---------------|-------|---------|
| TradingRun, Trade, Fill, ConfigSnapshot | odin-execution | `de.its.odin.execution.persistence` |
| LlmCall, DecisionLog | odin-brain | `de.its.odin.brain.persistence` |
| PipelineState, DailyPerformance | odin-core | `de.its.odin.core.persistence` |
| EventRecord | odin-audit | `de.its.odin.audit.entity` / `de.its.odin.audit.repository` (flach, kein `persistence/` Sub-Package) |

`odin-persistence` liefert nur die Infrastruktur (DataSource, JPA-Setup, Flyway).

**Zentrale Aufgabe:** Persistierung aller handelsrelevanten Daten. **Strukturierte Daten** (Trades, Fills, LLM-Calls etc.) werden von den Domaenen-Modulen direkt ueber eigene Repositories geschrieben. Das **flache Event-Archiv** (EventRecord) wird ueber den `AuditEventDrainer` in odin-audit aus dem EventLog-Spool geschrieben.

**Abgrenzung:**
- Domaenen-Module definieren ihre eigenen Entities und Repositories — kein zentrales Persistence-Modul fuer fachliche Inhalte
- odin-audit ist ausschliesslich fuer das flache Event-Archiv (EventRecord) zustaendig — keine strukturierten Tabellen

---

## 2. Architektur: Zwei Persistierungspfade

### 2.1 Uebersicht

```
Domaenen-Module (OMS, DecisionArbiter, LLM-Clients, LifecycleManager)
       │
       ├── PFAD 1: Direkte Persistierung (strukturierte Daten)
       │   OMS → TradeRepository, FillRepository          (odin-execution)
       │   DecisionArbiter → DecisionLogRepository         (odin-brain)
       │   LLM-Clients → LlmCallRepository                (odin-brain)
       │   LifecycleManager → TradingRunRepository,        (odin-execution)
       │                      PipelineStateRepository,     (odin-core)
       │                      DailyPerformanceRepository   (odin-core)
       │
       └── PFAD 2: EventLog → AuditEventDrainer (flaches Archiv)
           │ emit Events
           v
    ┌──────────────────────────────┐
    │  EventLog (Port, odin-api)   │
    │  Live: bounded In-Memory-Spool (async)
    │  Sim:  synchron (Bar-Close-Barrier)
    └──────────┬───────────────────┘
               │ drain
               v
    ┌──────────────────────────────┐
    │  AuditEventDrainer           │
    │  (odin-audit)                │
    │                              │
    │  NUR EventRecord schreiben   │
    │  (flaches Archiv)            │
    └──────────┬───────────────────┘
               │
               v
    ┌──────────────────────────────┐
    │  PostgreSQL                  │
    │  Schema: odin                │
    └──────────────────────────────┘
```

### 2.2 Pfad 1: Direkte Domaenen-Persistierung

Domaenen-Module persistieren ihre strukturierten Daten **direkt und synchron** ueber eigene Spring Data JPA Repositories. Beispiel:

```
OMS empfaengt BrokerEvent (Fill)
  ├── OMS aktualisiert Trade + Fill ueber TradeRepository/FillRepository (synchron)
  └── OMS emittiert FillEvent an EventLog-Port (fuer EventRecord-Archiv)
```

Die Repositories sind **Singletons** (Spring Beans) und werden Pro-Pipeline-Services per Konstruktor uebergeben (Kap 1, §7).

### 2.3 Pfad 2: AuditEventDrainer (nur EventRecord)

Der `AuditEventDrainer` in odin-audit konsumiert den EventLog-Spool und schreibt **ausschliesslich EventRecords** (flaches Append-Only-Archiv). Er kennt keine strukturierten Tabellen.

- **Live:** Der AuditEventDrainer pollt den EventLog-Spool periodisch (konfigurierbar, Default: 500ms). EventRecords werden batch-weise geschrieben
- **Simulation:** Synchroner Drain innerhalb der Bar-Close-Barrier (Kap 0, §3.3). Jedes EventRecord wird sofort persistiert — deterministisches Verhalten garantiert
- **Idempotenz:** Jeder Event hat eine eindeutige `eventId` (UUID, vom Emitter vergeben). Duplicate-Insert wird per Unique Constraint abgefangen (kein Fehler, kein Re-Processing)

> **Kein PersistenceEventDrainer mehr.** Die fruhere Architektur hatte einen zentralen Drainer der sowohl EventRecords als auch strukturierte Tabellen befuellte. Im DDD-Modulschnitt persistieren die Domaenen ihre strukturierten Daten direkt — der AuditEventDrainer ist nur noch fuer das flache Event-Archiv zustaendig.

---

## 3. Entity-Modell (9 Entities)

### Entity-Modulzuordnung

| Entity | Modul | Begruendung |
|--------|-------|------------|
| TradingRunEntity | odin-execution | finalPnl, tradeCount, contractCurrency = Execution-Semantik |
| TradeEntity | odin-execution | Entry-bis-Exit Round-Trip, OMS-verwaltet |
| FillEntity | odin-execution | Broker-Ausfuehrung, OMS-verarbeitet |
| ConfigSnapshotEntity | odin-execution | Teil des TradingRun-Kontexts (Reproduzierbarkeit) |
| DecisionLogEntity | odin-brain | Decision-Cycle = DecisionArbiter-Domaene |
| LlmCallEntity | odin-brain | LLM-Protokollierung + Cache-Source = brain-Domaene |
| EventRecordEntity | odin-audit | Flaches Append-Only-Archiv, Forensik/Replay |
| PipelineStateEntity | odin-core | State-Recovery beim Startup, Lifecycle-Domaene |
| DailyPerformanceEntity | odin-core | Tages-Aggregat ueber alle Pipelines |

### ER-Uebersicht

```
┌────────────────────┐
│  TradingRun         │
│                    │
│  runId (PK)        │────< alle Entities referenzieren runId
│  mode              │
│  instrumentId      │
│  exchange          │
│  tradingDate       │
│  ...               │
└────────┬───────────┘
         │
         │ 1:N
         v
┌────────────────────┐     ┌──────────────────────┐
│  Trade             │     │  Fill                │
│                    │     │                      │
│  tradeId (PK)      │────<│  fillId (PK)         │
│  runId (FK)        │     │  tradeId (FK)        │
│  instrumentId      │     │  runId (FK)          │
│  ...               │     │  ...                 │
└────────────────────┘     └──────────────────────┘

┌────────────────────┐     ┌──────────────────────┐
│  LlmCall           │     │  DecisionLog         │
│                    │     │                      │
│  callId (PK)       │     │  decisionId (PK)     │
│  runId (FK)        │     │  runId (FK)          │
│  ...               │     │  ...                 │
└────────────────────┘     └──────────────────────┘

┌────────────────────┐
│  EventRecord       │
│  (Append-Only)     │
│                    │
│  eventId (PK)      │
│  runId (FK)        │
│  eventType         │
│  payload (JSONB)   │
│  ...               │
└────────────────────┘
```

**Design-Prinzip:** `EventRecord` ist das flache, vollstaendige Archiv aller Events — **Source of Truth innerhalb seiner Retention** (§8). Nach Ablauf der EventRecord-Retention sind die strukturierten Tabellen (Trade, Fill, LlmCall, DecisionLog) massgeblich, da sie laengere Retention haben.

**Konsistenzmodell:** Strukturierte Daten (Pfad 1) werden synchron von den Domaenen-Modulen geschrieben. EventRecords (Pfad 2) werden im **Live-Modus asynchron** (eventual consistent zum Zeitpunkt des Drain-Batches), im **Sim-Modus synchron** (innerhalb der Bar-Close-Barrier). Im Live-Modus kann ein kurzes Fenster entstehen, in dem strukturierte Daten bereits geschrieben, das zugehoerige EventRecord aber noch im Spool liegt. Fuer Forensik und Replay ist das unkritisch — die `sequenceNumber` stellt die korrekte Reihenfolge sicher.

---

## 4. Entity-Details

### TradingRun

Ein TradingRun bildet einen `RunContext` (Kap 0, §3) **1:1** ab: eine Ausfuehrung (Live oder Simulation) fuer ein Instrument an einem Handelstag. `instrumentId` ist Teil des RunContext (jede Pipeline erzeugt ihren eigenen RunContext mit dem zugeordneten Instrument).

> **Zeitzone:** `tradingDate` ist der Handelstag in **Exchange-Timezone** (z.B. `America/New_York` fuer US-Aktien, Kap 0 §3.2: Currency/Exchange-Awareness). Nicht Server-TZ.

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| runId | UUID (PK) | Aus RunContext.runId — Join-Key fuer alle Entities |
| mode | Enum | LIVE, SIMULATION |
| instrumentId | String | Gehandeltes Instrument (z.B. "AAPL"). Dient als Pipeline-Identifier (1:1 in v1) |
| exchange | String | Exchange-Code (z.B. "NYSE") |
| tradingDate | LocalDate | Handelstag (Exchange-TZ) |
| promptVersion | String | Semantische Version des LLM-Prompts (aus RunContext) |
| modelId | String | LLM-Modell-ID (aus RunContext) |
| configSnapshotId | String | ID des Konfigurations-Snapshots (Reproduzierbarkeit) |
| startTime | Instant | Run-Start (MarketClock-Timestamp) |
| endTime | Instant | Run-Ende (MarketClock-Timestamp, null wenn noch aktiv) |
| finalPnl | BigDecimal | Tages-P&L (in Contract-Waehrung). Null bis Run abgeschlossen |
| tradeCount | int | Anzahl abgeschlossener Round-Trips |
| status | Enum | ACTIVE, COMPLETED, DAY_STOPPED, ERROR |
| contractCurrency | String | Waehrung des gehandelten Instruments (z.B. "USD") |
| batchId | String (nullable) | Optionale Batch-Kennung fuer Sim-Laeufe (Backtest-Serie). Null bei Live-Runs. Ermoeglicht Gruppierung und Vergleich von Sim-Laeufen |

### Trade

Ein Trade umfasst Entry bis Exit (abgeschlossene Position). Wird bei Entry angelegt (noch ohne exitTime/exitPrice/realizedPnl) und bei Exit komplettiert.

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| tradeId | UUID (PK) | Eindeutige Trade-ID |
| runId | UUID (FK) | Zugehoeriger TradingRun |
| instrumentId | String | Symbol |
| direction | Enum | LONG (v1: nur Long) |
| entryTime | Instant | MarketClock-Timestamp des ersten Entry-Fills |
| exitTime | Instant | MarketClock-Timestamp des letzten Exit-Fills (null wenn offen) |
| entryPrice | BigDecimal | Durchschnittlicher Entry-Preis (mengengewichtet) |
| exitPrice | BigDecimal | Durchschnittlicher Exit-Preis (null wenn offen) |
| quantity | int | Gesamte Stueckzahl |
| realizedPnl | BigDecimal | Realisierter P&L inkl. Kosten (null wenn offen) |
| grossPnl | BigDecimal | Realisierter P&L vor Kosten (null wenn offen) |
| totalCost | BigDecimal | Summe aller Transaktionskosten (Commission + Spread + FX, Kap 7 §10) |
| regime | Enum | Regime bei Entry (aus TradeIntent) |
| quantScore | double | Quant-Score bei Entry (aus TradeIntent) |
| llmCallId | UUID (nullable) | Referenz auf den Entry-relevanten LlmCall (scalar ID, keine JPA-Relation — Cross-Modul-Grenze execution→brain) |
| stopLevel | BigDecimal | Initiales Stop-Level (aus TradeIntent) |
| riskRewardRatio | double | Berechnetes R/R bei Entry (netto, nach Kosten, Kap 7 §3) |
| exitReason | Enum | STOP_LOSS, TRAILING_STOP, TARGET, FORCED_CLOSE, KILL_SWITCH, SCALING_COMPLETE |
| tranchenVariant | Enum | COMPACT_3T, STANDARD_4T, FULL_5T (Kap 7, §5) |

### Fill

Einzelne Order-Ausfuehrung. Ein Trade hat mehrere Fills (Entry, Tranchen-Exits, Stop).

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| fillId | UUID (PK) | |
| tradeId | UUID (FK) | Zugehoeriger Trade |
| runId | UUID (FK) | Zugehoeriger TradingRun (denormalisiert fuer Queries) |
| instrumentId | String | Symbol (denormalisiert) |
| brokerOrderId | String | Broker-seitige Order-ID (von BrokerGateway zurueckgegeben bei submitOrder). String fuer Broker-Agnostik (IB liefert int, andere Broker ggf. alphanumerisch). Naming: „brokerOrderId" zur Abgrenzung von clientOrderId (ODIN-intern). Entspricht dem Feld `orderId` in Kap 7 §5 EventLog-Schema — Rename auf `brokerOrderId` ist die Persistence-Konvention zur Disambiguation |
| brokerExecId | String | Broker-seitige Execution-ID (natuerlicher Key fuer Idempotenz). Broker-agnostisch — IB liefert `execId`, andere Broker ihren jeweiligen Key |
| clientOrderId | String | ODIN-Client-Order-ID (Pipeline-Prefix + Sequence, Kap 3) |
| fillPrice | BigDecimal | Tatsaechlicher Fill-Preis |
| quantity | int | Gefuellte Stueckzahl |
| side | Enum | BUY, SELL |
| slippageBps | double | Slippage in Basispunkten (Fill-Preis vs. Intent-Preis, Kap 7 §9) |
| marketClockTimestamp | Instant | Fill-Zeitpunkt (MarketClock) |
| fillType | Enum | ENTRY, EXIT_TARGET, EXIT_STOP, EXIT_TRAILING, EXIT_FORCED, EXIT_KILL_SWITCH |
| costBreakdown | JSONB (nullable) | Aufschluesselung: `{commission, spreadCost, fxCost}` (Kap 7, §10). Null bei Sim-Fills ohne Kostenberechnung. Trade.totalCost = Summe aller Fill-costBreakdowns |

### LlmCall

Jeder LLM-Aufruf wird protokolliert (Input + Output). Dient als Cache-Source fuer Simulation (CachedAnalyst, Kap 5 §3).

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| callId | UUID (PK) | |
| runId | UUID (FK) | |
| instrumentId | String | |
| provider | Enum | CLAUDE, OPENAI |
| model | String | Modell-ID (z.B. "claude-sonnet-4-5-20250929") |
| promptVersion | String | Semantische Version des Prompts |
| inputHash | String | SHA-256 Hash des kanonisierten Inputs (Cache-Key, Kap 5 §3) |
| inputPayload | JSONB | Vollstaendiger Input |
| outputPayload | JSONB | Vollstaendiger Output (parsed) |
| regime | Enum | Extrahiertes Regime |
| regimeConfidence | double | |
| requestMarketTime | Instant | MarketClock-Timestamp bei Request-Absendung |
| responseMarketTime | Instant | MarketClock-Timestamp bei Response-Empfang |
| latencyMs | int | Wanduhr-Latenz (fuer Performance-Monitoring) |
| validationResult | Enum | VALID, SCHEMA_ERROR, PLAUSIBILITY_ERROR, CONSISTENCY_ERROR |

### DecisionLog

Jeder Decision-Cycle wird protokolliert — auch abgelehnte (REJECT). Ermoeglicht Analyse, warum Entries nicht stattfanden.

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| decisionId | UUID (PK) | |
| runId | UUID (FK) | |
| instrumentId | String | |
| barCloseTime | Instant | MarketClock-Timestamp des Decision-Bars (Kap 6, §2) |
| tradeId | UUID (nullable) | Nur gesetzt wenn Trade ausgefuehrt (scalar ID, keine JPA-Relation — Cross-Modul-Grenze brain→execution) |
| intentType | Enum | ENTRY, EXIT, NO_ACTION |
| pipelineState | Enum | Pipeline-State bei Decision (STARTUP, WARMUP, OBSERVING, POSITIONED, PENDING_FILL, HALTED, DAY_STOPPED — vollstaendige FSM aus Kap 6). Decision-Cycles laufen nur in Loop-active States (WARMUP, OBSERVING, POSITIONED, PENDING_FILL), aber der State wird fuer alle Eintraege erfasst |
| quantScore | double | |
| regimeAtDecision | Enum | Regime zum Entscheidungszeitpunkt |
| regimeConfidence | double | LLM-regimeConfidence (0.0 wenn kein/stale LLM-Output, Kap 5 §4) |
| riskGateResult | JSONB | Risk-Gate-Pruefung (Sizing, R/R, Budget, Kap 7 §3) |
| finalDecision | Enum | EXECUTE, REJECT |
| rejectReason | Enum | Ablehnungsgrund (QUANT_*, WARMUP_*, LLM_*, REGIME_* aus Kap 6; RISK_* aus Kap 7). Null wenn EXECUTE |
| emitter | String | Modul das den Reject emittiert hat (RULES_ENGINE, RISK_GATE). Kap 6 §11, Kap 7 §3 |
| marketClockTimestamp | Instant | Zeitpunkt der Decision |

### EventRecord (Append-Only Archiv)

Flache Persistierung aller EventLog-Events. Source of Truth fuer Forensik und Replay.

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| eventId | UUID (PK) | Vom Emitter vergeben (Idempotenz-Key). Regel: derselbe logische Event behaelt bei Retry dieselbe eventId. Emitter erzeugt die UUID einmalig beim ersten Emit, nicht bei jedem Versuch |
| runId | UUID (FK) | |
| instrumentId | String | |
| eventType | String | Event-Typ (z.B. FILL, ORDER_STATUS, REJECT, LLM_CALL, STATE_CHANGE, RECONCILIATION_COMPLETED, MARKET_ORDER_USED) |
| emitter | String | Quellenmodul (OMS, RULES_ENGINE, LLM_ANALYST, CORE) |
| sequenceNumber | long | Monotone Sequenznummer pro Pipeline (instrumentId + runId). Primaerer Ordering-Key fuer Replay. Vom Emitter vergeben |
| clientOrderId | String (nullable) | ODIN-Client-Order-ID (gesetzt bei Order-bezogenen Events: FILL, ORDER_STATUS, REJECT, MARKET_ORDER_USED). Null bei LLM_CALL, STATE_CHANGE etc. |
| brokerOrderId | String (nullable) | Broker-seitige Order-ID (gesetzt bei Order-bezogenen Events). String fuer Broker-Agnostik. Null bei Events ohne Order-Kontext |
| marketClockTimestamp | Instant | MarketClock-Timestamp des Events |
| wallClockTimestamp | Instant | Server-Wanduhr (fuer Latenz-Debugging, nicht fuer Geschaeftslogik) |
| payload | JSONB | Vollstaendige Event-Details (strukturiert, schema-frei) |

> **Kein Hash-Chain.** v0.1 hatte eine AuditEntry-Entity mit Tamper-Proof Hash-Chain (SHA-256 Verkettung). **Entfernt** gemaess Lean-Prinzip: ODIN ist ein Single-User-System auf lokaler Infrastruktur. Hash-Chain-Audit ist Over-Engineering ohne nachgewiesenen Threat. Die Append-Only-Semantik von EventRecord (kein UPDATE, kein DELETE in der Applikation) genuegt fuer v1-Forensik. Falls spaeter Compliance-Anforderungen entstehen, kann Hash-Chain nachgeruestet werden.

> **Kein HMAC-Signing.** v0.1 hatte Trade-Intent-Signierung per HMAC-SHA256. **Entfernt** aus demselben Grund: kein Threat-Model das Signing rechtfertigt. TradeIntents werden prozessintern erzeugt und konsumiert — keine externe Angriffsflaeche.

---

## 5. Indizierung und Constraints

### Indizes

| Entity | Index | Begruendung |
|--------|-------|-------------|
| TradingRun | (tradingDate, instrumentId) | Tagesbasierte Abfragen |
| TradingRun | (mode, tradingDate) | Live/Sim-Filterung |
| TradingRun | (batchId) WHERE batchId IS NOT NULL | Sim-Batch-Gruppierung |
| Trade | (runId) | Alle Trades eines Runs |
| Trade | (instrumentId, entryTime) | Instrumentbasierte Zeitreihen |
| Trade | (exitReason) | Analyse nach Exit-Grund |
| Fill | (tradeId) | Fills eines Trades |
| Fill | (runId, instrumentId) | Run-basierte Fill-Analyse |
| Fill | (runId, clientOrderId) | Reconciliation-Matching (Kap 7, §6) — runId-scoped fuer effiziente Suche |
| LlmCall | (inputHash, promptVersion, provider, model) | Cache-Lookup fuer CachedAnalyst (Kap 5 §3) |
| LlmCall | (runId, requestMarketTime) | Chronologische LLM-Auswertung |
| DecisionLog | (runId, barCloseTime) | Chronologische Decision-Analyse |
| DecisionLog | (rejectReason) | Analyse nach Reject-Grund |
| EventRecord | (runId, instrumentId, sequenceNumber) | Deterministisches Event-Replay (primaer) |
| EventRecord | (runId, marketClockTimestamp) | Chronologisches Event-Replay (sekundaer) |
| EventRecord | (eventType) | Filterung nach Event-Typ |
| EventRecord | (clientOrderId) WHERE clientOrderId IS NOT NULL | Order-bezogene Event-Suche (Reconciliation, Debugging) |

### Unique Constraints (Idempotenz)

| Entity | Unique Key | Zweck |
|--------|-----------|-------|
| TradingRun | (runId) | PK, aus RunContext |
| Fill | (brokerExecId, runId) | Broker-Execution-ID als natuerlicher Key. Verhindert Duplicate-Insert bei Restart/Reconciliation |
| LlmCall | (inputHash, promptVersion, provider, model) | Cache-Key fuer CachedAnalyst Replay (Kap 5 §3) |
| EventRecord | (eventId) | PK, vom Emitter vergeben. Idempotenter Drain |

---

## 6. State Recovery

Beim Neustart (Live) oder Tagesstart laedt jede Pipeline ihren letzten persistierten Zustand. Die Recovery nutzt die Reconciliation aus Kap 7, §6.

| Schritt | Aktion |
|---------|--------|
| 1 | Letzten `TradingRun` fuer das Instrument laden (heute, mode=LIVE). Filter auf ACTIVE **oder** ERROR (fuer Crash-Erkennung) |
| 2 | Offene Trades (exitTime IS NULL) laden |
| 3 | Letzten Pipeline-State aus `PipelineStateRepository` laden (odin-core). Fallback: EventRecords (eventType=STATE_CHANGE) fuer Forensik |
| 4 | Broker-Reconciliation ausfuehren (Kap 7, §6: requestOpenOrders + requestPositions) |
| 5 | Broker-State ist Source of Truth — OMS korrigiert internen State (Kap 7, §6) |
| 6 | Pipeline in korrekten State versetzen: POSITIONED wenn offene Position, OBSERVING wenn keine |

> **Crash-Erkennung (Safe-Mode):** Wird in Schritt 1 ein TradingRun fuer den aktuellen Handelstag mit `outcome=ERROR` **oder** `endTime IS NULL` gefunden, liegt ein Intraday-Crash vor (oder ein vorheriger Crash wurde bereits erkannt). Der abgebrochene Run erhaelt Outcome `ERROR` und `endTime` wird auf den aktuellen Zeitpunkt gesetzt (falls noch NULL). Der Agent geht in den **Safe-Mode** (Kap 10, §5): kein Trading fuer den Rest des Tages, nur Dashboard/Reconciliation/Forensik. Die Broker-Reconciliation (Schritte 4-5) wird ausgefuehrt wenn IB Gateway erreichbar ist, damit der Operator den aktuellen Positions-Status im Dashboard sehen kann. Ist IB Gateway nicht erreichbar (z.B. manueller Login erforderlich), startet ODIN im DEGRADED Safe-Mode: Dashboard/Forensik verfuegbar, Reconciliation pending. **Mehrfach-Restart-Sicherheit:** Da die Erkennung auch ueber `outcome=ERROR` laeuft (nicht nur `endTime IS NULL`), funktioniert der Safe-Mode auch bei wiederholtem WinSW-Restart am selben Tag.

> **Simulation:** Kein Recovery noetig. Jeder simulierte Tag startet mit leerem State (NO_POSITION). Der SimulationRunner (Kap 0, §3.3) erzeugt pro Tag einen neuen RunContext/TradingRun.

---

## 7. Simulation-Semantik

### Identischer Codepfad

Die Persistence-Schicht verhaelt sich in Live und Simulation **identisch** (Kap 0, §3.1). Unterschiede:

| Aspekt | Live | Simulation |
|--------|------|------------|
| Drain-Modus | Asynchron (Polling, `drainIntervalMs`) | Synchron (innerhalb Bar-Close-Barrier) |
| TradingRun.mode | LIVE | SIMULATION |
| State Recovery | Ja (Startup-Reconciliation) | Nein (frischer State pro Tag) |
| Timestamps | SystemMarketClock (Echtzeit) | SimClock (vom Runner gesteuert) |

### RunContext als Schluessel

- `runId` identifiziert jeden Run eindeutig (Live und Sim)
- Sim-Runs mit unterschiedlichen Configs/Prompts erzeugen verschiedene runIds → vergleichbar
- `configSnapshotId` und `promptVersion` in TradingRun ermoeglichen Reproduzierbarkeit

### Sim-Datenvolumen

Bei Backtests (viele simulierte Tage x mehrere Instrumente) entstehen signifikante Datenmengen. Retention-Policy (§8) steuert die Bereinigung. Sim-Runs einer Backtest-Serie werden ueber `TradingRun.batchId` gruppiert — ermoeglicht gezielte Abfragen und kaskadierende Loeschung ganzer Batches.

---

## 8. Retention und Archivierung

| Datentyp | Retention | Aktion nach Ablauf |
|----------|----------|--------------------|
| TradingRun (LIVE) | 5 Jahre (konfigurierbar) | In Archiv-Schema verschieben |
| TradingRun (SIMULATION) | 90 Tage (konfigurierbar) | Loeschen (inkl. abhaengiger Entities) |
| Trade, Fill | Wie zugehoeriger TradingRun | Kaskadiert |
| LlmCall (Payloads) | 1 Jahr | inputPayload/outputPayload auf NULL setzen, Metadaten behalten |
| DecisionLog | 2 Jahre | Loeschen |
| EventRecord (LIVE) | 1 Jahr | Loeschen (strukturierte Tabellen behalten laenger) |
| EventRecord (SIMULATION) | 90 Tage | Loeschen |

> **Keine Archiv-Komplexitaet in v1.** "In Archiv-Schema verschieben" ist eine geplante Massnahme. Fuer v1 genuegt Retention mit Loeschung. Archivierung wird bei Bedarf nachgeruestet.

---

## 9. Konfiguration

```properties
# odin-persistence.properties (Infrastruktur)

# DataSource (Overrides per ENV)
odin.persistence.datasource.url=jdbc:postgresql://localhost:5432/odindb
odin.persistence.datasource.username=${ODIN_DB_USER}
odin.persistence.datasource.password=${ODIN_DB_PASSWORD}

# Retention (Tage) — gilt moduluebergreifend, zentral konfiguriert
odin.persistence.retention.live-run-days=1825
odin.persistence.retention.sim-run-days=90
odin.persistence.retention.llm-payload-days=365
odin.persistence.retention.decision-log-days=730
```

```properties
# odin-audit.properties

# Drain (AuditEventDrainer)
odin.audit.drain.interval-ms=500
odin.audit.drain.batch-size=100

# Retention (EventRecord)
odin.audit.retention.event-record-live-days=365
odin.audit.retention.event-record-sim-days=90
```

---

## 10. Abhaengigkeiten und Schnittstellen

### Persistierende Module und ihre Repositories

| Modul | Repositories | Primaere Konsumenten |
|-------|-------------|---------------------|
| odin-execution | TradingRunRepository, TradeRepository, FillRepository, ConfigSnapshotRepository | OMS, LifecycleManager, State-Recovery |
| odin-brain | LlmCallRepository, DecisionLogRepository | LLM-Clients, DecisionArbiter, CachedAnalyst (Cache-Lookup) |
| odin-core | PipelineStateRepository, DailyPerformanceRepository | LifecycleManager, State-Recovery |
| odin-audit | EventRecordRepository | AuditEventDrainer, Forensik/Replay |

### Konsumiert (pro Modul)

- **odin-audit:** `EventLog` (Port aus odin-api) — AuditEventDrainer konsumiert den Event-Spool
- **odin-execution:** `RunContext` (aus odin-api) — runId, mode, tradingDate fuer TradingRun-Erzeugung. BrokerEvents von BrokerGateway-Port
- **odin-brain:** `RunContext` fuer LlmCall- und DecisionLog-Kontext
- **odin-core:** Alle Fachmodule fuer Orchestrierung

### Bereitgestellt (Queries)

- State-Recovery-Queries: letzter aktiver Run (odin-execution), offene Trades (odin-execution), letzter Pipeline-State (odin-core)
- LLM-Cache-Queries: Lookup per inputHash + promptVersion + provider (odin-brain, fuer CachedAnalyst Kap 5 §3)
- Forensik: EventRecords chronologisch (odin-audit)

### Nicht in Scope

- Event-Erzeugung → jeweilige Module (OMS, Rules, LLM, Core)
- EventLog-Port-Implementierung → odin-audit
- Broker-Reconciliation → odin-execution (Kap 7, §6)
