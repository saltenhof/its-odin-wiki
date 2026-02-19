# ODIN — Kapitel 10: Deployment & Betrieb (Windows, On-Prem)

Version: 0.6 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2. Cross-Review R1+R2. DDD-Modulschnitt: Retention-Job → odin-audit, AuditEventDrainer → AuditEventDrainer

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt Deployment, Betrieb und operationelle Sicherheit von ODIN auf einer lokalen Windows-Maschine. Es umfasst Startup/Shutdown, Kill-Switch, Monitoring, Security-Hardening, Logging und Operator-Runbooks.

**Modulzuordnung:** Querschnittlich (primaer `odin-app`, `odin-core`)

**Abgrenzung — kein Duplikat:**
- Transaktionskosten → Kap 7, §10
- LLM-Response-Caching (CachedAnalyst) → Kap 5, §3
- Repricing-Degradation / Market Surveillance → Kap 7, §7
- Reconciliation-Algorithmus → Kap 7, §6
- Datenmodell / Retention → Kap 8
- Frontend-Endpoints → Kap 9

---

## 2. Betriebsumgebung

| Aspekt | Spezifikation |
|--------|---------------|
| OS | Windows 11 |
| Java | JDK 21 (LTS) |
| Datenbank | PostgreSQL 17, lokal |
| Broker | IB Gateway (headless), lokal, Port 4002 |
| Netzwerk | Localhost (DB, IB Gateway). Ausgehend: LLM-APIs (HTTPS) |
| Ressourcen | Dedizierter Rechner (kein Shared-System waehrend Handelszeiten) |

---

## 3. Prozesslandschaft

```
┌─────────────────────────────────────────────────┐
│                  Windows 11                      │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ IB Gateway   │  │ PostgreSQL   │             │
│  │ (headless)   │  │ Service      │             │
│  │ Port 4002    │  │ Port 5432    │             │
│  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                      │
│         └────────┬────────┘                      │
│                  │                               │
│         ┌────────┴────────┐                      │
│         │   ODIN JVM      │                      │
│         │   (java -jar)   │                      │
│         │   Port 8080     │──→ LLM APIs (HTTPS)  │
│         └────────┬────────┘                      │
│                  │                               │
│         ┌────────┴────────┐                      │
│         │   Watchdog      │                      │
│         │   (separater    │                      │
│         │    Prozess)     │                      │
│         └─────────────────┘                      │
└──────────────────────────────────────────────────┘
```

---

## 4. Startup-Sequenz

### Voraussetzungen (vor ODIN-Start)

| Voraussetzung | Pflicht? | Bei Fehler |
|---------------|---------|-----------|
| PostgreSQL-Service laeuft | Pflicht | Fail-Fast (kein Start) |
| IB Gateway gestartet und eingeloggt | Pflicht (Live-Modus, Normalbetrieb) | Fail-Fast. Im Sim-Modus: BrokerSimulator statt IB Gateway. Im Safe-Mode nach Crash: Optional — DEGRADED Safe-Mode ohne Broker (Dashboard/Forensik verfuegbar, Reconciliation pending) |
| LLM-API erreichbar | Optional | Start mit regimeConfidence=0.0 (Entries blockiert, Kap 5 §4). Warnung + Alert |

### ODIN-Start (Live-Modus)

