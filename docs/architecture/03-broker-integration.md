# ODIN — Kapitel 3: Broker-Anbindung (IB Gateway, TWS API 10.40)

Version: 0.3 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2 — Sim-Account-Semantik, EventLog-Drop-Policy, Idempotenz-Matching, Reconnect-Orchestrierung

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt die Broker-Anbindung von ODIN an Interactive Brokers (IB) ueber das IB Gateway und die TWS API 10.40. Das `odin-broker`-Modul ist der **einzige Beruehrungspunkt** mit der Broker-Infrastruktur — kein anderes Modul importiert TWS-API-Klassen. odin-broker implementiert zwei der fuenf zentralen Ports: `MarketDataFeed` und `BrokerGateway`.

**Modulzuordnung:** `odin-broker` (Package: `de.odin.broker`)

**Port-Implementierungen (aus odin-api):**

| Port | Live-Implementierung | Sim-Implementierung |
|------|---------------------|---------------------|
| `MarketDataFeed` | `IbMarketDataFeed` (pro Pipeline, Facade ueber IbSession) | — (in odin-data: `HistoricalMarketDataFeed`) |
| `BrokerGateway` | `IbBrokerGateway` (pro Pipeline, Facade ueber IbSession) | `BrokerSimulator` (pro Pipeline, in odin-broker) |

---

## 2. Systemkontext

```
┌──────────────────────────────────────────────────────────────┐
│                          ODIN JVM                            │
│                                                              │
│  ┌─────────────────── odin-broker ────────────────────────┐  │
│  │                                                        │  │
│  │  ┌─── IbSession (Singleton) ───────────────────────┐   │  │
│  │  │ EClientSocket │ Receiver │ Sender │ Heartbeat   │   │  │
│  │  │            IbDispatcher (ReqId → Pipeline)      │   │  │
│  │  │            ReqIdAllocator + RequestRegistry     │   │  │
│  │  │            Subscription-Ownership (Ref-Count)   │   │  │    ┌────────────┐
│  │  └─────────┬──────────────────────────┬────────────┘   │  │    │ IB Gateway │
│  │            │                          │                │──────>│ localhost   │
│  │    ┌───────┴────────┐      ┌──────────┴──────────┐     │  │    │ TWS API    │
│  │    │IbMarketDataFeed│      │  IbBrokerGateway    │     │  │    │ 10.40      │
│  │    │(per Pipeline)  │      │  (per Pipeline)     │     │  │    │ (Socket)   │
│  │    │: MarketDataFeed│      │  : BrokerGateway    │     │  │    └────────────┘
│  │    └───────┬────────┘      └──────────┬──────────┘     │  │
│  │            │                          │                │  │
│  │  ┌─────────────────────────────────────────────────┐   │  │
│  │  │  BrokerSimulator (per Pipeline, Sim-Modus)      │   │  │
│  │  │  : BrokerGateway                                │   │  │
│  │  └─────────────────────────────────────────────────┘   │  │
│  │                                                        │  │
│  │  ┌─ IbAccountAdapter (Account/Positions) ─────────┐    │  │
│  │  └────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
│         │ odin-api Ports                                     │
│         v                                                    │
│  odin-data / odin-execution / odin-core                      │
└──────────────────────────────────────────────────────────────┘
```

### Kommunikation mit IB Gateway

| Aspekt | Spezifikation |
|--------|---------------|
| Transport | TCP-Socket (localhost, konfigurierbar) |
| Protokoll | TWS API 10.40 (proprietaeres Binaerprotokoll) |
| Richtung | Bidirektional (Subscriptions/Orders senden, Callbacks empfangen) |
| Multiplexing | **Eine** Socket-Verbindung fuer alle Pipelines. Demultiplexing ueber Request-IDs via IbDispatcher |
| Thread-Safety | EClientSocket ist **nicht** thread-safe → serialisierter Zugriff ueber Sender-Thread |

---

## 3. IbSession (Singleton)

`IbSession` ist das **zentrale Singleton** in odin-broker. Es owned die eine Socket-Verbindung zum IB Gateway und stellt die Infrastruktur bereit, auf der die per-Pipeline Facades aufsetzen.

### Verantwortlichkeiten

