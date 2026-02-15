# ODIN — Kapitel 7: Order Management System (OMS)

Version: 0.3 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R3 — P0/P1 Fixes R2 (non-droppable Definition, clientOrderId Uniqueness, pipelineId→instrumentId, Cross-Module Config Contract, Sizing-Formel, Pre-Market repricing-enabled)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt das Order Management System von ODIN: Risk Gate (Position Sizing, Pre-Trade-Limits), Order-Erzeugung, Multi-Tranchen-Management, Fill-Event-Handling und die Execution Policy. Das OMS ist die Bruecke zwischen TradeIntent (aus dem Decision Arbiter, Kap 6) und der Broker-Anbindung (BrokerGateway, Kap 3).

**Modulzuordnung:** `odin-execution` (Package: `de.odin.execution`)

**Zentrale Abhaengigkeiten:**
- `TradeIntent` (von Decision Arbiter, Kap 6) — Entry/Exit-Absicht mit Parametern
- `BrokerGateway` (Port aus odin-api, Kap 3) — Order-Submission und BrokerEvent-Callbacks
- `MarketClock` (Port aus odin-api) — Sole Time Source fuer Repricing-Timing, Forced-Close, Timeouts
- `EventLog` (Port aus odin-api) — Persistierung aller Order-Events (non-droppable)
- `RunContext` (aus odin-api) — runId als Join-Key fuer alle Events
- `AccountRiskState` (von odin-core, read-only) — Globaler Risk-Status ueber alle Pipelines. Das OMS **liest** den State, **mutiert ihn nie direkt**. Updates erfolgen event-driven: OMS emittiert FillEvent/PositionUpdate → ein RiskStateUpdater in odin-core materialisiert den aktualisierten AccountRiskState
- `MarketSnapshot` (von odin-data) — Aktuelle Preise fuer Repricing
- `IndicatorResult` (von KPI-Engine, Kap 4) — Indikator-Werte fuer Runner-Trailing

---

## 2. Architektur

```
TradeIntent (von Decision Arbiter, Kap 6)
       │
       v
┌──────────────────────────────┐
│         Risk Gate            │
│                              │
│  Position Sizing             │
│  Tagesbudget-Pruefung        │
│  R/R-Validation              │
│  Exposure-Limits             │
│  → RejectReasons (Kap 6 §11)│
└──────────┬───────────────────┘
           │ (SizedOrder)
           v
┌──────────────────────────────┐
│  Order Management Service    │
│                              │
│  Tranchen-Calculator         │
│  Stop-Management             │
│  Fill-Event-Handling         │
│  Repricing-Logik             │
└──────────┬───────────────────┘
           │ (BrokerGateway Port)
           v
┌──────────────────────────────┐
│  odin-broker                 │
│  Live: IbBrokerGateway       │
│  Sim:  BrokerSimulator       │
└──────────────────────────────┘
```

> **Port-Nutzung:** Das OMS arbeitet ausschliesslich gegen den `BrokerGateway`-Port (Kap 1, Kap 3). Keine direkte IB-Abhaengigkeit — `odin-execution` haengt nur von `odin-api` ab. Im Live-Modus wird `IbBrokerGateway` injiziert, im Sim-Modus der `BrokerSimulator` (Kap 3).

---

## 3. Risk Gate

### Position Sizing: Fixed Fractional Risk

