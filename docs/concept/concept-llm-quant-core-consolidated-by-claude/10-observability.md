# 10 -- Observability, Monitoring, Logging, Audit, Dashboard, Alerting

---

## 1. Grundprinzip (Normativ)

ODIN MUSS jederzeit vollstaendig transparent und reproduzierbar sein. Jede Entscheidung, jeder Zustandswechsel und jeder Broker-Event MUSS nachvollziehbar protokolliert werden. Das System erzeugt keine "Black-Box"-Trades -- jeder Trade ist von der ersten Bar bis zum letzten Fill rueckverfolgbar.

Drei Saeulen der Observability:

1. **Echtzeit-Monitoring** -- Live-Streams fuer Operator und Dashboard
2. **Audit-Trail** -- Tamper-proof, append-only, hash-verkettet
3. **Reporting** -- Aggregierte Auswertung pro Trade, pro Tag, langfristig

---

## 2. Monitoring-Streams (Normativ)

### 2.1 Per-Pipeline-Stream

Das System MUSS pro Pipeline in Echtzeit bereitstellen:

| Datenkategorie | Felder |
|----------------|--------|
| **Pipeline State** | FSM-Zustand, cycleNumber, sessionPhase |
| **Kurs/Preis** | Aktueller 1m-Bar (OHLCV), VWAP |
| **Indikatoren** | EMA9, EMA21, EMA50, RSI14, ATR14, ADX14, volume_ratio, dist_to_vwap_atr |
| **Position** | Stueckzahl, Entry-Preis, unrealized P&L, realized P&L, MFE, MAE |
| **Orders** | Offene Orders (Stop, Targets, Entry), Status, Repricing-Zyklen |
| **LLM-Status** | Letzter Call (Timestamp, Latenz, Validation), TTL-Remaining, Degradation-Flag |
| **Regime** | Aktuelles Regime-Label, Confidence, Subregime, DayType-Flags |
| **Monitor-Events** | Aktive Events (PARABOLIC_ACCEL, EXHAUSTION_SIGNAL, SHOCK_DOWN, etc.) |

### 2.2 Globaler Stream

Das System MUSS global bereitstellen:

| Datenkategorie | Felder |
|----------------|--------|
| **Account Risk State** | remaining_budget, realisierte Verluste, unrealisierte Verluste, Exposure |
| **Aggregierte P&L** | Gesamt-P&L ueber alle Pipelines, pro Pipeline |
| **Kill-Switch State** | Aktiv/Inaktiv, letzter Trigger, Grund |
| **Trade-Counter** | round_trips_today (global), cycles per pipeline |
| **System-Health** | Heartbeat-Status, Datafeed-Status, Broker-Verbindung, LLM-Availability |
| **Alerts** | Aktive Alerts mit Level und Timestamp |

### 2.3 Kommunikationsmodell

| Protokoll | Zweck | Richtung |
|-----------|-------|----------|
| **SSE** | Monitoring-Streams (Pipeline-State, Kurse, P&L, Alerts) | Server -> Client |
| **REST POST** | Controls (Kill-Switch, Pause/Resume) | Client -> Server |

Kein WebSocket. SSE fuer unidirektionales Streaming, REST fuer seltene Control-Aktionen.

**SSE-Endpoints:**

| Endpoint | Daten |
|----------|-------|
| `/api/stream/instruments/{instrumentId}` | Pipeline-State, Kurs, Indikatoren, P&L, Orders, LLM-Status |
| `/api/stream/global` | Globaler Risk-Status, aggregierte P&L, Kill-Switch-State, Alerts |

**REST-Endpoints:**

| Endpoint | Zweck |
|----------|-------|
| `POST /api/controls/kill` | Kill-Switch (global) |
| `POST /api/controls/pause/{instrumentId}` | Pipeline pausieren |
| `POST /api/controls/resume/{instrumentId}` | Pipeline fortsetzen |
| `GET /api/runs/{runId}` | TradingRun-Details |
| `GET /api/runs/{runId}/decisions` | DecisionLog |
| `GET /api/runs/{runId}/llm-history` | LLM-Call-Historie |

---

## 3. Control-Interface (Normativ)

Minimale Controls die das System bereitstellen MUSS:

| Control | Scope | Beschreibung |
|---------|-------|--------------|
| `kill` | Global | Kill-Switch: Alle Positionen schliessen, alle Orders stornieren, DAY_STOPPED |
| `pause` | Per Pipeline | Pipeline pausiert: Keine neuen Orders, bestehende Position wird gehalten |
| `resume` | Per Pipeline | Pipeline nimmt Handel wieder auf |
| `safe_mode` | Global | Read-Only: System zeigt Daten an, handelt aber nicht |

---

## 4. Logging-Standards (Normativ)

### 4.1 Structured Logging

Alle Logs MUESSEN strukturiert sein (JSON-Format). Freitextlogs sind verboten.

| Feld | Pflicht | Beschreibung |
|------|---------|--------------|
| `timestamp` | Ja | MarketClock-Zeitstempel (nicht Instant.now()) |
| `level` | Ja | INFO, WARN, ERROR |
| `runId` | Ja (wenn Pipeline-Kontext) | Universeller Join-Key |
| `cycleNumber` | Ja (wenn Cycle-Kontext) | Cycle innerhalb des Runs |
| `decisionId` | Wenn Decision-Kontext | Verknuepfung zur Entscheidung |
| `component` | Ja | Quellkomponente (DataPipeline, KpiEngine, Arbiter, OMS, etc.) |
| `eventType` | Ja | Maschinenlesbarer Event-Typ |
| `payload` | Optional | Strukturierte Zusatzdaten |

### 4.2 Log-Level-Policy

| Level | Verwendung |
|-------|-----------|
| **ERROR** | Systemfehler, Broker-Rejects, unerwartete Zustaende |
| **WARN** | DQ-Gate-Verletzungen, LLM-Retries, Degradation-Events, hohe Slippage |
| **INFO** | Zustandswechsel, Entscheidungen, Fills, Start/Stop |
| **DEBUG** | Indikator-Berechnungen, Repricing-Details (nur in Dev/Paper) |

### 4.3 Secrets-Schutz

- Logs DUERFEN KEINE Secrets enthalten (API-Keys, Broker-Credentials, Account-Nummern)
- Ein **Log-Scrubber** MUSS aktiv sein, der bekannte Muster filtert
- Logs enthalten Trade-Daten (Preis, Stueckzahl), aber keine Kontoinformationen

---

## 5. Audit-Trail (Normativ)

### 5.1 Grundprinzip

Der Audit-Trail ist die unveraenderliche Aufzeichnung aller systemrelevanten Ereignisse. Er dient der Reproduzierbarkeit, Forensik und Nachweisbarkeit.

### 5.2 Append-Only mit Hash-Chain

- **Append-Only:** EventRecords koennen nur geschrieben, nicht geaendert oder geloescht werden
- **Hash-Chain:** Jeder EventRecord enthaelt den SHA-256-Hash des vorherigen Eintrags (Blockchain-Prinzip). Manipulation eines einzelnen Eintrags bricht die Kette und wird bei Verifikation sofort erkannt
- **Externe Kopie:** Taeglich wird eine signierte Kopie auf separaten Storage geschrieben
- **Retention:** Konfigurierbar, Default 5 Jahre (`odin.audit.log-retention-years`)

### 5.3 Event-Taxonomie

Jeder 3m-Decision-Cycle loggt folgende Events:

| Event-Kategorie | Inhalt |
|-----------------|--------|
| **Snapshot-Hash** | Hash des MarketSnapshot + FeatureSnapshot fuer Reproduzierbarkeit |
| **QuantVote** | Vollstaendiger Quant-Output (Score, Intent, Vetos, ReasonCodes) |
| **LlmVote** | Vollstaendiger LLM-Output (oder "missing" + Grund) inkl. Prompt-Version |
| **Fusion-Entscheidung** | Arbiter-Intent + Arbiter-Parameter + ReasonCode |
| **Risk-Gate-Result** | Pass/Fail + Fail-Reasons (SESSION_*, DQ_*, RISK_*, etc.) |
| **OMS-Aktionen** | Order-Intents (New, Modify, Cancel), Fill-Events |
| **FSM-Transition** | state_before -> state_after |
| **CycleNumber** | Cycle-Counter fuer Multi-Cycle-Day |

Zusaetzlich werden folgende Events ausserhalb des 3m-Cycles geloggt:

| Event-Typ | Trigger |
|-----------|---------|
| **MonitorEvent** | 1m-Events: PARABOLIC_ACCEL, EXHAUSTION_SIGNAL, SHOCK_DOWN, etc. |
| **DQ-Event** | Data-Quality-Verletzungen: STALE_FEED, OUTLIER_REJECTED, CRASH_DETECTED |
| **Broker-Event** | Fill, Reject, Partial Fill, Cancel-Confirmation |
| **Stop-Update** | Jede Stop-Nachfuehrung mit vorherigem und neuem Level |
| **State-Transition** | Jede FSM-Transition |
| **Degradation-Event** | QUANT_ONLY, DATA_HALT, Repricing-Degradation |
| **Kill-Switch-Event** | Trigger, Aktion, Ergebnis |
| **System-Health** | Heartbeat, Reconnect, Startup, Shutdown |

### 5.4 Trade-Intent-Signierung

Jeder Trade-Intent (Output der Decision Loop) wird mit einem **HMAC** signiert:

- **Payload:** Instrument, Richtung, Stueckzahl, Limit-Preis, Timestamp
- **Zweck:** Nachweisbarkeit, dass kein Trade-Intent nachtraeglich manipuliert wurde
- **Pruefung:** Die Execution-Schicht validiert die Signatur vor Order-Submission

### 5.5 Reproduzierbarkeit

- **Prompt-Versionierung:** Jede Prompt-Version ist mit Semantic Versioning versioniert (semver)
- **LLM Response Caching:** Alle LLM-Responses werden mit Input-Hash + Output + Model-ID + Timestamp persistiert
- **Replay Guarantee:** Gleiche Inputs + gleiche Konfiguration + gecachte LLM-Responses = gleiche Entscheidungen
- **Snapshot-Hashes:** Jede Entscheidung enthaelt den vollstaendigen Snapshot-Hash, um spaeter "genau dieselbe Situation" rekonstruieren zu koennen
- **Konfigurations-Snapshot:** Jeder Run speichert den vollstaendigen Config-Snapshot (Hash + Inhalt) inkl. prompt_version, schema_version, strategy_version, cost_model_version

---

## 6. Snapshot- und DecisionLog-Schemata (Normativ)

### 6.1 MarketSnapshot

Ein MarketSnapshot ist der minimale Input fuer Brain/Risk und MUSS versioniert werden.

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `runId` | UUID | Run-Referenz |
| `instrumentId` | String | Instrument-Identifier |
| `timestamp` | Instant | MarketClock-Zeitstempel |
| `sessionPhase` | Enum | PRE_MARKET, OPENING_BLOCK, RTH, FORCED_CLOSE, EOD |
| `bar1m` | OHLCV | Aktueller 1m-Bar |
| `bar3m_lastClosed` | OHLCV | Letzter abgeschlossener 3m-Bar |
| `bar10m_lastClosed` | OHLCV | Letzter abgeschlossener 10m-Bar |
| `vwap_day` | BigDecimal | Tages-VWAP (Source-of-Truth: odin-data) |
| `levels` | Object | prior_close, prior_high, prior_low, day_open, day_high, day_low |
| `dq_state` | Object | staleFeedFlag, lastDqEvent |
| `monitor_events_active` | List | Liste aktiver MonitorEvents |

### 6.2 FeatureSnapshot (3m)

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `ema9` | double | EMA(9) auf 3m-Close |
| `ema21` | double | EMA(21) auf 3m-Close |
| `rsi14` | double | RSI(14) auf 3m-Close |
| `atr14` | double | ATR(14) auf 3m |
| `adx14` | double | ADX(14) auf 3m |
| `volume_ratio` | double | Volume / SMA(Volume, 20) |
| `range_atr_ratio` | double | (High-Low) / ATR(14) |
| `dist_to_vwap_atr` | double | (Close - VWAP) / ATR(14) |
| `pattern_flags` | Set | Aktive Pattern-Flags (COIL, RECLAIM, etc.) |
| `feature_version` | String | Version des Feature-Sets |

### 6.3 DecisionLog

Jede 3m-Entscheidung erzeugt genau einen DecisionLog.

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `decisionId` | UUID | Eindeutige Decision-ID |
| `runId` | UUID | Run-Referenz |
| `cycleNumber` | int | Cycle-Counter |
| `timestamp` | Instant | MarketClock-Zeitstempel |
| `state_before` | Enum | FSM-State vor Entscheidung |
| `state_after` | Enum | FSM-State nach Entscheidung |
| `market_snapshot_hash` | String | SHA-256 des MarketSnapshot |
| `feature_snapshot_hash` | String | SHA-256 des FeatureSnapshot |
| `quant_vote` | JSON | Vollstaendiger QuantVote |
| `llm_vote` | JSON | Vollstaendiger LlmVote (oder "missing" + reason) |
| `arbiter_intent` | Enum | ENTER, ADD, SCALE_OUT, EXIT, NO_ACTION |
| `arbiter_params` | JSON | Fusions-Parameter (stop_level, target_levels, trail_mode, etc.) |
| `risk_gate_result` | JSON | Pass/Fail + ReasonCodes |
| `execution_commands` | JSON | Order-Intents |
| `outcome_link` | JSON | Wird spaeter gefuellt: Fill/Exit-Ergebnisse |