| Verantwortung | Details |
|---------------|---------|
| **Socket-Ownership** | Genau eine `EClientSocket`-Instanz. Verbindet zu `gateway-host:gateway-port` (Default: localhost:4002) |
| **Client-ID** | Konfigurierbar (`odin.broker.client-id`). Muss pro IB-Gateway-Session eindeutig sein |
| **Heartbeat** | Alle 30s via **lokalen monotonen Timer** (nicht `reqCurrentTime()`). Prueft, ob in den letzten N Sekunden ein valider IB-Callback empfangen wurde. MarketClock ist Business-Zeitquelle — Heartbeat ist Infrastruktur |
| **Reconnect** | Automatisch bei Verbindungsverlust. Bounded: max. 5 Versuche, Backoff 5s/10s/20s/40s/60s |
| **IbDispatcher** | Demultiplexing aller eingehenden TWS-API-Callbacks auf pipeline-spezifische Queues |
| **ReqIdAllocator** | Globaler `AtomicLong`. Monoton steigende, eindeutige Request-IDs. Seed aus `nextValidId()`-Callback bei Verbindungsaufbau |
| **RequestRegistry** | `ConcurrentMap<Integer, RequestContext>` — ordnet jede Request-ID dem Tuple (pipelineId, feedType, instrumentId) zu |
| **Subscription-Ownership** | IbSession owned den IB-Subscription-Lifecycle. Facades requesten/deregistrieren, IbSession setzt um mit **Referenzzaehlung** |
| **Outbound-Queue** | Alle ausgehenden EClientSocket-Aufrufe werden in eine Queue geschrieben und vom Sender-Thread sequentiell ausgefuehrt |

### Lifecycle

| Phase | Aktion |
|-------|--------|
| Startup | `IbSession.connect()` → Socket oeffnen, `nextValidId()`-Callback abwarten (Seed fuer ReqIdAllocator + Order-IDs), Account-Subscribe |
| Running | Heartbeat-Monitor, Reconnect bei Verlust, Dispatch eingehender Callbacks |
| Kill-Switch | `IbSession` **loest keinen Kill-Switch selbst aus**. Bei Heartbeat-Failure oder erschoepften Reconnect-Versuchen publiziert es ein `BrokerConnectivityEvent(ESCALATE, reason)` → odin-core (KillSwitchService) entscheidet |
| Shutdown | Alle Subscriptions cancellen, Socket schliessen, Threads beenden |

> **Kill-Switch-Ownership (Kap 0/2 konsistent):** IbSession eskaliert — odin-core entscheidet. Kein Modul ausser odin-core darf den Kill-Switch ausloesen.

---

## 4. Per-Pipeline Facades

### IbMarketDataFeed (implements `MarketDataFeed`)

Jede Pipeline erhaelt eine eigene `IbMarketDataFeed`-Instanz als Facade ueber die Singleton-IbSession.

| Aspekt | Details |
|--------|---------|
| Scope | Pro Pipeline instanziiert |
| Instrument | Genau ein Instrument pro Facade |
| Subscriptions | Tick (reqMktData), L2 (reqMktDepth), Historical (reqHistoricalData), RealTimeBars (reqRealTimeBars) |
| Sequence Number | Die Facade vergibt monotone `sequenceNumber`s fuer alle Events dieses Instruments (Tick, L2, Bar) — konsistent mit Kap 2 Sequence-Contract |
| Event-Transformation | IB-Callbacks → `MarketEvent`-Records (odin-api) mit `marketTime` (aus IB-Timestamp, konvertiert in Exchange-TZ), `instrumentId`, `sequenceNumber` |
| MarketClock | `marketTime` wird aus IB-Timestamps abgeleitet (Exchange-TZ). Fuer Business-Logik ist MarketClock die Zeitquelle (Kap 0) |

> **Subscription-Lifecycle:** Die Facade registriert Subscriptions bei IbSession. IbSession fuehrt Referenzzaehlung — bei Pipeline-Shutdown wird deregistriert, erst wenn Ref=0 wird die IB-Subscription tatsaechlich cancelled (TWS API `cancelMktData` etc.).

### IbBrokerGateway (implements `BrokerGateway`)

Jede Pipeline erhaelt eine eigene `IbBrokerGateway`-Instanz als Facade ueber die Singleton-IbSession.