| Schritt | Formel |
|---------|--------|
| 1. Risikobetrag (EUR) | `max_risk_eur = min(Tageskapital x maxRiskPercent, remaining_budget x safetyMargin)` |
| 1b. FX-Conversion | `max_risk_contract_ccy = max_risk_eur x FX_spot(EUR → contract_currency)`. FX-Rate aus Broker-/Marktdaten, Waehrung aus Contract-Definition (Kap 0, §3.2: Currency/Exchange-Awareness) |
| 2. Stop-Distance | `stop_dist = TradeIntent.stopLevel - TradeIntent.entryPrice` (bereits ATR-basiert, aus Kap 6) |
| 2b. Stress-Adjusted Stop | `effective_stop_dist = stop_dist x stressFactorHighVol` (Default: 1.0, bei Regime HIGH_VOLATILITY: konfigurierbar) |
| 3. Stueckzahl | `shares = floor(max_risk_contract_ccy / effective_stop_dist)` |
| 4. Positionswert | `position_value = shares x entry_price` (in Contract-Waehrung) |
| 5. Cap | `if position_value > maxPositionPercent x Tageskapital (in Contract-Waehrung) → shares reduzieren` |
| 6. Confidence- und Slippage-Skalierung | `shares = floor(shares x min(regime_confidence, quant_score) x slippageReductionFactor)`. `slippageReductionFactor` = 1.0 (normal) oder `1.0 - slippageAdjustmentReductionPercent/100` (wenn Slippage-Auto-Adjustment aktiv, §9). Niedrige Konfidenz oder hohe Slippage → kleinere Position. **Reihenfolge:** Caps (Schritt 5) werden VOR Skalierung angewandt. Skalierung (Schritt 6) kann shares nur reduzieren, nie erhoehen — daher kein erneuter Cap-Check noetig |

### Dynamisches Tages-Budget

```
realisierte_verluste = sum(alle Tages-Trades mit negativem P&L)  // nur Verluste, Gewinne ignoriert
unrealisierte_verluste = sum(alle offenen Positionen mit negativem P&L)  // nur negative
remaining_budget = (Tageskapital x hardStopPercent) - abs(realisierte_verluste) - abs(unrealisierte_verluste)
```

> **Nur Verluste reduzieren das Budget.** Gewinne erhoehen es nicht — sie dienen der Performance, nicht der Risikoerhoehung. Die Safety Margin (`safetyMargin`, Default: 0.8) stellt sicher, dass ein einzelner Trade den Hard-Stop nicht durchbrechen kann.

### Pre-Trade-Limits

> **Globale Koordination:** Tagesbudget, Exposure, Round-Trips und Cooling-Off sind **account-global** (nicht pro Pipeline). Der `AccountRiskState` in `odin-core` aggregiert ueber alle Pipelines (Kap 0, §2). Das Risk Gate fragt den globalen State ab, bevor es einen Trade erlaubt.

| Limit | Pruefung | Scope | RejectReason (Kap 6 §11) |
|-------|---------|-------|--------------------------|
| Tagesbudget erschoepft? | remaining_budget < min. Trade-Risk | Global | RISK_BUDGET_EXHAUSTED |
| Max. Round-Trips/Tag? | Counter >= `maxRoundTrips` | Global | RISK_BUDGET_EXHAUSTED |
| Exposure-Limit? | Summe aller Positionen + neue > `maxExposurePercent` x Tageskapital | Global | RISK_EXPOSURE_EXCEEDED |
| R/R-Ratio? | < `minRrRatio` | Per Trade | RISK_RR_INSUFFICIENT |
| Cooling-Off aktiv? | N Verluste in Folge (global) → Pause aktiv | Global | (Entry blockiert in Kap 6) |

### Risk/Reward-Berechnung

| Situation | Target-Ermittlung |
|-----------|------------------|
| TradeIntent.targetPrice vorhanden (R-basiert oder LLM-validiert, Kap 6 §8) | `reward = targetPrice - entryPrice` |
| Kein targetPrice | `reward = 2 x ATR(14)` (synthetisches Target). `synthetic_target = entryPrice + reward` |

`risk_reward = reward / stop_dist` — muss >= `minRrRatio` sein.

---

## 4. Order-Typ-Strategie

### Entry-Orders

| Situation | Order-Typ | Preis-Bestimmung |
|-----------|----------|-----------------|
| Normal Entry | Limit Order | `entryPrice` aus TradeIntent (MarketSnapshot.currentPrice als Referenz). Wenn LLM-`entry_price_zone.ideal` vorhanden und innerhalb ATR-Plausibilitaet (Kap 5, §4): als bevorzugter Preis verwenden. Sonst: Current Ask |
| Entry nach `repricingIntervalS` ohne Fill | Cancel/Replace | Neuer Limit = min(Ask, `entryPrice + repricingOffsetAtr x ATR`). Max. `maxRepricingCycles` Zyklen |
| Nach `maxRepricingCycles` ohne Fill | Abandon | Kein Entry. State → OBSERVING |
| Preis wesentlich verschoben (> 1x ATR seit Intent) | Abandon | Opportunity verpasst. Intent verwerfen |