### 6.4 LLMCall Log

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `llmCallId` | UUID | Eindeutige Call-ID |
| `decisionId` | UUID | Optional: Verknuepfung zur Entscheidung |
| `timestamp` | Instant | Zeitstempel |
| `prompt_version` | String | Semver der Prompt-Version |
| `schema_version` | String | Semver des Output-Schemas |
| `input_hash` | String | SHA-256 des Inputs |
| `token_count_estimate` | int | Geschaetzte Token-Anzahl |
| `latency_ms` | long | Antwortzeit in Millisekunden |
| `timeout_flag` | boolean | Timeout aufgetreten |
| `output_json` | JSON | Roher LLM-Output |
| `validation_result` | Enum | VALID, SCHEMA_INVALID, CONTENT_INVALID, TIMEOUT |
| `ttl_seconds` | int | Finaler TTL-Wert |

---

## 7. Report-Artefakte

### 7.1 Trade-Journal (pro Trade, Normativ)

Jeder abgeschlossene Trade MUSS folgende Informationen enthalten:

| Kategorie | Felder |
|-----------|--------|
| **Trade-Daten** | Instrument, Entry-Zeit/Preis, Exit-Zeit/Preis, Stueckzahl, Richtung, P&L (absolut + %), MAE, MFE |
| **Setup** | Setup-Typ (A/B/C/D), Pattern-Flags bei Entry |
| **LLM-Analyse** | Regime, regime_confidence, intent_bias, urgency, red_flags, exit_bias, trail_mode, market_context_signals |
| **Quant-Validierung** | quant_score (gesamt + je Check), Vetos, ReasonCodes |
| **Fusion** | Arbiter-Intent, Arbiter-ReasonCode, Dual-Key-Ergebnis |
| **Risk-Gate** | Position-Size, Stop-Level, R/R-Ratio, Budget-Verbrauch |
| **Execution** | Order-Typ, Fill-Preis, Slippage, Cancel/Replace-Zyklen, Dauer bis Fill |
| **Kontext** | Tageszeit, Regime bei Entry, bisherige Tages-P&L, Trade-Nr, cycleNumber |
| **Profit-Protection** | R-Level bei Exit, Trail-Mode, MFE-Lock aktiv ja/nein |

### 7.2 Tages-Report (EOD, Normativ)

Am Ende jedes Handelstages MUSS ein Tages-Report erzeugt werden:

| Kategorie | Inhalt |
|-----------|--------|
| **P&L** | Gesamt-P&L, P&L pro Pipeline, P&L pro Cycle |
| **Trade-Statistik** | Anzahl Trades, Win-Rate, avg. Gewinn/Verlust, groesster Gewinn/Verlust |
| **Drawdown** | Max. Drawdown (intraday), Max. Single-Trade-Verlust |
| **LLM-Analyse** | LLM-Accuracy (Regime-Diagnose korrekt?), Anteil LLM-Vetos, Agreement-Rate Quant/LLM |
| **Quant-Analyse** | Quant-Veto-Rate und Veto-Qualitaet, Score-Verteilung |
| **Rules-Engine** | Welche Regeln haben am haeufigsten Entry/Exit ausgeloest? (Rule-ID + Count) |
| **Regime-Calibration** | Ground-Truth vs. Predicted (Vergleich Quant/LLM-Regime mit 30-Min-Lookforward) |
| **DQ/Incidents** | DQ-Violations, Degradation-Events, Broker-Rejects |
| **Parameter-Snapshot** | Alle aktiven Konfigurationsparameter + Versionen |

### 7.3 Langfrist-Metriken (ueber 100+ Handelstage)