| Aspekt | Details |
|--------|---------|
| Scope | Pro Pipeline instanziiert |
| clientOrderId | Format: `{pipelineId}-{date}-{sequence}` (Pipeline-Prefix garantiert Isolation) |
| Order-ID (IB) | Allokation via IbSession.ReqIdAllocator (globaler AtomicLong, Seed aus `nextValidId()`) |
| Operationen | `submitOrder(TradeIntent)`, `cancelOrder(orderId)`, `modifyOrder(orderId, TradeIntent)` |
| Events | Empfaengt `BrokerEvent`s (Accepted, Rejected, PartiallyFilled, Filled, Cancelled) via IbDispatcher, gefiltert auf eigene clientOrderIds |
| GTC-Stops | Stops werden als **GTC (Good-Till-Cancel)** beim Broker platziert — Crash-Recovery v1 (Kap 0): GTC-Stops schuetzen Positionen auch wenn ODIN abstuerzt |

---

## 5. BrokerGateway Event-Modell und Idempotenz

### Event-Typen (definiert in odin-api)

```
BrokerEvent (Record, immutable)
├── eventType        : BrokerEventType (ACCEPTED, REJECTED, PARTIALLY_FILLED, FILLED, CANCELLED)
├── clientOrderId    : String (Idempotenz-Key)
├── orderId          : long (IB-intern, bzw. Sim-intern)
├── instrumentId     : String
├── filledQuantity   : int (bei FILLED/PARTIALLY_FILLED)
├── fillPrice        : double (bei FILLED/PARTIALLY_FILLED)
├── commission       : double (bei FILLED, geschaetzt)
├── reason           : String (bei REJECTED/CANCELLED)
├── marketTime       : Instant (MarketClock-Zeitpunkt)
├── runId            : String (Join-Key)
└── sequenceNumber   : long (monoton pro Pipeline, Ordering-Garantie)
```

### Idempotenz-Contract

| Regel | Spezifikation |
|-------|---------------|
| `clientOrderId` (ODIN) | Eindeutig pro Run: `{pipelineId}-{tradingDate}-{sequence}`. Bei Retry/Reconnect wird dieselbe ID verwendet. **Primaerer Idempotenz-Key** |
| `orderId` (IB) | Von IB bei `placeOrder()` zugewiesen. Monoton steigend, session-spezifisch |
| `permId` (IB) | Permanente, session-uebergreifende Order-ID. **Autoritativer Key fuer Reconnect-Matching** |
| `execId` (IB) | Eindeutige Execution-ID pro Fill. Fuer Deduplizierung von Fill-Events bei Reconnect/reqExecutions |

**Matching-Strategie:**

| Situation | Matching-Key | Logik |
|-----------|-------------|-------|
| Normalbetrieb (Order platziert, Events empfangen) | `clientOrderId` → `orderId/permId` via OrderRegistry |
| Reconnect: `reqOpenOrders()` | Match auf `clientOrderId`. IB liefert `orderId`+`permId`. Bekannt → State aktualisieren. Unbekannt → ESCALATE |
| Reconnect: `reqExecutions()` | Deduplizierung via `execId`. Jede `execId` wird nur einmal verarbeitet. Bereits bekannte Fills (in OrderRegistry) werden ignoriert |
| Reconnect: `reqPositions()` | Matching auf `instrumentId`+`account`. Vergleich mit internem Positions-State |

> **Duplikat-Pruefung bei submitOrder:** IbBrokerGateway prueft in der OrderRegistry, ob eine Order mit gleicher `clientOrderId` bereits existiert. Falls ja: kein erneuter Submit, bestehende Order weiter tracken.

### EventLog-Integration

Broker-Events werden **asynchron** ins EventLog geschrieben (gleiche Architektur wie Kap 2: async Write + Spool-Buffer im Live-Modus). **Kein EventLog-Write blockiert den IB-Receiver-Thread** — Events werden in eine In-Memory-Queue enqueued und vom EventLog-Writer-Thread asynchron persistiert.