```
1. JVM-Start (java -jar odin-app.jar)
   │
2. Spring Boot Initialization
   │  - Properties laden (Kaskade: application → odin-core → Module, CSpec)
   │  - DataSource validieren (Fail-Fast bei DB-Fehler)
   │  - Bean-Assembly
   │
3. MarketClock initialisieren (Bootstrap-Clock)
   │  - SystemMarketClock erzeugen (Exchange-TZ aus Config)
   │  - tradingDate aus MarketClock ableiten (NICHT aus Systemzeit)
   │  - MarketClock ist ab hier verfuegbar fuer RunContext-Erzeugung
   │
4. Crash-Check (Safe-Mode-Guard, §5)
   │  - Letzten TradingRun fuer tradingDate laden
   │  - Crash erkannt wenn: outcome=ERROR AND tradingDate=heute, ODER endTime IS NULL
   │  - Bei Crash: abgebrochenen Run auf outcome=ERROR + endTime setzen
   │    → Safe-Mode aktivieren (SAFE_MODE_UNTIL_NEXT_SESSION)
   │    → Weiter mit Schritt 5 (BrokerGateway), aber KEIN TradingRun anlegen,
   │      KEIN Decision Loop, KEIN Trading
   │
5. BrokerGateway-Verbindung herstellen (odin-broker)
   │  - Socket-Connect zu IB Gateway (Port aus Config)
   │  - Heartbeat starten
   │  - [Normal]: Fehler → Retry (max. 5x, Backoff) → Abbruch
   │  - [Safe-Mode]: Fehler → Retry (max. 5x, Backoff) → DEGRADED Safe-Mode
   │    (kein Abbruch — Dashboard/Forensik/EventLog verfuegbar, Reconciliation pending)
   │
6. Pipelines initialisieren (odin-core, PipelineFactory)
   │  - [NUR wenn NICHT Safe-Mode]:
   │    - Pro konfiguriertem Instrument: Pipeline erzeugen
   │    - RunContext erstellen (runId, mode=LIVE, instrumentId, exchange,
   │      tradingDate aus MarketClock, promptVersion, modelId, configSnapshotId)
   │    - TradingRun in DB anlegen (Kap 8)
   │    - Historical Data laden (odin-data)
   │    - State Recovery: letzten TradingRun laden, offene Trades pruefen (Kap 8 §6)
   │    - Broker-Reconciliation ausfuehren (Kap 7 §6)
   │    - Pipeline-State: STARTUP → WARMUP (Indikatoren aufbauen, Kap 4)
   │  - [Safe-Mode]:
   │    - Broker-Reconciliation ausfuehren wenn BrokerGateway verbunden (Operator-Sichtbarkeit)
   │    - DEGRADED Safe-Mode (kein Broker): Reconciliation pending, Dashboard zeigt Broker-Status
   │    - Keine Pipelines erzeugen, kein TradingRun anlegen
   │
7. Echtzeit-Subscriptions starten (odin-data)
   │  - [NUR wenn NICHT Safe-Mode]: MarketDataFeed: Realtime Bars, Ticks, DQ-Gates
   │  - [Safe-Mode]: Keine Subscriptions
   │
8. Frontend-Server starten
   │  - SSE-Endpoints aktiv (/api/stream/instruments/{instrumentId}, /api/stream/global)
   │  - REST-Endpoints aktiv
   │  - [Safe-Mode]: Dashboard zeigt SAFE_MODE-Banner + Reconciliation-Ergebnis (oder pending bei DEGRADED)
   │
9. Agent READY
   │  - [Normal]: Pipelines durchlaufen WARMUP → OBSERVING (nach warmupComplete, Kap 4/6)
   │    Decision Loop beginnt
   │  - [Safe-Mode]: Agent laeuft, aber kein Trading. Nur Dashboard/Forensik/Reconciliation
```

> **MarketClock-Primat:** Die MarketClock wird in Schritt 3 initialisiert, **bevor** RunContext/TradingRun erzeugt werden. `tradingDate` wird aus der MarketClock abgeleitet (Exchange-TZ-basiert), nie aus `LocalDate.now()` oder Systemzeit. Damit ist der Grundsatz "MarketClock = sole time source" ab dem ersten geschaeftsrelevanten Zeitstempel eingehalten.

### Startup (Sim-Modus)

```
1-2. JVM + Spring Boot (identisch)
3.   MarketClock initialisieren (SimulatedMarketClock, vom Runner gesteuert)
4.   BrokerSimulator statt IB Gateway (kein Socket-Connect)
5.   SimulationRunner erzeugt RunContext pro simuliertem Tag
     - mode=SIMULATION, frischer State (NO_POSITION)
     - tradingDate aus SimulatedMarketClock
     - Kein State Recovery (jeder Tag startet leer)
6.   Historical Data aus DB laden (kein Live-Feed)
7-8. Frontend optional (SSE-Streams zeigen Sim-Daten)
```

### Shutdown-Sequenz

```
1. Shutdown-Signal empfangen (Service-Stop, Kill-Switch, oder manuell)
   │  (Windows: Service Control Manager "Stop" oder WinSW; kein SIGTERM im POSIX-Sinne)
   │
2. Pipelines stoppen
   │  - Keine neuen TradeIntents
   │  - Offene Positionen: Forced-Close per Market-Order (wenn waehrend RTH)
   │  - GTC-Stops bleiben beim Broker aktiv (Crash-Recovery, Kap 0 §10)
   │  - Alle MarketDataFeed-Subscriptions beenden
   │
3. State persistieren
   │  - EventLog-Spool vollstaendig drainen (AuditEventDrainer, Kap 8 §2)
   │  - TradingRun.endTime setzen, Outcome → COMPLETED (oder ABORTED/ERROR)
   │
4. BrokerGateway-Verbindung schliessen
   │
5. Spring Boot Shutdown
   │  - DataSource schliessen
   │  - JVM-Exit
```

### TradingRun-Outcome vs Pipeline-State

Pipeline-State (Kap 6 FSM) und TradingRun-Outcome sind getrennte Konzepte:

| Konzept | Werte | Quelle |
|---------|-------|--------|
| Pipeline-State (Kap 6 FSM) | STARTUP, WARMUP, OBSERVING, POSITIONED, PENDING_FILL, HALTED, DAY_STOPPED | PipelineFSM, transient, live im Speicher |
| TradingRun Outcome | COMPLETED, ABORTED, ERROR | TradingRun-Entitaet (Kap 8), persistiert bei Run-Ende |

- **COMPLETED:** Normaler oder kontrollierter Abschluss (Pipeline terminal in DAY_STOPPED)
- **ABORTED:** Sicherheitsverletzung (Pipeline terminal in HALTED). Operator-Eingriff war erforderlich
- **ERROR:** Unkontrollierter Abbruch (Crash, unbehandelte Exception). TradingRun hat initial kein endTime — Startup-Recovery setzt `outcome=ERROR` und `endTime` nachtraeglich und aktiviert den Safe-Mode (§5). Bei wiederholtem Restart am selben Tag wird ERROR ueber `outcome=ERROR AND tradingDate=heute` erkannt (nicht nur ueber `endTime IS NULL`)

> **Detailliertes Mapping siehe §6 "Pipeline-Terminal-States und ihre Trigger".**

---

## 5. Windows Service und Watchdog

### Windows Service

ODIN wird als **Windows Service** betrieben (nicht als manuell gestarteter Prozess):

| Aspekt | Umsetzung |
|--------|-----------|
| Service-Wrapper | WinSW (leichtgewichtig, XML-Config) |
| Auto-Start | Service startet bei System-Boot |
| Restart-Policy | Automatischer Restart bei Absturz (max. `maxRestartsPerHour`, Default: 3). **Safe-Mode bei Intraday-Crash** (siehe unten) |
| Logging | Service-Logs nach `${ODIN_HOME}/logs/` |

### Safe-Mode nach Intraday-Crash

WinSW startet den ODIN-Prozess bei Absturz automatisch neu. Damit kein unkontrolliertes Weiterhandeln erfolgt, erzwingt die **Startup-Recovery** bei erkanntem Intraday-Crash einen **Safe-Mode**:

1. **Erkennung:** Startup-Recovery prueft den letzten TradingRun fuer den aktuellen Handelstag. Crash erkannt wenn: `outcome=ERROR` (vorheriger Crash bereits klassifiziert) ODER `endTime IS NULL` (noch nicht klassifizierter Crash). Damit funktioniert die Erkennung auch bei wiederholtem WinSW-Restart am selben Tag
2. **Crash-Run abschliessen:** Abgebrochener Run erhaelt `outcome=ERROR` und `endTime` = aktueller Zeitpunkt (falls noch NULL)
3. **Safe-Mode aktivieren:** Systemweites **globales Gate** `SAFE_MODE_UNTIL_NEXT_SESSION` (in odin-core). Das Gate blockiert Decision Loop, OMS-Actions und neue TradeIntents. Es ist **kein Pipeline-State** — Pipelines werden im Safe-Mode gar nicht erzeugt (Kap 10, §4 Startup-Sequenz)
4. **Was funktioniert:** Dashboard, REST-API, EventLog-Forensik, Metriken. Broker-Reconciliation wenn IB Gateway erreichbar (sonst DEGRADED Safe-Mode: Reconciliation pending, Operator-UI trotzdem verfuegbar)
5. **Was gesperrt ist:** Pipeline-Erzeugung, Decision Loop, neue TradeIntents, Entry/Exit-Signale, MarketData-Subscriptions. Kein Trading bis zur naechsten regulaeren Session
6. **Kein neuer TradingRun:** Im Safe-Mode wird kein neuer TradingRun angelegt — der gecrashe Run ist der letzte fuer diesen Handelstag
7. **Operator-Sichtbarkeit:** Dashboard zeigt SAFE_MODE-Banner mit Crash-Zeitpunkt und Reconciliation-Ergebnis (oder pending bei DEGRADED Safe-Mode). CRITICAL-Alert an Operator

> **Konsistenz mit Kap 0, §10:** "ODIN ist nicht intraday-restartable fuer Trading" bedeutet: Der Prozess wird neu gestartet (WinSW), aber der Agent handelt nicht weiter (Safe-Mode). GTC-Stops beim Broker schuetzen offene Positionen unabhaengig vom JVM-Zustand.

### Watchdog-Prozess

**Rollenabgrenzung WinSW vs Watchdog:** WinSW ist der reine Prozess-Restart-Mechanismus (JVM abgestuerzt → neu starten). Der Watchdog ist ein **Application-Health- und Environment-Monitor** mit erweiterter Prueflogik, die ueber Prozess-Existenz hinausgeht:

| Pruefung | Intervall | Reaktion | Warum nicht WinSW |
|----------|-----------|---------|-------------------|
| HTTP Health-Check (`/api/health`) | Alle 30s | 3x Failure → Kill + Restart | WinSW erkennt nur Prozess-Tod, nicht Application-Hangs |
| IB Gateway laeuft? | Alle 30s | Nicht laufend → Alert (kein Auto-Restart — erfordert manuellen Login) | Fremdprozess, nicht von WinSW verwaltet |
| Disk-Space | Alle 60s | < 10% frei → WARNING Alert | Environment-Check, keine WinSW-Kompetenz |
| Zeitsynchronisation | Alle 60s | IB-Serverzeit vs Systemzeit: >1s → WARNING, >5s → CRITICAL | Geschaeftslogik-relevanter Check |

> **Harter Kill und Position-Sicherheit:** Wenn der Watchdog die JVM hart killt, koennen keine Cancel/Market-Close-Orders mehr gesendet werden. Schutz: (1) Stop-Orders als GTC beim Broker aktiv (Kap 7, §5) — ueberleben JVM-Crash. (2) Startup-Reconciliation (Kap 7 §6, Kap 8 §6): offene Orders pruefen, Orphans stornieren, Positionen gegen Broker abgleichen. (3) Operator-Runbook (§12).

---

## 6. Kill-Switch (Multi-Layer)

### Vier Kill-Switch-Ebenen

| Ebene | Ausloeser | Mechanismus |
|-------|-----------|-------------|
| 1. Software (intern) | Heartbeat-Failure, Stale-Data, Hard-Stop erreicht, EventLog-Spool-Overflow (ESCALATE) | odin-core → DAY_STOPPED, Forced-Close aller Positionen |
| 2. UI/REST | Manueller Button im Dashboard, oder `POST /api/controls/kill` (Kap 9 §3) | REST → odin-core → Kill-Entscheidung (ACCEPTED/REJECTED/NOOP) |
| 3. Watchdog | JVM-Anomalie (Health-Check-Failure) | Watchdog → Process-Kill → Broker-seitige GTC-Stops greifen |
| 4. Broker-seitig | IB-Account-Limits | IB Gateway liquidiert autonom bei Account-Limit-Verletzung |

### Kill-Switch-Semantik

Bei Ausloesung durch odin-core (Ebene 1 oder 2):
1. Alle offenen Limit-/Target-Orders stornieren
2. Alle Positionen per Market-Order schliessen (Kap 7, §4: Market-Order-Policy)
3. Alle Pipeline-FSMs → DAY_STOPPED (Kap 6)
4. Keine neuen Subscriptions, keine neuen TradeIntents
5. EventLog: Kill-Switch-Trigger dokumentieren (non-droppable)
6. Alert: CRITICAL an Operator (Kap 9 §4.4)

### Pipeline-Terminal-States und ihre Trigger (Klarstellung)

| Terminal-State | Trigger | Semantik |
|---------------|---------|----------|
| **DAY_STOPPED** | Normaler Tagesablauf (EOD Forced Close), Kill-Switch (Ebene 1+2: Heartbeat-Failure, Stale-Data, Hard-Stop, EventLog-ESCALATE, manueller Kill), Watchdog-Kill (Ebene 3, nachtraeglich gesetzt bei Recovery) | Kontrollierter Stop. Forced Close wurde ausgefuehrt (oder versucht). Positionen sind geschlossen oder durch GTC-Stops gesichert. |
| **HALTED** | Sicherheitsinvariante gebrochen: Position ohne bestaetigten GTC-Stop, GTC-Stop unerwartet storniert, Broker-Integritaetsverletzung | Trading sofort gesperrt, KEINE automatische Forced Close (Operator muss Situation bewerten). Positionen koennen noch offen sein — GTC-Stops beim Broker sollten weiterhin greifen. Operator-Eskalation zwingend. |

**Outcome-Mapping:**

| Pipeline-Terminal-State | TradingRun-Outcome | Bedeutung |
|------------------------|-------------------|-----------|
| DAY_STOPPED | COMPLETED | Normaler oder kontrollierter Abschluss |
| HALTED | ABORTED | Sicherheitsverletzung, Operator-Eingriff erforderlich |
| (kein Terminal-State — Crash) | ERROR | TradingRun hat kein endTime (wird bei Startup-Recovery nachtraeglich gesetzt). Erkannt bei Startup-Recovery ueber `endTime IS NULL` oder `outcome=ERROR` fuer aktuellen Handelstag. Triggert Safe-Mode (§5) |

**Latenz-SLO:** Kill-Switch-Ausloesung bis Order-Submission: < 2 Sekunden **bei aktiver IB-Verbindung und normalem Pacing.** Bei Disconnect: Best-Effort — GTC-Stops beim Broker greifen als Fallback unabhaengig von der JVM (Kap 0, §10).

### EventLog-Spool-Overflow-Semantik

