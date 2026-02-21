# 06 -- Risk Management, Position Sizing, Kill-Switch, Crash-Recovery, Security

> **Quellen:** Fachkonzept v1.5 (Kap. 10: Risikomanagement & Guardrails; Kap. 19: Security & Isolation; Kap. 20: Operationelle Sicherheit), Master-Konzept v1.1 (Risk Management, Kill-Switch, Crash-Recovery), Build-Spec v3.0 (Kap. 10: Risk, Sizing, Guardrails; Kap. 15: Security), Strategie-Sparring (Position Sizing, Fixed Fractional Risk, Degradation Modes)

---

## 1. Drei Verteidigungslinien (Normativ)

Das Risikomanagement ist in drei hierarchische Verteidigungslinien gegliedert. Jede Linie operiert unabhaengig -- der Ausfall einer Linie DARF NICHT zum unkontrollierten Verlust fuehren.

---

### 1.1 Linie 1: Per-Trade (Trade-Ebene)

| Regel | Wert | Beschreibung |
|-------|------|-------------|
| Stop-Loss-Distance | **2.2 x ATR(14)** auf 3m-Bars | ATR wird bei Entry eingefroren (`entryAtr`). Faktor 2.2 ist Stakeholder-Entscheidung |
| Max. Positionsgroesse | 50% des Tageskapitals | Kein einzelner Trade darf mehr als die Haelfte des Kapitals binden |
| Min. Risk/Reward-Ratio | 1 : 1.5 | Entry wird vom Risk-Gate blockiert, wenn R/R unter 1.5 liegt |
| Max. Slippage-Toleranz | 0.3% vom Entry-Preis | Bei Ueberschreitung: Stop anpassen, ggf. sofortiger Exit wenn R/R zerstoert |
| Stop-Distance-Bounds | Zu eng oder zu weit → blockieren | Verhindert unrealistische Stops (z.B. < 0.5 x ATR oder > 5 x ATR) |

---

### 1.2 Linie 2: Tages-Ebene (global ueber alle Pipelines UND alle Zyklen)

| Regel | Wert | Scope |
|-------|------|-------|
| **Hard-Stop** | -10% Tageskapital (realisiert + unrealisiert). Echtzeit-Pruefung | **Global** ueber alle Pipelines |
| Max. Trades (Cycles)/Tag | 5 | **Global** ueber alle Pipelines und Cycles |
| Max. Zyklen pro Pipeline/Tag | 3 (konfigurierbar) | Per Pipeline |
| Cooling-Off nach Verlustserie | 30 Minuten nach 3 Verlusten in Folge | Global |
| Exposure-Limit | 80% des Tageskapitals | Global (Pre-Trade-Check) |

> **Limit-Hierarchie:** Das globale 5-Cycles-Limit dominiert ueber das per-Pipeline-Cycle-Limit (`maxCyclesPerInstrumentPerDay`). Beispiel: Bei 3 Pipelines mit je 2 Cycles waeren 6 Cycles theoretisch moeglich -- das globale Limit von 5 greift vorher. In der Praxis ist das globale Limit das schaerfere Constraint.

---

### 1.3 Linie 3: System-Ebene

| Regel | Wert | Beschreibung |
|-------|------|-------------|
| Broker-seitige GTC-Stops | Immer aktiv | Redundanz bei System-Ausfall: Stops liegen beim Broker |
| Heartbeat-Monitor | 3 x 30s = 90s → Kill-Switch | Prozess-Watchdog erkennt System-Ausfall |
| Daten-Feed-Monitor | Kein Tick seit > 60s → Kill-Switch | DQ-Gate eskaliert bei persistierendem Datenausfall |
| LLM-Availability | 3 Failures in Folge → Quant-Only | Degradation statt sofortiger Halt |
| OMS/Broker-Heartbeat | Fehlende Bestaetigungen → ALERT | Broker-Verbindungsproblem erkennen |

---

## 2. Position Sizing: Fixed Fractional Risk (Normativ)

Die Positionsgroesse wird ueber ein Fixed-Fractional-Risk-Modell berechnet. Ziel: Kein einzelner Trade DARF den Hard-Stop durchbrechen koennen.