> **LLM-Zone als Kontext (Analyst-Prinzip, Kap 5 §2):** Die `entry_price_zone` aus dem LLM ist ein **optionaler Kontext-Hinweis** fuer die Preisfindung. Fehlt sie, verwendet das OMS den `entryPrice` aus dem TradeIntent. Die Zone triggert nie eigenstaendig einen Entry — der Entry-Trigger kommt aus der Rules Engine (Kap 6 §5).

### Exit-Orders

| Situation | Order-Typ | Details |
|-----------|----------|---------|
| Normal Exit (Tranchen) | Limit Order | Preis = aktueller Bid. Repricing alle `repricingIntervalS` (MarketClock-basiert) |
| Forced Close | Market Order | Zeitpunkt: `MarketClock.now() > RTH-Ende - forcedCloseBufferMin` (Exchange-Kalender, Kap 0 §3.2). Position MUSS geschlossen werden |
| Kill-Switch | Market Order | Notfall-Schliessung (odin-core, sofort) |
| Stop-Loss | Stop-Limit | Stop-Trigger = `TradeIntent.stopLevel`, Limit-Offset = `stopLimitOffsetPercent` vom Stop-Level. v1: nur Long (Sell-Stop-Limit) |

### Market-Order-Policy

Market Orders sind **ausschliesslich** erlaubt bei:
1. Forced Close (MarketClock-basiert, RTH-Ende - Buffer)
2. Kill-Switch
3. Stop-Market-Fallback (Stop-Limit nach `stopMarketFallbackTimeoutS` nicht gefuellt)
4. Partial-Fill-Notfall auf Stop
5. Gap-Opening unter Stop

Jeder Market-Order-Einsatz wird mit Ausnahmegrund ins **EventLog** geschrieben (non-droppable, Typ: `MARKET_ORDER_USED`).

> **Non-droppable Garantie (Live):** „Non-droppable" bedeutet: das OMS **verwirft nie absichtlich** einen Event-Write. Mechanismus: bounded In-Memory-Spool (Kap 2). Spool-Overflow → ESCALATE an odin-core (Kill-Switch-Entscheidung), nicht stillschweigendes Verwerfen. **Crash-Persistenz ist kein v1-Ziel** — bei JVM-Crash gehen in-flight Spool-Eintraege verloren (akzeptiertes v1-Risiko, Kap 0 §10). Post-Crash-Reconciliation (§6) stellt den konsistenten State wieder her. In der Simulation: synchrone Writes innerhalb der Bar-Close-Barrier, kein Verlustrisiko.

### Stop-Market-Fallback

| Bedingung | Aktion |
|-----------|--------|
| Stop-Limit nicht gefuellt nach `stopMarketFallbackTimeoutS` (MarketClock) | Cancel, ersetze durch Stop-Market |
| Gap-Opening unter Stop-Level | Sofort Market-Order bei Eroeffnung |
| LULD-Halt aktiv | Nach Resume sofort Market-Close |

---

## 5. Scaling-Out-Strategie (Multi-Tranchen)

### Tranchen-Varianten (budgetabhaengig)

| Variante | Bedingung (Positionswert) | Tranchen | Aufteilung |
|----------|--------------------------|----------|------------|
| Kompakt (3T) | < `compactThresholdUsd` | 3 | 40% @ 1R, 40% @ 2R, 20% Runner |
| Standard (4T) | `compactThresholdUsd`–`standardThresholdUsd` | 4 | 30% @ 1R, 30% @ Key-Level, 25% @ 2R, 15% Runner |
| Voll (5T) | > `standardThresholdUsd` | 5 | 25% @ 1R, 25% @ Key-Level, 25% @ 2R, 15% @ 3R, 10% Runner |

Die Variante wird automatisch bei Entry basierend auf der Positionsgroesse gewaehlt.

> **Abgrenzung zu Kap 6:** Scaling-Out ist **OMS-Maintenance** (Kap 6, §3.2) — kein TradeIntent. Die Tranchen-Berechnung und -Ausfuehrung laufen OMS-intern, nicht im Decision Loop.