| Metrik | Beschreibung |
|--------|--------------|
| **LLM Confidence Calibration** | Korrelation zwischen regime_confidence und tatsaechlicher Performance |
| **Regime-Detection-Accuracy** | Vergleich Quant/LLM-Regime mit Ground-Truth (Abschnitt 09) |
| **Quant-Override-Analyse** | Waere der Trade profitabel gewesen, wenn LLM-Veto ignoriert worden waere? |
| **Adaptions-Tracking** | Optimale Schwellenwerte ueber Zeit (Drift-Erkennung) |
| **Sharpe Ratio** | Gegenueber Buy-and-Hold-Benchmark |
| **Geometric Score** | `S = mean(ln(1 + r_d)) - lambda * ES_95 - mu * MDD` (fortlaufend) |
| **Capture Ratio** | Anteil der Tagesrange, die eingefangen wurde |
| **Overtrade Score** | Unnoetige Trades in Chop (Range-Days) |
| **Safety Score** | Verhalten in Crash/Halt-Szenarien |

---

## 8. Alert-Routing und Eskalation (Normativ)

### 8.1 Alert-Levels

| Alert-Level | Beispiele | Routing | Reaktionszeit |
|-------------|----------|---------|---------------|
| **INFO** | Tages-Report, Trade-Summary, Cycle-Complete | Dashboard + Email (EOD) | Kein SLO |
| **WARNING** | Hohe Slippage, LLM-Retry, DQ-Gate-Verletzung, Repricing-Degradation, abnormale Fill-Rate | Push-Notification | < 30 Minuten |
| **CRITICAL** | Kill-Switch ausgeloest, LLM-Ausfall (3 Timeouts), Daten-Feed-Ausfall (> 60s), Halluzinations-Kaskade (3+ verworfen), Broker-Reject bei ungeschuetzter Position | Push + SMS/Call | < 5 Minuten |
| **EMERGENCY** | "Cannot Liquidate", Account-Restriction, System-Crash, Heartbeat-Timeout (90s), Flash-Crash-Detection | SMS + Call + Auto-Escalation | < 2 Minuten |

### 8.2 Eskalationspfade

Die vier Eskalationsstufen MUESSEN implementiert sein:

| Stufe | Beschreibung | Beispiel |
|-------|-------------|---------|
| **1. Automatische Reaktion** | System reagiert selbst ohne menschlichen Eingriff | Kill-Switch, QUANT_ONLY-Degradation, DATA_HALT, Repricing-Degradation |
| **2. Operator-Alert** | Mensch wird informiert, System laeuft weiter | WARNING-Level: hohe Slippage, LLM-Retry, DQ-Warnung |
| **3. Operator-Eingriff** | System pausiert, wartet auf manuellen Eingriff | Safe-Mode nach Crash, LULD ueber Close |
| **4. Notfall-Shutdown** | System wird komplett gestoppt, alle Positionen geschlossen | EMERGENCY-Level: Cannot Liquidate, System-Crash |

### 8.3 Runbooks

Fuer jeden Alert-Typ SOLL ein dokumentiertes Runbook mit Schritt-fuer-Schritt-Anleitung zur Diagnose und Behebung existieren.

**Beispiel-Runbooks:**

| Alert | Runbook-Schritte |
|-------|-----------------|
| **DQ_STALE_FEED WARNING** | 1. Pruefen ob Markt offen, 2. Datafeed-Verbindung pruefen, 3. Wenn persistent > 60s: automatischer Kill-Switch |
| **LLM Outage** | 1. QUANT_ONLY automatisch, 2. Keine neuen Entries, 3. Bestehende Position managen, 4. Erster erfolgreicher Call hebt QUANT_ONLY auf |
| **Broker Reject** | 1. CRITICAL Alert, 2. Automatische Retries (max. 3), 3. Wenn Position ungeschuetzt: risk-reducing Exit, 4. Kill-Switch wenn persistent |
| **Cannot Liquidate** | 1. EMERGENCY, 2. Kill-Switch, 3. Retry alle 30s, 4. Manueller Eingriff dokumentieren |

---

## 9. SLOs (Service Level Objectives, Normativ)

| Metrik | SLO | Messung |
|--------|-----|---------|
| System-Uptime (waehrend RTH) | 99.5% pro Monat | Heartbeat-Monitoring |
| LLM-Response-Latenz (p95) | < 5 Sekunden | Per-Call-Messung |
| Order-Latenz (Intent -> Broker) | < 500ms | Per-Order-Messung |
| Kill-Switch-Latenz (Trigger -> Order-Submission) | < 2 Sekunden | Regelmaessige Tests |
| DQ-Gate-False-Positive-Rate | < 1% | Taeglich auswerten |