| Event-Klasse | Volume | Drop-Policy |
|-------------|--------|-------------|
| `BrokerEvent` (ACCEPTED, FILLED, CANCELLED, REJECTED, PARTIALLY_FILLED) | Niedrig | **Non-droppable** — Pflicht fuer Replay und Reconciliation |
| `OrderSubmitted` (Zeitpunkt + TradeIntent) | Niedrig | **Non-droppable** |
| `OrderModified` (Zeitpunkt + neuer Intent) | Niedrig | **Non-droppable** |
| `BrokerConnectivityEvent` (Connected, Disconnected, Reconnecting, ESCALATE) | Niedrig | **Non-droppable** |
| `PositionSyncEvent` (Reconnect-Resync-Ergebnis) | Niedrig | **Non-droppable** |
| Heartbeat-Telemetrie (letzte Callback-Zeit, Reconnect-Zaehler) | Niedrig | **Best-effort** — darf bei Ueberlast gedroppt werden |

Jedes Event traegt `runId` als Join-Key zum `RunContext`.

> **Spool-Failure-Policy (Live):** Identisch zu Kap 2 — lokaler In-Memory Spool-Buffer. Bei Ueberlauf: `DataQualityEvent(ESCALATE, EVENTLOG_OVERFLOW)` → odin-core entscheidet.

---

## 6. IbAccountAdapter

**Verantwortung:** Account-Daten und Positions-Abfragen fuer Reconciliation und globalen Risk-Manager.

| Operation | TWS-API-Methode | Semantik |
|-----------|----------------|----------|
| Account-Updates | `reqAccountUpdates(subscribe=true)` | **Stream** (kein Request/Response). Liefert kontinuierlich Account-Werte (NetLiquidation, BuyingPower, etc.). Subscribe bei Connect, automatisch bei Reconnect |
| Positionen | `reqPositions()` | One-Shot Snapshot aller offenen Positionen. Genutzt bei: Start-of-Day, nach Reconnect, EOD-Reconciliation |

> **Modulgrenze:** IbAccountAdapter ist **kein separater Port**. Account-/Positionsdaten fliessen intern an odin-core (GlobalRiskManager, LifecycleManager) — nicht ueber die 4+1 Port-Schnittstelle. odin-core greift ueber ein internes Interface `BrokerAccountService` (in odin-api definiert) darauf zu.

---

## 7. Simulation: BrokerSimulator

### Architektur

`BrokerSimulator` implementiert den `BrokerGateway`-Port fuer den Simulationsmodus. Es wird **pro Pipeline instanziiert** fuer Order-/Fill-Isolation.

> **Account-Semantik:** Der BrokerSimulator simuliert **nur** Order-Lifecycle und Fills (pro Pipeline). **Account-Level-State** (Margin, Buying Power, Total Exposure, aggregierter Tagesverlust) wird von **GlobalRiskManager in odin-core** aggregiert — exakt wie im Live-Modus, wo IB ein gemeinsames Konto hat. Der Simulator braucht keinen eigenen Account-State; odin-core fragt die aggregierten Positionen/P&L ueber die BrokerGateway-Facades ab.

| Aspekt | Spezifikation |
|--------|---------------|
| Scope | Pro Pipeline (Order/Fill-Isolation). Account-State in odin-core (aggregiert) |
| Modul | odin-broker (Package: `de.odin.broker.sim`) |
| Interface | `BrokerGateway` (identisch zum Live-Modus) |
| Fill-Modell | Order akzeptiert → Fill auf **naechstem Bar-Open (t+1)** + konfigurierbare Slippage |
| Event-Emission | **Asynchron** — BrokerSimulator emittiert Events ueber denselben Callback-Mechanismus wie IbBrokerGateway. Nicht synchron „sofort Filled" |
| Latenz-Simulation | Konfigurierbare Verzoegerung zwischen submitOrder und ACCEPTED-Event (`odin.broker.sim.accept-delay-ms`) |

### Fill-Modell (v1)

```
submitOrder(TradeIntent)
  → [nach accept-delay] BrokerEvent(ACCEPTED)
  → [naechster Bar-Close → SimulationRunner taktet weiter]
  → [Bar-Open t+1] BrokerEvent(FILLED, fillPrice = open + slippage)
```

| Parameter | Default | Beschreibung |
|-----------|---------|-------------|
| `odin.broker.sim.slippage-bps` | 5 | Slippage in Basispunkten auf den Open-Preis |
| `odin.broker.sim.accept-delay-ms` | 100 | Simulierte Latenz bis ACCEPTED |
| `odin.broker.sim.partial-fill-enabled` | false | Partial Fills simulieren (v2) |
| `odin.broker.sim.rejection-rate-percent` | 0.0 | Zufaellige Rejections simulieren (Stress-Test, v2) |