### Order-Architektur: Stop und Targets getrennt

- **Stop-Loss-Order:** Eigenstaendige Stop-Order auf **gesamte Restposition**. Wird bei jedem Teil-Exit auf verbleibende Stueckzahl angepasst (Reduce-Only). **Nicht in OCA-Gruppe.** GTC (Good-Til-Cancel) beim Broker — ueberlebt OMS-Neustart (Crash-Recovery, Kap 0 §10)
- **Target-Orders:** Separate Limit-Sell-Order pro Tranche mit exakter Stueckzahl. Targets untereinander unabhaengig (keine OCA). Verwaltung durch OMS-Logik

> **Kein OCA fuer Stop/Target-Kombination.** OCA wuerde bei Target-Fill den Stop canceln — fatal. Synchronisation erfolgt ueber interne OMS-Logik.

### Crash-Recovery und Order-Orphan-Schutz

Bei OMS-Crash oder Disconnect koennen Target-Orders verwaist weiterleben, waehrend der Stop bereits ausgefuehrt wurde (oder umgekehrt). Schutzmechanismen:

| Massnahme | Details |
|-----------|---------|
| Startup-Reconciliation | Bei jedem OMS-Start: `BrokerGateway.requestOpenOrders()` + `BrokerGateway.requestPositions()`. Verwaiste Orders ohne zugehoerige Position werden storniert |
| Position-Side-Guard | OMS trackt Netto-Position. Jeder Fill wird gegen erwartete Richtung geprueft. Fill der Netto-Position umkehren wuerde → sofort Cancel aller offenen Orders + Alert |
| Stop als GTC | Stop-Order als GTC beim Broker — ueberlebt OMS-Neustart (Kap 0 §10: Crash-Recovery via GTC-Stops) |
| Heartbeat-Pruefung | OMS prueft periodisch (`reconciliationIntervalS`, MarketClock-basiert): offene Orders konsistent mit internem State? Diskrepanz → Reconciliation-Trigger |

### Fill-Event-Handling

Fills kommen als asynchrone `BrokerEvent`-Callbacks ueber den `BrokerGateway`-Port (Kap 3):

| Event | OMS-Reaktion |
|-------|-------------|
| Target-Tranche gefuellt | Stop-Quantity auf Restposition reduzieren. Bei Tranche 1: Stop auf Break-Even nachziehen |
| Stop getriggert | Alle offenen Target-Orders stornieren. Position geschlossen → State OBSERVING |
| Partial Fill auf Target | Gefuellte vs. offene Menge tracken. Stop-Quantity anpassen |
| Partial Fill auf Stop | Restmenge + alle Targets sofort als Market schliessen (Notfall) |

> **EventLog-Schema (Mandatory Fields):** Jeder Fill, jede Order-Aenderung und jedes Reject wird ins EventLog geschrieben (non-droppable). Pflichtfelder aller Order-Events: `runId`, `instrumentId`, `brokerOrderId`, `clientOrderId`, `eventType`, `marketClockTimestamp`. Zusaetzlich je Typ: FillEvent: `fillPrice`, `fillQuantity`, `side`, `slippageBps`. OrderStatusEvent: `previousStatus`, `newStatus`, `reason`. RejectEvent: `rejectReason`, `emitter` (Kap 6 §11). MarketOrderUsedEvent: `justification` (Enum: FORCED_CLOSE, KILL_SWITCH, STOP_FALLBACK, PARTIAL_FILL_EMERGENCY, GAP_OPENING).

### Runner-Trailing

Runner-Trailing ist **OMS-Maintenance** (Kap 6, §3.2) — kein TradeIntent. Die Nachfuehrung erfolgt auf **Tick-Ebene** (nicht nur bei Bar-Close):

| Situation | Trailing-Regel |
|-----------|---------------|
| Standard | Trail unter EMA(9) oder letztes Higher-Low (naeher am Kurs) |
| Ueberhitzt (RSI > rsiOverheatThreshold, oberes Bollinger-Band) | Engerer Trail, unter letzte Kerze |
| Ruecksetzer gesund (Preis haelt MA-Zone) | Runner halten |
| Ruecksetzer kippt (Close unter MA-Zone, Lower-Low) | Runner sofort schliessen |