---

## 10. Market Surveillance (Normativ)

Ueberwachung des eigenen Orderverhaltens zur Selbstkontrolle und operationellen Sicherheit:

| Metrik | Schwelle | Aktion |
|--------|----------|--------|
| **Cancel/Replace-Rate** | > 5:1 (ueber 30 Min) | WARNING + Repricing-Degradation (Intervall 5s -> 10s) |
| **Order/Trade-Ratio** | > 10:1 (ueber 60 Min) | WARNING + Repricing auf 1 Zyklus, kein Abandon-Retry |
| **Clustering-Detection** | Repetitive Orders in gleichen Preisbaendern | WARNING + Untersuchung (koennte als Marktmanipulation interpretiert werden) |
| **Fill-Rate** | < 30% (ueber 5 Versuche) | Positionsgroesse halbieren ODER OBSERVING |

### Market-Surveillance-Degradation

Die Degradation-Stufen MUESSEN im Audit-Log protokolliert werden. Parameter normalisieren sich automatisch, wenn die Ratio-Metriken wieder unter den Schwellenwerten liegen:

| Bedingung | Degradation | Recovery |
|-----------|-------------|----------|
| Cancel/Replace-Rate > 5:1 (30 Min) | Repricing-Intervall 5s -> 10s | Rate < 3:1 |
| Order/Trade-Ratio > 10:1 (60 Min) | Repricing 1 Zyklus, kein Abandon-Retry | Ratio < 6:1 |
| Fill-Rate < 30% (5 Versuche) | Positionsgroesse halbieren oder OBSERVING | Naechster LLM-Zyklus |

---

## 11. LLM-Monitoring in Produktion (Normativ)

Folgende LLM-spezifische Metriken MUESSEN ueberwacht werden:

| Metrik | Beschreibung | Alert-Schwelle |
|--------|--------------|----------------|
| **Timeout Rate** | Anteil LLM-Calls die timeout erreichen | > 2% intraday -> Degradation + Incident |
| **Schema Invalid Rate** | Anteil Responses mit Schema-Verletzung | > 0% -> Release blockieren |
| **Response Latency p95** | 95. Perzentil der Antwortzeit | > 5s -> WARNING |
| **LLM-Veto-Anteil** | Anteil der Entscheidungen mit LLM-Veto | Monitoring (kein fester Alert, aber Drift-Indikator) |
| **Performance-Delta** | Hybrid vs. Quant-Only (Shadow Mode moeglich) | Negative Delta ueber 20 Tage -> Untersuchung |
| **Agreement-Rate** | Uebereinstimmung Quant/LLM Regime-Label | Sinkende Agreement-Rate -> Drift-Warnung |
| **Confidence-Verteilung** | Verteilung der regime_confidence Werte | Shift der Verteilung -> Drift-Indikator |

---

## 12. Dashboard-Komponenten

### 12.1 Chart

- **TradingView Lightweight Charts** (Candlestick, Volumen, Indikatoren)
- **Overlay:** Entry/Exit-Marker, Stop-Level, Target-Level, VWAP-Linie, EMA-Linien
- **Farbschema:** Bullish `#0EB35B`, Bearish `#FC3243`, VWAP `#2196F3`, EMA50 `#214226`, EMA100 `#26211B`, RSI `#A78BFA`
- **Session-Baender:** Pre-Market braun, RTH normal, After-Hours blau (dezent)

### 12.2 Pipeline-Status-Panel

- State (FSM-Zustand mit Farbcodierung)
- Position (Stueckzahl, Entry-Preis, unrealisiertes P&L)
- Orders (offene Orders mit Typ und Status)
- Letzte Entscheidung (Arbiter-Intent + ReasonCode)

### 12.3 Global Dashboard

- AccountRiskState (Budget, Exposure, Drawdown)
- Aggregierte P&L (alle Pipelines)
- Kill-Switch-State (mit Farbindikator)
- Trade-Counter (Round-Trips heute)

### 12.4 Alert-Panel

- Eskalationsstufen (INFO -> WARNING -> CRITICAL -> EMERGENCY)
- Chronologische Alert-Liste mit Timestamp und Quelle
- Aktive CRITICAL/EMERGENCY Alerts hervorgehoben

### 12.5 Controls

- Kill-Switch-Button (global, mit Bestaetigung)
- Pause/Resume pro Pipeline
- Safe-Mode-Toggle