### GTC-Stops in Simulation

GTC-Stops werden vom BrokerSimulator **aktiv ueberwacht**: Bei jedem neuen Bar-Event prueft der Simulator, ob ein Stop-Preis erreicht wurde. Fill erfolgt auf Stop-Preis + Slippage. Das simuliert das Broker-Verhalten bei GTC-Stops auch waehrend der Bar-Close-Barrier.

### Limitations (v1)

- Kein Partial-Fill-Modell (alle Orders werden komplett gefuellt)
- Kein Orderbook-Simulation (kein Market-Impact, kein Slippage-Modell basierend auf Liquiditaet)
- Keine Margin-Pruefung im Simulator (wird von GlobalRiskManager in odin-core durchgefuehrt)
- Rejection-Simulation nur via konfigurierbare Rate (kein regelbasiertes Rejection-Modell)

Diese Limitationen sind fuer v1 akzeptabel. Erweiterungen (Partial Fills, Order-Buch-basierte Slippage) koennen spaeter hinzugefuegt werden, ohne die BrokerGateway-Schnittstelle zu aendern.

### Kopplung an SimulationRunner

Der BrokerSimulator wird vom SimulationRunner ueber die **Bar-Close-Barrier** koordiniert:
1. Pipeline meldet Quiescence (Kap 2)
2. Brain verarbeitet Snapshot → Decision → TradeIntent
3. OMS → BrokerSimulator.submitOrder() → ACCEPTED-Event
4. SimulationRunner taktet zum naechsten Bar → BrokerSimulator prueft Fills → FILLED-Event
5. Global Risk aktualisiert → Barrier aufgehoben → naechstes Event

---

## 8. Reconnect und Subscription-Rehydration

### Reconnect-Ablauf

**Orchestrierung:** odin-core (LifecycleManager) steuert den Reconnect-Ablauf. IbSession fuehrt die technische Reconnect-Logik aus, odin-core entscheidet ueber Pipeline-State-Transitionen.

```
Phase 1: Erkennung + Pause
  1. IbSession erkennt Verlust (Heartbeat-Miss oder Connection-Error)
  2. IbSession publiziert BrokerConnectivityEvent(DISCONNECTED) → EventLog (non-droppable)
  3. odin-core empfaengt Event → versetzt ALLE Pipelines in PAUSED (keine neuen Orders, bestehende Stops bleiben aktiv bei IB)

Phase 2: Reconnect (IbSession, autonom)
  4. Reconnect-Versuch (bounded: 5 Versuche, Backoff 5/10/20/40/60s)
  5. Bei Erfolg: nextValidId()-Callback abwarten → ReqIdAllocator re-seed

Phase 3: Resync (IbSession → odin-core)
  6. IbSession publiziert BrokerConnectivityEvent(RECONNECTING, RESYNC_START)
  7. reqOpenOrders() → Offene Orders abgleichen (Matching via clientOrderId/permId)
  8. reqExecutions() → Verpasste Fills abgleichen (Deduplizierung via execId)
  9. reqPositions() → Positions-Snapshot → PositionSyncEvent ins EventLog
  10. Broker-State mit internem State abgleichen (Broker = Source of Truth)
  11. reqAccountUpdates(subscribe=true) → Account-Stream neu aufbauen
  12. Subscription-Rehydration (siehe unten)

Phase 4: Resume
  13. IbSession publiziert BrokerConnectivityEvent(RECONNECTED, RESYNC_DONE)
  14. odin-core setzt Pipelines aus PAUSED zurueck in vorherigen State

Fehlerfall:
  Alle Reconnect-Versuche erschoepft →
  BrokerConnectivityEvent(ESCALATE, RECONNECT_EXHAUSTED) → odin-core entscheidet Kill-Switch
```

### Subscription-Rehydration

IbSession fuehrt eine `SubscriptionRegistry` ueber alle aktiven Subscriptions:

| Feld | Beschreibung |
|------|-------------|
| `reqId` | Request-ID der Subscription |
| `pipelineId` | Zugehoerige Pipeline |
| `subscriptionType` | TICK, L2, REALTIME_BAR, HISTORICAL |
| `instrumentContract` | IB-Contract-Spezifikation |
| `refCount` | Anzahl aktiver Referenzen (fuer shared Subscriptions) |