---

## 6. OMS Order-Lifecycle

### Zustandsmaschine (pro Order)

```
CREATED → SUBMITTED → [ACCEPTED] → FILLED | PARTIALLY_FILLED | CANCELLED | REJECTED
                                       │
                            PARTIALLY_FILLED → FILLED | CANCELLED (Restmenge)
```

### OMS-Gesamtlifecycle (pro Pipeline)

```
NO_POSITION → ENTRY_PENDING → POSITIONED → EXIT_PENDING → NO_POSITION
                  │                              │
                  └── (Abandon/Reject) ──────────┘── → NO_POSITION
```

| OMS-State | Trigger | Verhalten |
|-----------|---------|----------|
| NO_POSITION | TradeIntent (Entry) vom Arbiter | → Risk Gate → SizedOrder → Entry-Order submittieren → ENTRY_PENDING |
| ENTRY_PENDING | Fill-Event | Stop-Order + Target-Orders platzieren → POSITIONED |
| ENTRY_PENDING | Abandon (Repricing-Limit, Preis-Drift) | Cancel Entry-Order → NO_POSITION |
| ENTRY_PENDING | Reject (Risk Gate) | → NO_POSITION |
| POSITIONED | Target-Tranche-Fill | Stop-Quantity reduzieren. Alle gefuellt → NO_POSITION |
| POSITIONED | Stop-Fill | Alle offenen Targets canceln → NO_POSITION |
| POSITIONED | TradeIntent (Exit) | Exit-Order submittieren → EXIT_PENDING |
| POSITIONED | Forced-Close | Market-Order → EXIT_PENDING |
| EXIT_PENDING | Fill-Event | Alle offenen Orders canceln → NO_POSITION |

### Stop-Fallback-Safety (No-Stop-Gap-Minimierung)

Beim Wechsel von Stop-Limit zu Stop-Market (Fallback nach Timeout):

1. **Bevorzugt: Modify-in-place** (`BrokerGateway.modifyOrder()`) — aendert den bestehenden Stop-Limit in Stop-Market ohne Cancel-Luecke
2. **Fallback: Cancel+Replace** — Cancel bestehenden Stop-Limit, sofort neuen Stop-Market submittieren. Zwischen Cancel-Confirm und neuer Order existiert ein **No-Stop-Gap**
3. **Guard:** Waehrend des No-Stop-Gap prueft das OMS jeden eingehenden Tick gegen das Stop-Level. Bei Unterschreitung → sofort Market-Order (unabhaengig vom Stop-Status)
4. **Cancel-Failure:** Wenn Cancel fehlschlaegt (Order bereits filled) → OMS verarbeitet den Fill-Event normal (Stop wurde ausgefuehrt)

### Reconciliation-Algorithmus (Startup + periodisch)

| Schritt | Aktion |
|---------|--------|
| 1 | `BrokerGateway.requestOpenOrders()` → Set der Broker-Orders |
| 2 | `BrokerGateway.requestPositions()` → Netto-Position pro Instrument |
| 3 | Matching: Broker-Orders gegen OMS-internes Order-Tracking (Key: `clientOrderId`) |
| 4 | Orphaned Orders (beim Broker, nicht im OMS): Cancel |
| 5 | Missing Orders (im OMS, nicht beim Broker): OMS-State korrigieren (Order als CANCELLED markieren) |
| 6 | Position-Mismatch: Broker-Position als Source of Truth. OMS passt internen State an. Alert wenn Diskrepanz > 0 |
| 7 | EventLog-Eintrag: `RECONCILIATION_COMPLETED` mit Ergebnis (matches, orphans, mismatches) |

> **Idempotenz:** Reconciliation ist idempotent — mehrfache Ausfuehrung aendert den State nicht, wenn er bereits konsistent ist. Key fuer Matching ist die `clientOrderId` (Kap 3: Pipeline-Prefix + Sequence).

---

## 7. Repricing-Degradation (alle Zeitfenster MarketClock-basiert)

### Pacing-Guard

Repricing erzeugt Cancel/Replace-Paare (neue Order-IDs). Um API-Rejects und Disconnects zu vermeiden:

| Massnahme | Limit |
|-----------|-------|
| Max. Cancel/Replace pro Instrument | `maxCancelReplacePerMinute` (Default: 6, konservativ unter IB-Limits) |
| Bevorzugte Methode | `BrokerGateway.modifyOrder(orderId, newParams)` (Port-Capability, Kap 3). Wenn die Implementierung Modify unterstuetzt (IB: ja), wird die Order in-place geaendert. Wenn nicht (z.B. anderer Broker): automatischer Fallback auf Cancel + New Order. Die Ersatz-Order erhaelt eine **neue clientOrderId** (gleicher Pipeline-Prefix, naechste Sequence — Kap 3). Der Port gibt die neue orderId/clientOrderId zurueck, das OMS aktualisiert sein internes Tracking. Uniqueness der clientOrderId ist Port-Vertrag |
| Global Rate | Repricing-Traffic zaehlt gegen die Outbound-Command-Queue (Kap 3), Sender-Thread serialisiert |

### Degradation-Stufen

| Bedingung | Reaktion | Recovery |
|-----------|----------|---------|
| Cancel/Replace-Rate > 5:1 (30 Min MarketClock) | Repricing-Intervall verdoppeln | Bis Rate < 3:1 |
| Order/Trade-Ratio > 10:1 (60 Min MarketClock) | Repricing auf 1 Zyklus, kein Retry | Bis Ratio < 6:1 |
| Fill-Rate < 30% (5 Versuche) | Naechsten Intent halbieren oder OBSERVING | Bis naechster Decision-Cycle |

---

## 8. Pre-Market-Regeln

| Regel | Default | Config-Key |
|-------|---------|-----------|
| Mindest-Spread | < 0.5% | `odin.execution.premarket.min-spread-percent` |
| Mindest-Volumen | > 10.000 in letzten 30 Min Pre-Market | `odin.execution.premarket.min-volume` |
| Positionsgroesse | Max. 25% Tageskapital (statt `maxPositionPercent`) | `odin.execution.premarket.max-position-percent` |
| Repricing | Deaktiviert (`premarket.repricing-enabled=false`). Nur ein Limit-Order-Versuch, kein Cancel/Replace | `odin.execution.premarket.repricing-enabled` |

> **MarketClock-basiert:** Pre-Market-Erkennung ueber den Exchange-Kalender der MarketClock (Kap 0, §3.2). Kein hartcodierter Zeitwert.
>
> **Override-Vorrang:** Pre-Market-Regeln ueberschreiben die entsprechenden Normalwerte vollstaendig: `premarket.max-position-percent` statt `risk.max-position-percent`, kein Repricing (statt `repricing.max-cycles`), Pre-Market-Spread-/Volumen-Gate als zusaetzliche Vorbedingung (kein Normalfall-Pendant). Sobald MarketClock RTH-Open signalisiert, gelten wieder die Normalwerte.

---

## 9. Slippage-Management

| Aspekt | Regelung |
|--------|---------|
| Per-Trade Tracking | Fill-Preis vs. Intent-Preis. Im EventLog persistiert |
| Alert-Schwelle | Avg. Slippage > `slippageAlertThresholdPercent` ueber 5 Trades → Warnung |
| Auto-Adjustment | Avg. Slippage > `slippageAdjustmentThresholdPercent` ueber `slippageAdjustmentWindowSize` Trades → **naechste** Positionsgroesse (Risk Gate Schritt 6) um `slippageAdjustmentReductionPercent` reduzieren. Gilt bis Slippage-Durchschnitt unter Schwelle faellt. Betrifft nur neue Entries, nicht bestehende Positionen |
| Max-Slippage-Cap | Slippage > `maxSlippagePerTradePercent` auf einem Trade → Warnung im EventLog + Cooling-Off-Trigger-Pruefung |

---

## 10. Transaktionskosten

> **Konfigurierbare Naeherungswerte.** Die Defaults sind konservative Schaetzungen fuer IB-Tiered-Pricing (US-Aktien). Fuer v1 ausreichend — praezise volumenabhaengige Staffelung bei Bedarf spaeter.