---

## 13. Operational Runbook

### 13.1 Start-of-Day Checklist (Normativ)

Vor Handelsbeginn MUSS folgende Checkliste abgearbeitet werden:

1. **System Health:** Datafeed erreichbar (Heartbeat ok), Broker erreichbar (Auth ok), LLM erreichbar (optional, aber Status bekannt)
2. **Config Freeze:** Watchlist/Instrumente festlegen, configSnapshot erzeugen (Hash)
3. **Mode-Pruefung:** Simulation/Paper/Live Mode pruefen -- falscher Mode = Start verweigern
4. **Kill-Switch Test:** Dry-Run -- Control-Channel erreichbar
5. **Flat-Check:** Keine offenen Positionen aus dem Vortag (Reconciliation)

### 13.2 Intraday Incident Handling (Normativ)

| Incident | Reaktion |
|----------|----------|
| **DQ_STALE_FEED WARNING** | Beobachten, keine neuen Entries |
| **DQ_STALE_FEED CRITICAL** | DATA_HALT, ggf. risk-reducing Exit |
| **LLM Outage** | QUANT_ONLY: keine neuen Entries, bestehende Position managen |
| **Broker Reject** | CRITICAL Alert, automatische Retries begrenzt, wenn Position ungeschuetzt: risk-reducing Exit (Fallback) |
| **Flash Crash** | Stop/Exit ohne LLM, ggf. DAY_STOPPED |
| **Cannot Liquidate** | EMERGENCY: Kill-Switch, Retry alle 30s, manuelle Eskalation |

### 13.3 End-of-Day Checklist (Normativ)

Nach Handelsschluss MUSS folgende Checkliste abgearbeitet werden:

1. **EOD-Flat verifizieren:** Position = 0 fuer alle Pipelines
2. **Reconciliation:** Orders/Fills konsistent? Interne Position = Broker-Position?
3. **Reports:** RunSummaryEod erzeugen (Tages-Report)
4. **Audit:** EventLog Hash-Chain verifizieren
5. **Archiv:** Taeglich signierte Kopie des EventLogs auf separaten Storage

---

## 14. ReasonCode-Taxonomie (Normativ)

ReasonCodes MUESSEN maschinenlesbar und hierarchisch sein. Sie ermoeglichen spaetere Analyse (Heatmaps, Fehlkalibrierung, LLM-Veto-Qualitaet).

| Praefix | Quelle | Beispiele |
|---------|--------|-----------|
| `SESSION_*` | Session-Gate | `SESSION_OPENING_BLOCK`, `SESSION_ENTRY_CUTOFF` |
| `DQ_*` | Data-Quality-Gate | `DQ_STALE_FEED`, `DQ_OUTLIER_REJECTED`, `DQ_CRASH_DETECTED` |
| `RISK_*` | Risk-Gate | `RISK_BUDGET_EXHAUSTED`, `RISK_EXPOSURE_LIMIT`, `RISK_RR_TOO_LOW` |
| `LLM_*` | LLM-Gate | `LLM_VETO_ENTRY`, `LLM_TIMEOUT`, `LLM_SCHEMA_INVALID` |
| `EXEC_*` | Execution-Gate | `EXEC_SPREAD_TOO_WIDE`, `EXEC_VOLUME_TOO_LOW` |
| `A_*` | Setup A | `A_NO_VOL_DECAY`, `A_NO_RANGE_COMPRESSION`, `A_BELOW_VWAP`, `A_EMA_NOT_ALIGNED`, `A_OVEREXTENDED`, `A_RSI_OVERHEAT`, `A_SCORE_TOO_LOW` |
| `B_*` | Setup B | `B_NO_FLUSH_EVENT`, `B_NEW_LOW_CONTINUES`, `B_NO_RECLAIM_VWAP`, `B_NO_RECLAIM_MA`, `B_CONFIRMATION_MISSING` |
| `C_*` | Setup C | `C_NO_COIL`, `C_BREAKOUT_WEAK`, `C_VOLUME_NOT_CONFIRMED`, `C_NO_RETEST_ABORT` |
| `D_*` | Setup D | `D_NOT_ELIGIBLE`, `D_COOLDOWN_ACTIVE`, `D_NO_RECLAIM`, `D_NO_HIGHER_LOW`, `D_RISK_POLICY_BLOCK` |

---

## 15. Datenspeicherung und Retention

### 15.1 Speicherobjekte