### 2.1 Berechnungsschritte

| Schritt | Formel | Beschreibung |
|---------|--------|-------------|
| **1. Risikobetrag** | `max_risk = min(Tageskapital x 3%, remaining_budget x 0.8)` | 3% pro Trade, Safety Margin 80% auf verbleibendes Budget |
| **1b. FX-Conversion** | `max_risk_usd = max_risk_eur x EUR/USD_spot` | Umrechnung fuer US-Aktien |
| **2. Stop-Distance** | `stop_dist = 2.2 x entryAtr` | ATR eingefroren bei Entry |
| **2b. Stress-Adjusted** | `effective_stop_dist = stop_dist x stress_factor` | `stress_factor = 1.0` normal, `1.3` bei HIGH_VOLATILITY |
| **3. Stueckzahl** | `shares = floor(max_risk_usd / effective_stop_dist)` | Ganzzahlig abgerundet |
| **4. Cap** | Max. 50% Tageskapital als Positionswert | Absolute Obergrenze |
| **5. Confidence-Skalierung** | `shares = shares x min(regime_confidence, quant_score)` | Reduzierung bei niedriger Confidence |

### 2.2 Cycle-2+ Sizing

Fuer Multi-Cycle-Day gilt: Zyklus 2+ wird mit einem konfigurierbaren Faktor (Default: 0.6 -- 0.8) multipliziert:

```
shares_cycle2 = shares_standard x cycle2_size_factor
```

Das verbleidende Risk-Budget nach realisiertem P&L bestimmt die Obergrenze.

---

## 3. Dynamisches Tages-Budget (Normativ)

```
remaining_budget = (Tageskapital x 10%) - abs(realisierte_verluste) - abs(unrealisierte_verluste)
```

**Kernregel:** Gewinne erhoehen das Budget NICHT. Die Safety Margin (0.8) stellt sicher, dass ein einzelner Trade den Hard-Stop nicht durchbrechen kann.

**Multi-Cycle Risk-Budget:**

| Aspekt | Single-Trade-Day | Multi-Cycle-Day |
|--------|-------------------|-----------------|
| Positionsgroesse | Standard-Sizing | Zyklus 2+: Reduziert (Default 60%) |
| Risk-Budget pro Zyklus | Gesamtes Tagesbudget | Restbudget nach realisiertem P&L |
| Max. Zyklen | 1 | 3 (konfigurierbar, Hard-Cap) |

---

## 4. Kill-Switch (Normativ)

Der Kill-Switch ist das ultimative Sicherheitsinstrument und MUSS mehrfach implementiert sein:

### 4.1 Implementierungs-Schichten

| Schicht | Implementierung | Beschreibung |
|---------|----------------|-------------|
| **1. Software** | Agent → DAY_STOPPED | Alle Positionen per Market-Order schliessen, alle Orders stornieren |
| **2. Broker-seitig** | IB Account-Limits | Fallback bei Software-Ausfall. GTC-Stops schliessen Positionen |
| **3. Hardware** | Externer Watchdog-Prozess | Beendet den Agenten bei Anomalien (Heartbeat-Timeout) |
| **4. Manuell** | Tastendruck, REST-Endpoint, CLI | Jederzeit per Operator ausloesbar |

### 4.2 Ausloeser (automatisch)

| Trigger | Schwelle |
|---------|----------|
| Tages-Drawdown | >= 10% (realisiert + unrealisiert) |
| Data-Feed-Ausfall | > 60s keine validen Ticks waehrend RTH |
| Heartbeat-Timeout | 3 x 30s = 90s ohne Heartbeat |
| Flash Crash | > 5% Preisbewegung in < 1 Minute + Volume > 3x |
| System-Fehler im OMS | Broker rejects / cannot liquidate |
| Mehrfaches Crash-Event | Kombiniert mit grosser Slippage |

### 4.3 Aktion

1. Alle offenen Orders stornieren
2. Alle Positionen per **Market-Order** schliessen (einer der erlaubten Market-Order-Faelle)
3. Pipeline-State auf **DAY_STOPPED** setzen
4. **EMERGENCY Alert** emittieren + Audit Event loggen
5. Kein weiterer Handel fuer den Rest des Tages