Der EventLog-Spool ist bounded (in-memory). Die Droppability-Policy ist **eventklassenabhaengig** (Kap 2, §8): MarketEvents (Ticks, L2-Updates) sind **Best-effort** und duerfen bei Ueberlast vor/bei Enqueue gesampelt oder gedroppt werden. Alle anderen Eventklassen (BarClose, MarketSnapshot-Referenz, DQ-Events, OMS-Events) sind **non-droppable**. Die Overflow-Semantik bezieht sich auf die non-droppable Events:

| Phase | Schwelle | Verhalten |
|-------|----------|-----------|
| Normal | Spool < 80% | Asynchron: Produzent schreibt in Spool, AuditEventDrainer pollt und persistiert |
| Warning | Spool >= 80% | WARNING-Alert. Drainer beschleunigt (kuerzeres Poll-Intervall) |
| ESCALATE | Spool >= 95% | ESCALATE an odin-core → DAY_STOPPED, Forced Close. Keine neuen TradeIntents oder Business-Aktionen. Die Stop-Sequenz selbst (Trigger, Cancels, Submits, Acks/Fills, Outcome) erzeugt weiterhin garantiert Events — dafuer ist die 5%-Reserve vorgesehen |
| Backpressure | Spool 100% (sollte nach ESCALATE nicht eintreten) | Produzent blockiert kurz (max. 500ms), bis Drainer Platz schafft. Timeout → CRITICAL Alert + Prozess-Exit (ERROR-Outcome). WinSW startet Prozess neu → **Safe-Mode** greift (§5), kein Trading |

**Warum dieses Modell:** Die ESCALATE-Schwelle bei 95% stellt sicher, dass noch Kapazitaet fuer die Events der Stop-Sequenz (Kill-Switch-Trigger, Forced-Close-Orders, Acks/Fills) verbleibt. Nach DAY_STOPPED werden keine neuen Business-Aktionen initiiert, aber die Stop-Sequenz laeuft zu Ende und ihre Events werden garantiert geloggt (non-droppable). Der Drainer arbeitet den Restbestand ab. Ein tatsaechlicher 100%-Overflow ist ein Indikator fuer einen schweren Fehler (Drainer haengt, DB nicht erreichbar) und fuehrt zum Prozess-Exit mit ERROR-Outcome. WinSW startet den Prozess automatisch neu, aber die Startup-Recovery erkennt den Intraday-Crash (TradingRun mit `outcome=ERROR` oder `endTime IS NULL` fuer den aktuellen Handelstag) und erzwingt den **Safe-Mode** (§5) — kein Trading, nur Dashboard/Reconciliation/Forensik.

### Position-Safety-Invariante

**Harte Betriebsregel:** Jede offene Position (Pipeline-State POSITIONED oder PENDING_FILL) MUSS zu jedem Zeitpunkt einen beim Broker bestaetigten GTC-Stop haben (Kap 7, §5). Diese Invariante ist Voraussetzung dafuer, dass ein harter Kill (Watchdog, Crash) keine ungeschuetzten Positionen hinterlaesst.

| Situation | Reaktion |
|-----------|---------|
| GTC-Stop-Submission schlaegt fehl | Kein Entry (Trade wird nicht eroeffnet, Kap 7 §4) |
| Bestaetigter GTC-Stop wird unerwartet storniert | CRITICAL Alert + Pipeline → HALTED. Operator muss Position manuell pruefen |
| Positioned ohne bestaetigten GTC-Stop erkannt (Monitoring) | Sofort CRITICAL Alert. Pipeline → HALTED bis Operator eingreift |

> **Monitoring-Metrik:** `odin_positions_without_stop` (Gauge). Jeder Wert > 0 ist ein CRITICAL Alert. Diese Metrik wird bei jedem OMS-State-Change aktualisiert.

---

## 7. Monitoring und Observability

### Metrik-Exposition

| Aspekt | v1-Umsetzung |
|--------|-------------|
| Framework | Micrometer (Spring Boot Actuator) |
| Endpoint | `/actuator/metrics` (nur loopback, nicht extern exponiert) |
| JMX | Aktiviert (lokaler Zugriff via JConsole/VisualVM fuer Debugging) |
| Prometheus/Grafana | Nicht v1-Scope. Optional: `/actuator/prometheus` bei Bedarf aktivierbar |
| Distributed Tracing | Nicht v1-Scope (Single-JVM, keine Notwendigkeit) |

### Metriken (Micrometer/JMX) — v1-Baseline