| Komponente | Default | Config-Key |
|-----------|---------|-----------|
| Kommissionen | 0.0035 USD/Share (min. 0.35, max. 1%) | `odin.execution.costs.commission-per-share` |
| Spread-Kosten | Halber Bid-Ask-Spread in R/R-Berechnung | — (live aus MarketSnapshot) |
| FX-Kosten | 2 bps (0.002%) | `odin.execution.costs.fx-cost-bps` |

**Verwendung im OMS:**
1. **Risk Gate (§3, R/R-Berechnung):** `total_cost = commission_round_trip + spread_cost + fx_cost`. `effective_reward = reward - total_cost`. `risk_reward = effective_reward / stop_dist` — total_cost reduziert die Reward-Seite
2. **EventLog (P&L-Tracking):** Jedes FillEvent enthaelt die geschaetzten Kosten. `netPnl = grossPnl - total_cost` fuer den Tages-P&L im AccountRiskState
3. **Simulation:** Identische Kostenberechnung. BrokerSimulator liefert Fill-Events ohne Kosten → OMS addiert Kosten nachtraeglich (gleicher Codepfad wie Live)

---

## 11. Simulation-Semantik

### Identischer Codepfad

Das OMS ist in Live und Simulation **identisch** (Kap 0, §3.1). Unterschiede entstehen durch die injizierten Port-Implementierungen:

| Aspekt | Live | Simulation |
|--------|------|------------|
| BrokerGateway | IbBrokerGateway (TWS API) | BrokerSimulator (Kap 3) |
| Fill-Modell | Echte IB-Fills (async Callbacks) | Bar-Open + konfigurierbare Slippage (Kap 0, §10). Kein Fill in same Bar |
| MarketClock | SystemMarketClock (Echtzeit) | SimClock (vom Runner gesteuert) |
| Repricing-Timing | `repricingIntervalS` MarketClock-Sekunden (Echtzeit) | SimClock-Sekunden (deterministische Schritte) |
| EventLog-Writes | Asynchron (non-blocking Spool) | Synchron (innerhalb Bar-Close-Barrier) |

### RunContext-Nutzung

- `runId` als Join-Key in allen EventLog-Eintraegen (Order-Events, Fill-Events, Reject-Events)
- `mode` fuer EventLog-Schreibmodus (sync vs. async)

### BrokerSimulator-Interaktion

Der BrokerSimulator (Kap 3) liefert **asynchrone** BrokerEvent-Callbacks — identisch zum Live-Modell. Das OMS unterscheidet nicht zwischen Live und Sim bei der Fill-Verarbeitung. Der SimulationRunner koordiniert die Bar-Close-Barrier (Kap 0, §3.3): OMS-Events muessen bis zum definierten Cutoff verarbeitet sein, bevor der naechste Bar freigegeben wird.

---

## 12. Konfiguration