Nach Reconnect:
1. Fuer jeden Registry-Eintrag: neue Request-ID allokieren (via ReqIdAllocator)
2. TWS-API-Methode erneut aufrufen (reqMktData, reqMktDepth, etc.)
3. RequestRegistry mit neuen Request-IDs aktualisieren
4. Buffer-Luecken (waehrend Disconnect): odin-data erkennt den Gap via Sequence-Number-Luecke (Kap 2) und requestet Historical Data zum Auffuellen

### Position-Resync

| Situation | Reaktion |
|-----------|---------|
| Intern: Position offen, Broker: gleiche Stueckzahl | OK — kein Handlungsbedarf |
| Intern: Position offen, Broker: kleiner (Partial Fill waehrend Disconnect) | Internen State korrigieren, Stops anpassen |
| Intern: Position offen, Broker: geschlossen (Stop getriggert) | State auf geschlossen, P&L berechnen, BrokerEvent(FILLED) ins EventLog |
| Intern: Keine Position, Broker: Position offen | **ESCALATE** → odin-core entscheidet. Konservativ: GTC-Stop setzen, Operator-Alert |

> **Broker = Source of Truth.** Bei jedem Reconnect gilt der Broker-State als autoritativ. Der interne State wird angepasst, nicht umgekehrt.

---

## 9. TWS API 10.40 — Constraints und Mitigations

| Constraint | Auswirkung | Mitigation |
|------------|-----------|-----------|
| **Single-threaded Callbacks** | Alle EWrapper-Callbacks auf einem Reader-Thread | Non-blocking Dispatch in Pipeline-Queues via IbDispatcher |
| **EClientSocket nicht thread-safe** | Gleichzeitige Sends fuehren zu Korruption | Outbound-Command-Queue + dedizierter Sender-Thread (in IbSession) |
| **Market-Data-Lines begrenzt** | Account hat limitierte gleichzeitige Subscriptions | Monitoring der Subscription-Anzahl. Bei 2–3 Instrumenten unkritisch (< 50 Lines) |
| **Historical-Data-Pacing** | Max 6 identical-Contract-Requests in 2s, max 60 Requests/10 Min gesamt | `PacingGuard` in IbSession: Contract-aware Rate-Limiting-Queue mit Token-Bucket |
| **Order-ID monoton steigend** | IB verlangt strikt steigende Order-IDs | ReqIdAllocator (AtomicLong), Seed aus `nextValidId()`-Callback. Seed wird bei Reconnect aktualisiert |
| **Market-Data-Farm-Wechsel** | IB-Backend-Server-Wechsel → kurzer Datenverlust | Erkannt via Error-Codes 2103/2105/2106 → Resubscribe nur betroffener Feeds. Sequence-Gap wird von odin-data erkannt |
| **reqAccountUpdates = Stream** | Kein Request/Response — subscribe-based | Subscribe bei Connect, re-subscribe bei Reconnect. Account-Events asynchron verarbeiten |

---

## 10. Thread-Modell (Broker Layer)

| Thread | Aufgabe | Scope |
|--------|---------|-------|
| IB-Receiver | TWS-API Reader-Thread (von API bereitgestellt, nicht konfigurierbar) | Global (1 Thread) |
| IB-Sender | Outbound-Command-Queue → serialisierte EClientSocket-Aufrufe | Global (1 Thread) |
| Heartbeat-Monitor | Lokaler monotoner Timer, prueft letzten Callback-Zeitpunkt alle 30s | Global (1 Thread) |

> **IbDispatcher ist kein eigener Thread.** Das Dispatching (Callback → Pipeline-Queue) findet **auf dem IB-Receiver-Thread** statt. Es muss non-blocking sein (nur Queue-Enqueue). Aufwendige Verarbeitung findet in den Pipeline-Threads statt (in odin-data und odin-execution).

> **Sender-Thread:** Alle ausgehenden Aufrufe (`placeOrder`, `reqMktData`, `cancelOrder`, `reqHistoricalData`, etc.) werden ueber die Outbound-Command-Queue serialisiert. Ein dedizierter Sender-Thread konsumiert und fuehrt sequentiell aus. Damit kein Race/Interleaving bei parallelen Pipeline-Operationen.