| Metrik | SLO | Messung |
|--------|-----|---------|
| E2E-Latenz (Bar-Close → Order-Submit) | < 100ms (p99, ohne LLM) | Histogram pro Pipeline |
| LLM Response Time | < 10s (p95) | Timer pro Call |
| DQ Drop Rate | < 1% pro Session | Counter pro Pipeline |
| EventLog Spool Depth | < 50 Events | Gauge |
| Order Roundtrip (Submit → Ack) | < 500ms | Timer |
| GC Pause | < 50ms (p99) | JVM Metrics |
| Positions without Stop | 0 (CRITICAL bei > 0) | Gauge (odin_positions_without_stop) |
| System Uptime (RTH) | 99.5% pro Monat | Heartbeat |

### Alert-Routing

| Level | Beispiele | Routing |
|-------|----------|---------|
| INFO | Tages-Report, Pipeline-State-Change | Dashboard (SSE) |
| WARNING | Hohe Slippage, LLM-Retry, DQ-Verletzung | Dashboard + Push (Ntfy/Pushover) |
| CRITICAL | Kill-Switch, LLM-Ausfall, Feed-Ausfall, EventLog-Overflow | Dashboard + Push + E-Mail |

> **v1-Pragmatik:** Dashboard (SSE) + Push-Notification (Ntfy/Pushover, HTTPS-basiert) sind ausreichend. E-Mail fuer EOD-Reports und CRITICALs. SMS/Call als optionale Erweiterung (nicht v1-Scope). Alle Transports sind HTTPS-basiert (outbound), kein Inbound-Port noetig.

---

## 8. Security-Hardening

### Netzwerk

| Massnahme | Details |
|-----------|---------|
| Outbound-Whitelist | IB Gateway (localhost), LLM-APIs (FQDN), Push-Service (ntfy/pushover FQDN), SMTP-Host (wenn E-Mail enabled), NTP-Server (w32time). Alle ueber HTTPS/TCP, keine weiteren Outbound-Ziele |
| Firewall | Windows Firewall: Inbound nur Port 8080 (loopback). Outbound auf Whitelist beschraenkt. Windows-Updates: separates Maintenance-Window (nicht waehrend RTH) |
| Kein Remote-Zugriff | Dashboard nur via localhost erreichbar (Kap 9 §3: Localhost-Scope). Remote bei Bedarf via RDP/VPN (nicht ODIN-seitig) |

### Secrets

| Massnahme | Details |
|-----------|---------|
| API-Keys | Ausschliesslich via Environment-Variablen (Kap 8 §9: ${ODIN_DB_USER} etc.) |
| Rotation | LLM-API-Keys alle 30 Tage. Separate Keys pro Environment |
| Logging | Keine Secrets in Logs. LLM-Prompts ohne Credentials/Account-Infos |

### System

| Massnahme | Details |
|-----------|---------|
| Least Privilege | ODIN-Service laeuft als dedizierter Windows-User (kein Admin) |
| Update-Policy | Windows-Updates ausserhalb der Handelszeiten (geplanter Reboot) |

---

## 9. Logging

### Structured Logging (JSON)

Alle Log-Eintraege im JSON-Format fuer maschinelle Auswertung:

```json
{
  "timestamp": "2026-02-14T15:30:00.123Z",
  "level": "INFO",
  "logger": "de.odin.brain.rules.RulesEngine",
  "instrumentId": "AAPL",
  "runId": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Entry intent generated",
  "regime": "TREND_UP",
  "quantScore": 0.72
}
```

> **MarketClock vs Log-Timestamp:** Der `timestamp` im Log ist die **Wanduhr** (fuer Ops-Debugging). Geschaeftsrelevante Zeitstempel (Decision-Bar, Fill-Time etc.) kommen aus der MarketClock und stehen in den EventLog-Events (Kap 8 §4: marketClockTimestamp).

> **instrumentId in Logs:** In v1 ist `instrumentId` das Ticker-Symbol (z.B. "AAPL"). Falls spaeter eine numerische ID eingefuehrt wird, wird das Log-Format angepasst. Fuer Ops-Lesbarkeit ist das Symbol in Logs optimal.

### Log Rotation

| Aspekt | Regelung |
|--------|---------|
| Rotation | Taeglich + bei 100 MB |
| Retention | 30 Tage auf Disk |
| Komprimierung | Rotierte Logs werden gzip-komprimiert |
| Pfad | `${ODIN_HOME}/logs/` |

---

## 10. Tages-Lifecycle-Steuerung

Der `LifecycleManager` (odin-core) steuert den Tagesablauf. Alle Zeiten ueber **MarketClock** in **Exchange-TZ** (Kap 0, §3.2). Die Phasen sind konfigurierbar (§13), nicht hartcodiert.