```properties
# odin-execution.properties

# Position Sizing
odin.execution.risk.max-risk-percent=3.0
odin.execution.risk.max-position-percent=50
odin.execution.risk.max-exposure-percent=80
odin.execution.risk.safety-margin=0.8
odin.execution.risk.stress-factor-high-vol=1.3
odin.execution.risk.min-rr-ratio=1.5

# Tages-Limits
odin.execution.limits.max-round-trips=5
odin.execution.limits.hard-stop-percent=10.0
odin.execution.limits.cooling-off-losses=3
odin.execution.limits.cooling-off-duration-min=30

# Scaling-Out (Schwellenwerte in USD)
odin.execution.tranchen.compact-threshold-usd=5000
odin.execution.tranchen.standard-threshold-usd=15000

# Repricing (MarketClock-basiert)
odin.execution.repricing.interval-s=5
odin.execution.repricing.max-cycles=3
odin.execution.repricing.offset-atr-factor=0.1

# Stop
odin.execution.stop.limit-offset-percent=0.1
odin.execution.stop.market-fallback-timeout-s=5

# Reconciliation
odin.execution.reconciliation.interval-s=60

# Slippage
odin.execution.slippage.alert-threshold-percent=0.2
odin.execution.slippage.adjustment-threshold-percent=0.3
odin.execution.slippage.adjustment-window-size=10
odin.execution.slippage.adjustment-reduction-percent=25
odin.execution.slippage.max-per-trade-percent=0.5

# Runner-Trailing
odin.execution.trailing.rsi-overheat-threshold=80

# Pre-Market
odin.execution.premarket.min-spread-percent=0.5
odin.execution.premarket.min-volume=10000
odin.execution.premarket.max-position-percent=25
odin.execution.premarket.repricing-enabled=false

# Forced Close — Cross-Module Config Contract:
# Der Wert forcedCloseBufferMin wird NICHT in odin-execution definiert,
# sondern aus odin-rules konsumiert (odin.rules.exit.forced-close-buffer-min=30).
# Begründung: Die Entscheidung WANN forced-close greift ist eine Rules-Entscheidung (Kap 6).
# Mechanismus: Property wird ueber die Spring Environment aufgeloest
# (application.properties Import-Kaskade im App-Modul).
# KEINE Maven-Dependency, KEIN Code-Import: odin-execution hat weder Compile-
# noch Runtime-Abhaengigkeit auf odin-rules. Die Property ist ein reiner
# String-Key im Spring Environment — kein Modul-Coupling.

# Pacing-Guard
odin.execution.pacing.max-cancel-replace-per-minute=6

# Degradation (MarketClock-basierte Fenster)
odin.execution.degradation.cancel-replace-rate-threshold=5
odin.execution.degradation.cancel-replace-lookback-min=30
odin.execution.degradation.order-trade-ratio-threshold=10
odin.execution.degradation.order-trade-lookback-min=60
odin.execution.degradation.fill-rate-threshold-percent=30
odin.execution.degradation.fill-rate-min-attempts=5

# Transaktionskosten
odin.execution.costs.commission-per-share=0.0035
odin.execution.costs.fx-cost-bps=2
```

---

## 13. Abhaengigkeiten und Schnittstellen

### Konsumiert

- `TradeIntent` (von Decision Arbiter, Kap 6) — Entry/Exit-Absicht. Enthaelt: instrumentId, direction, entryPrice, stopLevel, targetPrice, quantScore, regime, patternState, llmAnalysisId, runId
- `BrokerGateway` (Port aus odin-api, Kap 3) — `submitOrder(OrderIntent) → brokerOrderId`, `cancelOrder(brokerOrderId)`, `modifyOrder(brokerOrderId, newParams)` (Port-Capability, Fallback: Cancel+Replace). Async BrokerEvent-Callbacks (Accepted, Rejected, Filled, PartiallyFilled, Cancelled)
- `MarketClock` (Port aus odin-api) — Timing fuer Repricing, Forced-Close, Reconciliation, Slippage-Fenster
- `MarketSnapshot` (von odin-data, Kap 2) — Aktuelle Preise fuer Repricing und Trailing-Stop-Berechnung
- `IndicatorResult` (von KPI-Engine, Kap 4) — EMA(9), RSI, Bollinger fuer Runner-Trailing. warmupComplete-Flag (keine Execution bei Warmup)
- `AccountRiskState` (von odin-core) — Globaler Tages-Risk-Status (remaining_budget, realisierte P&L, Exposure, Round-Trip-Counter, Cooling-Off-Status)
- `RunContext` (aus odin-api) — runId als Join-Key

### Produziert

- `FillEvent` (ins EventLog, non-droppable) — Gefuellte Order mit Preis, Stueckzahl, Slippage, runId
- `PositionUpdate` (an odin-core / AccountRiskState) — Aktuelle Position (Stueckzahl, avg. Entry, unrealisierte P&L)
- `OrderStatusEvent` (ins EventLog, non-droppable) — Status-Aenderungen (SUBMITTED, FILLED, CANCELLED, REJECTED, MARKET_ORDER_USED)
- `RejectEvent` (ins EventLog, non-droppable) — Risk Gate Rejects mit RejectReason-Code (RISK_EXPOSURE_EXCEEDED, RISK_BUDGET_EXHAUSTED, RISK_RR_INSUFFICIENT), emitter = RISK_GATE

### Nicht in Scope

- Rules Engine / Decision Arbiter → Kap 6
- KPI-Berechnung → Kap 4
- LLM-Integration → Kap 5
- Broker-Anbindung (IB-spezifisch) → Kap 3
- Global Risk Manager → Kap 0 (odin-core)