> **Simulation:** Im Sim-Modus existieren keine IB-Threads. Der BrokerSimulator wird direkt vom Pipeline-Thread aufgerufen (synchron im Decision-Cycle) bzw. vom SimulationRunner (Fills bei Bar-Open).

### Broker-Event-Queue Backpressure

IbDispatcher enqueued BrokerEvents in per-Pipeline Queues. Da das Volumen von Broker-Events niedrig ist (Orders/Fills/Positions, nicht Ticks), sind diese Queues **bounded mit grosszuegiger Kapazitaet**:

| Queue | Kapazitaet | Drop-Policy |
|-------|-----------|-------------|
| Broker-Event-Queue (per Pipeline) | 1.000 Events | **Kein Drop.** Bei Voll: ESCALATE → odin-core. Das sollte nie eintreten (max. 100 Orders/Tag/Pipeline) |

> **Kein Blocking auf dem Receiver-Thread:** Falls die Queue unerwartet voll ist (z.B. durch blockierten Consumer), publiziert IbDispatcher ein `BrokerConnectivityEvent(ESCALATE, BROKER_QUEUE_FULL)` und verwirft das Event. Das ist ein Extremfall-Fallback — unter normalen Bedingungen tritt er nicht ein.

---

## 11. Konfiguration

```properties
# odin-broker.properties

# IB Gateway Verbindung
odin.broker.gateway-host=localhost
odin.broker.gateway-port=4002
odin.broker.client-id=1

# Heartbeat (Infrastruktur-Timer, nicht MarketClock)
odin.broker.heartbeat-interval-s=30
odin.broker.heartbeat-miss-threshold=3

# Reconnect
odin.broker.reconnect-max-attempts=5
odin.broker.reconnect-initial-backoff-ms=5000
odin.broker.reconnect-max-backoff-ms=60000

# Historical Data Pacing (TWS API Constraint)
odin.broker.pacing.max-identical-contract-requests=6
odin.broker.pacing.identical-contract-window-s=2
odin.broker.pacing.max-total-requests-per-10min=60

# BrokerSimulator (Sim-Modus)
odin.broker.sim.slippage-bps=5
odin.broker.sim.accept-delay-ms=100
odin.broker.sim.partial-fill-enabled=false
odin.broker.sim.rejection-rate-percent=0.0
```

---

## 12. Abhaengigkeiten und Schnittstellen

### Implementierte Ports (aus odin-api)

| Port | Implementierung | Scope |
|------|----------------|-------|
| `MarketDataFeed` | `IbMarketDataFeed` (Live) | Per Pipeline (Facade ueber IbSession) |
| `BrokerGateway` | `IbBrokerGateway` (Live) | Per Pipeline (Facade ueber IbSession) |
| `BrokerGateway` | `BrokerSimulator` (Sim) | Per Pipeline (isoliert) |

### Zusaetzliche interne Interfaces (in odin-api definiert)

| Interface | Implementierung | Konsumenten |
|-----------|----------------|-------------|
| `BrokerAccountService` | `IbAccountAdapter` | odin-core (GlobalRiskManager, LifecycleManager, Reconciliation) |

> **Kein separater Port.** `BrokerAccountService` ist ein internes Interface fuer Account-/Positionsdaten. Es erweitert die 4+1 Port-Architektur nicht, sondern ist ein infrastruktureller Zugang fuer odin-core.

### Interne Komponenten (nur in odin-broker)

| Komponente | Zweck |
|-----------|-------|
| `IbSession` | Singleton — Socket, Threads, ReqIdAllocator, RequestRegistry, Subscription-Ownership |
| `IbDispatcher` | Callback-Routing auf Pipeline-Queues (Teil von IbSession, kein eigener Thread) |
| `PacingGuard` | Rate-Limiting fuer Historical-Data-Requests (TWS API Constraint) |
| `SubscriptionRegistry` | Mapping: reqId → (pipelineId, subscriptionType, contract, refCount) |
| `OrderRegistry` | Mapping: clientOrderId → (orderId/permId, pipelineId, state) |

### Abhaengigkeiten

- `odin-api` (Ports, Events, DTOs)
- TWS API 10.40 (als externe Abhaengigkeit)
- Keine Abhaengigkeit auf andere Fachmodule (strikt, Kap 1)