| Phase | Default (ET, NYSE) | Aktionen |
|-------|---------------------|---------|
| Pre-Market | 07:00 – RTH-Open | System-Init, Historical Data, LLM-Tagesanalyse. Pipeline-State: STARTUP → WARMUP |
| Opening Calibration | RTH-Open – RTH-Open+15min | Keine Trades. ATR + Volume Baseline kalibrieren (Kap 4). Pipeline wartet auf warmupComplete |
| Active Trading | Calibration-Ende – forcedCloseBufferMin vor RTH-End | Decision Loop aktiv, Trades erlaubt. Pipeline-State: OBSERVING/POSITIONED/PENDING_FILL |
| No-New-Entries | Konfigurierbar (Default: 15:30 ET) | Keine neuen Entries. Bestehende Positionen: Exit-Rules weiterhin aktiv |
| Forced Close | RTH-Ende - forcedCloseBufferMin (Kap 7 §4) | Alle Positionen zwangsschliessen (Market-Order). Pipeline → DAY_STOPPED |
| EOD | Nach Forced Close | EventLog-Spool drainen, Reconciliation (Kap 7 §6), TradingRun abschliessen, Tages-Report |

> **Exchange-Kalender:** Feiertage und Early-Close-Tage werden ueber eine konfigurierbare Kalenderdatei gesteuert (`odin.lifecycle.holidays-file`).
>
> **Dateiformat (CSV):** `date,type,close_time` — Beispiel: `2026-12-25,CLOSED,` (ganzer Tag zu), `2026-11-27,EARLY_CLOSE,13:00` (frueherer Schluss). Spalten: ISO-8601-Datum, Typ (CLOSED|EARLY_CLOSE), optionale Close-Time (Exchange-TZ, nur bei EARLY_CLOSE).
>
> **Update-Prozess:** Jaehrlich vor Jahresbeginn aktualisieren (Exchange-Kalender sind vorab bekannt). Aenderung erfordert ODIN-Neustart (Datei wird beim Startup geladen und validiert). **Startup-Validierung:** Datei muss existieren und parsebar sein → Fail-Fast bei Fehler. Fehlende Eintraege fuer den aktuellen Tag → Warnung (Default: normaler Handelstag).

---

## 11. Zeitsynchronisation

| Aspekt | Regelung |
|--------|---------|
| NTP | Windows Time Service (w32time) mit externem NTP-Server. Max. tolerierter Drift: 500ms |
| Drift-Alarm | Watchdog prueft Systemzeit gegen IB-Serverzeit (§5, `BrokerGateway.requestCurrentTime()`). Abweichung > 1s → WARNING, > 5s → CRITICAL |
| DST | Alle Lifecycle-Zeiten in Exchange-TZ. DST-Wechsel wird von MarketClock (ZonedDateTime) automatisch gehandhabt |
| MarketClock-Integration | MarketClock (Kap 0) ist die Sole Time Source fuer alle Geschaeftslogik. Zeitsynchronisation stellt sicher, dass die MarketClock (SystemMarketClock im Live-Modus) auf einer praezisen Systemuhr basiert |

---

## 12. Backup und Disaster Recovery

| Aspekt | Regelung |
|--------|---------|
| DB-Backup | Taeglich (nach EOD) via `pg_dump`. Retention: 30 Tage lokal, 90 Tage auf externem Storage |
| Backup-Verschluesselung | AES-256, Key via ENV-Variable |
| Restore-Test | Monatlich: Restore auf Testinstanz, Datenintegritaet pruefen |
| RPO | 1 Handelstag (Backup nach EOD). Intraday: PostgreSQL WAL fuer Point-in-Time-Recovery |
| RTO | < 30 Minuten (Restore + Startup + Reconciliation) |

> **Keine Audit-Chain-Validierung.** v0.1 referenzierte eine Hash-Chain-basierte Audit-Validierung bei Restore. **Entfernt** gemaess Lean-Prinzip (Kap 8: kein Hash-Chain, kein HMAC). Datenintegritaet wird ueber PostgreSQL-Constraints und Reconciliation (Kap 7 §6) sichergestellt.

> **WAL/PITR:** PostgreSQL WAL-Archivierung fuer Point-in-Time-Recovery ist eine **optionale Erweiterung** (nicht v1-Baseline). In v1 reicht der taeglich pg_dump nach EOD. Falls PITR spaeter benoetigt wird: WAL-Archivierung auf externen Storage konfigurieren, Restore-Prozedur dokumentieren.

### Laufender DB-Betrieb