| Objekt | Aufbewahrung | Beschreibung |
|--------|-------------|--------------|
| **Raw Bars (1m)** | Min. 6--12 Monate (lizenzabhaengig) | Fuer Replay und Forensik |
| **Resampled Bars (3m/10m)** | Optional (ableitbar aus 1m) | Komfort, nicht zwingend |
| **EventLog** | 5 Jahre (Default, konfigurierbar) | Append-Only, hash-verkettet |
| **ConfigSnapshots** | 5 Jahre | Versionen, Parameter |
| **Reports (EOD)** | 5 Jahre | Tages-Reports |
| **LLM-Cache** | Mindestens solange Backtests reproduziert werden muessen | Input + Output + Model-ID |

### 15.2 Zwei Persistierungspfade

1. **Direkte Domaenen-Persistierung:** Domaenen-Module schreiben strukturierte Daten synchron ueber eigene Repositories (DDD)
2. **AuditEventDrainer:** Konsumiert EventLog-Spool, schreibt NUR EventRecords (flaches Archiv)

---

## 16. Performance-Anforderungen

### 16.1 Latenzbudgets

Da 1m-Bars die kleinste Zeiteinheit sind, ist das System nicht ultra-low-latency:

| Pfad | Ziel |
|------|------|
| Decision Cycle (3m close -> Intent) | < 500ms typisch |
| OMS-Updates (Stop-Tighten auf 1m close) | < 200ms typisch |
| LLM-Calls | TTL respektieren, p95 < 5s |
| Kill-Switch (Trigger -> Order-Submission) | < 2s (SLO) |

### 16.2 Fail-Fast-Prinzip (Normativ)

Bei widerspruechlichen Daten oder Zustaenden:

- Lieber **NO_ACTION** als falscher Trade
- Lieber **HALT** als unkontrolliertes Weiterhandeln

---

## 17. Teststrategie fuer Observability-Komponenten

### 17.1 Unit Tests

- **Indikatoren:** EMA/RSI/ATR/ADX auf bekannten Testreihen
- **Resampling:** 1m -> 3m/10m Aggregation korrekt (Open/High/Low/Close/Volume)
- **DQ Gates:** Reihenfolge korrekt (Crash-Markierung vor Outlier-Reject)
- **Arbiter Tables:** Jede Kombination QuantVote/LLMBias -> erwarteter Intent

### 17.2 Integration Tests

- **Replay:** Eines Tages mit deterministischem Ergebnis (gleiche Events, gleiche Trades)
- **OMS:** Partial Fills + Stop/Target Updates
- **Degradation:** LLM Timeout -> Quant-Only + No-New-Entry
- **Kill-Switch:** End-to-End (Orders Cancel + Position Close)

### 17.3 Property Tests (Invarianten)

Folgende Invarianten MUESSEN jederzeit gelten und SOLLEN durch Property-Tests verifiziert werden:

- Position > 0 => es existiert ein Schutzmechanismus (Broker-Stop oder interner Stop)
- Stops werden nie gelockert (Highwater-Mark-Invariante)
- EOD => Position == 0
- EventLog Hash-Chain ist valid (kein Eintrag manipuliert)

---

## 18. Parameterkatalog (Observability-relevant, konfigurierbar)

| Parameter | Default | Beschreibung |
|-----------|---------|--------------|
| `odin.audit.log-retention-years` | 5 | Aufbewahrungsdauer EventLog |
| `odin.monitoring.heartbeat-interval-ms` | 5000 | Heartbeat-Intervall |
| `odin.monitoring.sse-timeout-ms` | -1 (unbegrenzt) | SSE-Emitter Timeout |
| `odin.monitoring.dq-stale-seconds-warn` | 90 | Warnschwelle keine neuen Bars |
| `odin.monitoring.dq-stale-seconds-halt` | 180 | Halt-Schwelle keine neuen Bars |
| `odin.monitoring.cancel-replace-rate-warn` | 5.0 | Cancel/Replace-Rate Warnschwelle |
| `odin.monitoring.order-trade-ratio-warn` | 10.0 | Order/Trade-Ratio Warnschwelle |
| `odin.alert.warning-response-minutes` | 30 | SLO: Reaktionszeit WARNING |
| `odin.alert.critical-response-minutes` | 5 | SLO: Reaktionszeit CRITICAL |
| `odin.alert.emergency-response-minutes` | 2 | SLO: Reaktionszeit EMERGENCY |