### 4.4 Kill-Switch-Ownership

**odin-data eskaliert, odin-core entscheidet.** Das Data-Modul erkennt DQ-Verletzungen und emittiert Events. Das Core-Modul (GlobalRiskManager) evaluiert diese Events im Kontext aller Pipelines und trifft die Kill-Switch-Entscheidung.

### 4.5 Kill-Switch-Latenz (SLO)

Der Kill-Switch MUSS innerhalb von **< 2 Sekunden** vom Trigger bis zur Order-Submission beim Broker ausgefuehrt werden. Regelmaessige Tests SOLLEN die Latenz verifizieren.

---

## 5. Crash-Recovery (Normativ)

ODIN ist in v1 NICHT intraday-restartable fuer Trading. Bei einem System-Crash:

### 5.1 Sofort-Schutz

- **GTC-Stops beim Broker** schuetzen offene Positionen automatisch
- Broker-seitige Stops sind unabhaengig vom ODIN-Prozess und ueberlebt jeden Crash

### 5.2 Automatischer Neustart

- **WinSW** (Windows Service Wrapper) startet den Prozess automatisch neu
- Startup-Recovery erkennt den Crash-Zustand
- System startet in **Safe-Mode** (kein Trading, nur Dashboard-Anzeige)

### 5.3 Naechster regulaerer Tagesstart

- **Flat-Check Reconciliation:** Positionen werden aus der Broker-API gelesen und mit dem internen State abgeglichen
- Offene Positionen aus dem Vortag werden identifiziert und geschlossen
- Erst nach erfolgreicher Reconciliation startet normaler Handel

---

## 6. Degradation Modes (Normativ)

Bei teilweisem Systemausfall MUSS das System kontrolliert degradieren statt sofort zu stoppen:

| Situation | Mode | Verhalten |
|-----------|------|-----------|
| LLM Timeout / invalid Schema | **QUANT_ONLY** | Keine neuen Entries. Bestehende Position managen (Trailing, Targets). EOD close |
| Data-Feed stale / stottert | **DATA_HALT** | Keine neuen Orders. Nur risk-reducing Exits erlaubt. Bei Persistenz: Kill-Switch |
| Broker rejects / cannot liquidate | **EMERGENCY** | Kill-Switch, manuelle Eskalation, Retry alle 30s |

**Degradation vor Kill-Switch:** Bei einzelnen Quality-Gate-Verletzungen wird das System zunaechst degradiert (groessere Intervalle, konservativere Regeln), bevor der Kill-Switch gezogen wird. Nur bei persistierendem Datenausfall (> 60s keine validen Ticks) wird sofort geschlossen.

---

## 7. Security und Secrets (Normativ)

### 7.1 Secrets-Handling

| Bereich | Regelung |
|---------|---------|
| **Broker-Credentials** | Ausschliesslich via Environment-Variablen oder Secrets-Manager. Nie in Code, Config-Files oder Logs |
| **LLM API-Keys** | Rotation alle 30 Tage. Separater Key pro Environment (Dev/Paper/Live) |
| **LLM-Kontext** | Keine Account-Groesse, persoenliche Daten oder historische P&L im Prompt |
| **Logging** | Logs enthalten Trade-Daten (Preis, Stueckzahl), aber keine Kontoinformationen. Log-Scrubber aktiv |

### 7.2 Netzwerk-Isolation

- **Outbound-Whitelist:** Nur IB TWS Gateway (lokal oder feste IP) + LLM API (fester FQDN)
- **Kein allgemeiner Internet-Zugang** im Trading-Prozess
- News-Daten werden ueber separaten, isolierten Prozess vorverarbeitet und nur als strukturierte Metadaten eingespeist

### 7.3 Audit-Logs (Tamper-Proof)

- **Append-Only:** Logs koennen nur geschrieben, nicht geaendert oder geloescht werden
- **Hash-Chain:** Jeder Log-Eintrag enthaelt den Hash des vorherigen Eintrags (Blockchain-Prinzip)
- **Externe Kopie:** Taeglich wird eine signierte Kopie auf separaten Storage geschrieben
- **Retention:** Konfigurierbar, Default 5 Jahre