| Aspekt | Regelung |
|--------|---------|
| Autovacuum | PostgreSQL-Defaults. Keine manuelle VACUUM-Konfiguration in v1 |
| DB-Disk-Monitoring | Watchdog prueft Disk-Space (§5). Zusaetzlich: PostgreSQL-Tablespace-Groesse als Metrik (Gauge) |
| Connection-Pool | HikariCP (Spring Boot Default). Max 10 Connections (Single-JVM, kein Concurrency-Problem) |
| Retention-Job | Geplanter Job (EOD, nach Backup): SIM-Runs aelter als 90 Tage loeschen (odin-execution Repositories), EventRecords kuerzer als strukturierte Tabellen (Kap 8 §5, odin-audit). Als Spring `@Scheduled`-Task in odin-core (orchestriert Loeschung ueber die Domaenen-Repositories) |
| Index-Pflege | PostgreSQL-eigene Statistiken (pg_stat_user_tables). Kein manuelles REINDEX in v1; bei Performance-Problemen: EXPLAIN ANALYZE + gezielter Index |

---

## 13. Operator-Runbooks (Referenz)

| Szenario | Kernschritte |
|----------|-------------|
| IB Gateway Relogin | IB Gateway UI oeffnen → Login → ODIN erkennt Reconnect automatisch (BrokerGateway Heartbeat) |
| LLM-Outage | regimeConfidence=0.0 → Entries blockiert (Kap 5 §4). Keine manuellen Aktionen noetig. Pruefe Dashboard `llm-update` Events |
| DB Full | Disk-Alert pruefen → alte Logs rotieren → ggf. Retention-Job manuell triggern |
| Stale Data | Dashboard `alert` pruefen → IB Gateway Status → ggf. Reconnect/Restart |
| Reconcile nach Crash (Safe-Mode) | ODIN startet automatisch im Safe-Mode (§5). Dashboard zeigt SAFE_MODE-Banner + Reconciliation-Ergebnis (oder pending bei DEGRADED). **Offene Positionen pruefen:** GTC-Stops beim Broker aktiv? Positionen manuell in IB Gateway/TWS schliessen falls noetig (ODIN handelt im Safe-Mode nicht). Bei DEGRADED (kein Broker): IB Gateway manuell starten/einloggen, dann ODIN neu starten fuer vollstaendige Reconciliation. Naechster regulaerer Tagesstart: Flat-Check via Reconciliation. Falls nicht flat: CRITICAL Alert, Operator muss Position vor Handelsbeginn manuell schliessen |
| Sim → Live Switch | `odin.agent.mode=LIVE` setzen. Neuer Start. Kapital + Risk-Limits validieren. IB Gateway muss laufen |
| Forced Close Failure | Dashboard pruefen → manuell im IB Gateway/TWS Position schliessen → ODIN-State korrigiert sich beim naechsten Start (Reconciliation) |

---

## 14. Konfiguration

```properties
# application.properties (odin-app)

# Server
server.port=8080

# Pipeline-Konfiguration (instrumentId als Pipeline-Identifier, 1:1 in v1)
odin.agent.instruments[0]=AAPL
odin.agent.instruments[1]=MSFT

# Tageskapital
odin.agent.daily-capital-eur=10000

# Betriebsmodus
odin.agent.mode=LIVE
# Enum: LIVE | SIMULATION (kein PAPER, kein SIM — einheitlich in allen Kapiteln)

# Lifecycle-Zeiten (Exchange-TZ, Defaults fuer NYSE)
odin.lifecycle.timezone=America/New_York
odin.lifecycle.pre-market-start=07:00
odin.lifecycle.calibration-duration-min=15
odin.lifecycle.no-new-entries-before-close-min=30
odin.lifecycle.holidays-file=${ODIN_HOME}/config/holidays.csv

# Watchdog
odin.watchdog.health-check-interval-s=30
odin.watchdog.max-restarts-per-hour=3

# Logging
logging.file.path=${ODIN_HOME}/logs
logging.file.max-size=100MB
logging.file.max-history=30

# Alert-Transport
odin.alerts.push.enabled=true
odin.alerts.push.service=ntfy
odin.alerts.push.url=${ODIN_PUSH_URL}
odin.alerts.email.enabled=true
odin.alerts.email.smtp-host=${ODIN_SMTP_HOST}
odin.alerts.email.from=${ODIN_ALERT_EMAIL_FROM}
odin.alerts.email.to=${ODIN_ALERT_EMAIL_TO}
```

---

## 15. Abhaengigkeiten und Schnittstellen

### Konsumiert (querschnittlich)

- Alle Module (odin-api, odin-core, odin-data, odin-brain, odin-broker, odin-execution, odin-audit, odin-persistence, odin-frontend) — als Spring Boot Application zusammengefuehrt in odin-app
- PostgreSQL (lokal) — Persistence (Kap 8)
- IB Gateway (lokal) — BrokerGateway (Kap 3)
- LLM-APIs (extern, HTTPS) — LlmAnalyst (Kap 5)

### Bereitgestellt

- ODIN als lauffaehiges System (java -jar odin-app.jar)
- Windows Service + Watchdog
- Operator-Dashboard (Frontend, Kap 9)
- Alert-Transport (Push, E-Mail)