### 7.4 Trade-Intent-Signierung

Jeder Trade-Intent (Output der Decision Loop) wird mit einem **HMAC** signiert:

- **Payload:** Instrument, Richtung, Stueckzahl, Limit-Preis, Timestamp
- **Zweck:** Nachweisbarkeit, dass kein Trade-Intent nachtraeglich manipuliert wurde
- **Pruefung:** Die Execution-Schicht validiert die Signatur vor Order-Submission

---

## 8. Anti-Overtrade-Mechanismen

| Mechanismus | Regel | Beschreibung |
|-------------|-------|-------------|
| Cooldown nach Stop-Out | 15 -- 30 Minuten (konfigurierbar) oder bis Regime stabil | Verhindert Impuls-Re-Entry direkt nach Verlust |
| Entry nur auf 3m-Takt | Kein Entry zwischen Decision-Bars | Reduziert Noise-getriebene Entries |
| Max Orders/Minute (soft) | Monitoring-Alert | Erkennt abnormales Order-Verhalten |
| Max Cancel/Replace-Rate | Alert bei > 5:1 (30 Min) | Repricing-Degradation (siehe OMS-Kapitel) |
| RANGE_BOUND/UNCERTAIN | Strenger Gate (no-trade Default) | In Seitwaerts-/Unsicherheitsphasen keine Entries |
| Multi-Cycle-Guardrails | Max 3 Zyklen, Profit-Gate, Budget-Gate, LLM-Pflicht | Verhindert unkontrolliertes Re-Entry |

---

## 9. Liquiditaetsgates ohne Spread (Proxy-basiert)

Da kein Bid/Ask verfuegbar ist, werden Liquiditaets-Proxies verwendet:

| Proxy | Schwelle | Aktion bei Verletzung |
|-------|----------|----------------------|
| Mindestpreis | > 5 USD/EUR | Instrument nicht handelbar |
| Mindest-Volumen (letzte 30m) | > konfigurierbare Schwelle | Entry blockiert |
| Keine 0-Volumen-Bars | Anhaltend 0-Volumen waehrend RTH | DQ-Warning, ggf. Entry-Sperre |
| Fill Difficulty Proxy | < 30% Fill-Rate ueber 5 Versuche | Positionsgroesse halbieren oder OBSERVING |

---

## 10. Pre-Trade Exposure-Limit (Normativ)

Bevor eine neue Position eroeffnet wird, MUSS geprueft werden, ob die resultierende Gesamt-Exposure innerhalb der Limits bleibt:

```
neue_exposure = bestehende_exposure + geplante_position
wenn neue_exposure > Tageskapital x 80% → Entry blockieren
```

Nur nach Teil-Exit einer bestehenden Position darf erneut versucht werden.

---

## 11. Operationelle Sicherheit

### 11.1 Kontext

Der Agent wird zunaechst ausschliesslich im privaten Kontext betrieben. Regulatorische Compliance (MiFID II, SEC) ist kein Designtreiber. Die Massnahmen dienen der **eigenen operationellen Sicherheit**.

### 11.2 Massnahmen-Uebersicht

| Bereich | Umsetzung |
|---------|-----------|
| **Pre-Trade Limits** | Hard-Stop bei -10%, Max. 5 Trades/Tag, Max. Position 50%, Min. R/R 1:1.5, Exposure-Limit 80% |
| **Validierung** | Paper Trading → Klein-Live Pipeline. Walk-Forward Validation |
| **Real-Time Monitoring** | Heartbeat-Monitor, P&L-Tracking, Anomalie-Detection |
| **Kill-Switch** | 4-Schichten-Implementierung (Software, Broker, Hardware, Manuell) |
| **Audit-Trail** | Append-Only mit Hash-Chain. Jeder Trade-Intent, jede LLM-Antwort, jeder Fill protokolliert |
| **Change Management** | Prompt-Versionierung, Evaluation-Suites, Approval-Prozess |
