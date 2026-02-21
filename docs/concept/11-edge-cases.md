# 11 -- Edge Cases, Failure Modes, Degradation

> **Quellen:** Fachkonzept v1.5 (Kap. 11: Anti-Halluzination; Kap. 16: News Security; Kap. 21: Edge Cases & Failure Modes), Master-Konzept v1.1 (Edge Cases, Degradation Modes), Build-Spec v3.0 (Kap. 16: Edge Cases; Kap. 8.8: Anti-Halluzination), Strategie-Sparring (Degradation-Kaskade, Challenger-Suite-Abdeckung, News Security), Stakeholder-Feedback (Fail-Safe-Prinzip, Kill-Switch-Ownership)

---

## 1. Grundprinzip (Normativ)

ODIN operiert in einem Umfeld, in dem Markt-Extremsituationen, Infrastruktur-Ausfaelle und unerwartete Zustaende jederzeit auftreten koennen. Das System MUSS auf jeden bekannten Edge Case eine definierte Reaktion haben. Die Grundhaltung ist **fail-safe**: Im Zweifel wird nicht gehandelt, Positionen werden geschuetzt, und das System degradiert kontrolliert statt unkontrolliert weiterzuhandeln.

**Leitprinzipien:**

- Lieber **NO_ACTION** als falscher Trade
- Lieber **HALT** als unkontrolliertes Weiterhandeln
- Lieber **konservativ schliessen** als ungeschuetzte Position halten
- Jeder Edge Case MUSS im Audit-Log protokolliert werden

---

## 2. Markt-Extremfaelle

### 2.1 Flash Crash (> 5% in < 1 Minute)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Quant-Engine: MoveDown > 5% in <= 2 Bars AND volume_ratio > 3.0. Event: `SHOCK_DOWN` (CRITICAL) |
| **Reaktion** | Sofortiger Exit/Stop ohne LLM. DAY_STOPPED. Kill-Switch wenn noetig |
| **Lockout** | shockLockout: Neue Entries fuer X Minuten blockiert (konfigurierbar) |
| **Bestehende Position** | Nur Safety (Stops/Exit), kein Add, kein Scale-Out. Stop greift automatisch |
| **Audit** | CRITICAL Alert + Audit Event mit Trigger-Details |

### 2.2 Gap ueber Stop (Gap Opening)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Pre-Market-Check: Open-Preis < Stop-Level |
| **Reaktion** | Sofortiger Market-Order bei Markteroeffnung (einer der erlaubten Market-Order-Faelle). Event: `GAP_STOP_SLIPPAGE` |
| **Slippage** | Stop Fill zum naechsten Bar-Open (Worst-Case). Slippage wird im Audit-Log dokumentiert |
| **R/R-Check** | Wenn Slippage > 0.5%: Pruefen ob R/R zerstoert, ggf. sofortiger Full-Exit |

### 2.3 Parabolic Run + Plateau + Dump into Close

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Drei-Phasen-Erkennung: 1. `PARABOLIC_ACCEL` (Acceleration + Volume), 2. Exhaustion/Climax Events (Dochte, Volume-Climax), 3. Momentum-Umkehr |
| **Reaktion** | Scale-Out-Ladder MUSS zwingend vor Dump ausgefuehrt werden. Trail tighten bei Exhaustion. EOD Forced Close als ultimative Absicherung |
| **Pass/Fail** | Trade MUSS vor dem Dump skaliert sein (Tranchen bei 1R, 2R, oder bei PARABOLIC_ACCEL-Event) |

### 2.4 Opening Flush + Schnelle Erholung

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Event: `FLUSH_DETECTED` (Range > 2x ATR, Close nahe Low). Dann: `RECLAIM_CONFIRMED` (Close ueber VWAP + EMA-Cluster) |
| **Reaktion** | Setup B: Starter-Entry erst NACH Reclaim. Add erst nach Higher-Low-Bestaetigung + 10m Confirmation nicht DOWN |
| **Schutz** | Kein "Averaging Down" in den Flush. shockLockout blockiert Entry waehrend des Falls |

### 2.5 Halt / LULD Pause

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Indirekt: Ploetzliche Bar-Luecken + keine Updates. Feed zeigt Halt-Indikator |
| **Reaktion** | Agent pausiert: Keine neuen Orders, nur risk-reducing Cancels. Nach Resume: LLM-Re-Assessment. Keine blinden Entries |
| **Bestehende Position** | Stop bleibt beim Broker (GTC). Position wird gehalten bis Resume, dann Re-Evaluation |

### 2.6 LULD ueber Close

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Halt dauert bis nach 15:55 ET (Forced Close Phase vorbei) |
| **Reaktion** | Position kann nicht geschlossen werden. Broker-seitige MOC-Order (Market-on-Close) als Fallback |
| **Naechster Tag** | Am naechsten Morgen pruefen: Position aus Broker-API lesen, Reconciliation, sofort schliessen |
| **Eskalation** | EMERGENCY Alert + Operator-Eingriff erforderlich |

### 2.7 Cannot Liquidate

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Broker lehnt Close-Order ab (Instrument halted, Account-Restriction, etc.) |
| **Reaktion** | EMERGENCY Alert. Kill-Switch. Retry alle 30s. Manueller Eingriff dokumentieren |
| **Eskalation** | SMS + Call + Auto-Escalation. Hoechste Eskalationsstufe |

### 2.8 Explosiver Breakout nach langer Konsolidierung

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | 1m Monitor: Breakout/Acceleration Event nach Coil-Phase |
| **Reaktion** | Arbiter erlaubt schnelle Teilgewinne (Scale-Out). Trail tighten. Parabolic-Protection aktiv wenn Ausbruch explodiert |

---

## 3. Pattern-Failures

### 3.1 Bull Trap / Reclaim ohne Follow-Through

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Setup A/B: Preis reclaimed VWAP/MAs, wird aber sofort wieder abverkauft |
| **Reaktion** | Position wird durch Initial-Stop (unter Flush-Low) geschuetzt |
| **Konsequenz** | Wenn Starter-Position ausgestoppt: Kein Re-Entry fuer dieses Pattern heute |

### 3.2 Fakeout am Apex (Setup C)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Ausbruch aus Coil-Formation wird sofort negiert, Preis kehrt in Formation zurueck |
| **Reaktion** | Exit bei Reentry in die Formation. Am Apex wird grundsaetzlich kein Entry eroeffnet (Fakeout-Schutz) |
| **LLM-Rolle** | LLM red_flag `FAKEOUT_RISK`: Bevorzugt Retest-Entry (State C_RETEST_OPTIONAL). Wenn kein Retest innerhalb N Bars: Abort. ReasonCode: `C_NO_RETEST_ABORT` |
| **Cooldown** | Hard-Stop greift schnell. Cooldown verhindert sofortigen Re-Entry |

### 3.3 Zu tiefer Pullback / Lower-Low

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Ruecksetzer frisst den Impuls vollstaendig, Lower-Low entsteht statt Higher-Low |
| **Reaktion** | Setup invalid. Sofortiger Exit bei Lower-Low. Pattern-State-Machine zurueck auf OBSERVING |
| **ReasonCode** | `B_NEW_LOW_CONTINUES`, `D_NO_HIGHER_LOW` |

### 3.4 Spaeter Entry (> 1R ueber optimalem Level)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Entry erst nach der 2./3. gruenen Kerze statt beim Reclaim. Preis bereits > 1R ueber dem OMS-Struktur-Level-Referenzpreis |
| **Reaktion** | Rules Engine blockiert Entry. ReasonCode: `A_OVEREXTENDED` oder Setup-spezifisch |
| **Begruendung** | Schlechtes R/R -- normaler Pullback wuerde Position bereits ausspuelen |

### 3.5 Low-Volume Breakout

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Breakout ohne Volume-Confirmation (volume_ratio < threshold) |
| **Reaktion** | Entry blockiert. ReasonCode: `C_VOLUME_NOT_CONFIRMED` |
| **Challenger** | S16 (Challenger Suite): Prueft dass Block-Gate korrekt greift |

---

## 4. Infrastruktur-Failures

### 4.1 LLM-Ausfall

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | 3 aufeinanderfolgende Call-Timeouts (je > configurable Timeout) ODER HTTP 5xx ODER Schema-Validation-Failure |
| **Degradation** | QUANT_ONLY Modus: Keine neuen Entries. Bestehende Position deterministisch managen (Trailing, Targets). EOD schliessen |
| **Recovery** | Erster erfolgreicher Call hebt QUANT_ONLY auf. Kein automatischer Re-Entry in vorherige Strategie -- naechster regulaerer Decision-Cycle entscheidet |
| **Monitoring** | LLM Timeout Rate, Schema Invalid Rate, Response Latency p95 |

### 4.2 Halluzinations-Kaskade (3+ verworfene Outputs)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Counter fuer Schema-ungueltige oder inhaltlich verworfene LLM-Outputs. Bei 3+ in Folge: Eskalation |
| **Reaktion** | QUANT_ONLY Modus fuer Rest des Tages. CRITICAL Alert |
| **Anti-Halluzination** | 5 Schichten: Schema-Validation, Bounded Enums, Confidence Plausibility Check, TTL-Mechanismus, Negative Tests |

### 4.3 Daten-Feed-Ausfall

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Keine Ticks/Bars seit > 60s waehrend RTH. DQ-Gate eskaliert |
| **Stufen** | WARNING (90s): Beobachten, keine neuen Entries. CRITICAL (180s): DATA_HALT, risk-reducing Exits. Persistent: Kill-Switch |
| **Degradation** | DATA_HALT: Keine neuen Orders. Nur risk-reducing Exits erlaubt. Bei Persistenz: Kill-Switch |
| **Ownership** | odin-data eskaliert, odin-core entscheidet (Kill-Switch-Ownership) |

### 4.4 Datafeed stottert (intermittierend)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Unregelmae ssige Pausen, dann wieder Daten. DQ_STALE_FEED Events |
| **Reaktion** | Degradation vor Kill-Switch: Groessere Intervalle, konservativere Regeln. Erst bei persistierendem Ausfall (> 60s) wird geschlossen |

### 4.5 Outlier-Bar (Datenfehler)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Einzelne Bar mit absurd hoher Range (> 5% Move), aber OHNE Crash-Signal (Volume normal). Outlier Filter greift |
| **Reaktion** | Bar wird verworfen. Alert `DQ_OUTLIER_REJECTED`. Keine falschen Trades |
| **Reihenfolge** | Crash-Detection laeuft VOR dem Outlier-Filter, damit extreme aber reale Marktbewegungen nicht faelschlich gefiltert werden |
| **Monitoring** | `DQ_OUTLIER_REJECTED` > 3 mal/Tag -> Investigate |

### 4.6 Quote Staleness

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Best Bid/Ask aelter als 15s bei aktivem Markt |
| **Stufen** | 15s: Order-Sperre (keine neuen Orders bis frische Quotes). 30s: DQ-Warnung. 60s: Kill-Switch |

### 4.7 Broker-Reject bei Order

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | OMS erkennt Reject-Event vom Broker |
| **Reaktion** | CRITICAL Alert (insbesondere wenn Position ungeschuetzt -- kein aktiver Stop). Retry begrenzt (max. 3). Ggf. Kill-Switch je nach Severity |
| **Monitoring** | Reject Rate > 0.5% -> Incident |

### 4.8 Partial Fills

| Aspekt | Beschreibung |
|--------|--------------|
| **Entry Partial Fill** | Position mit Teilmenge weiterverwalten. Rest stornieren nach 30s. Stops und Targets proportional anpassen |
| **Target Partial Fill** | OMS trackt gefuellte vs. offene Menge. Stop-Quantity auf Restposition anpassen. Kein neuer Target-Order fuer Restbetrag der Tranche |
| **Stop Partial Fill** | NOTFALL: Verbleibende Stop-Menge + alle Targets sofort als Market-Order schliessen |
| **Idempotenz** | Doppelte Fill-Events DUERFEN KEINE doppelte Position erzeugen. Deduplizierung ueber `orderId + eventSeq` |

### 4.9 Duplicate Fills

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | OMS Idempotenz-Pruefung: `orderId + eventSeq` bereits verarbeitet |
| **Reaktion** | Event wird verworfen (idempotent). Kein doppeltes Position-Update |

### 4.10 System-Neustart bei offener Position

#### V1-Schutz (aktuell)

> **Nachgelagerte Entwicklungsstufe.** In der Anfangsphase ueberwacht der Operator das System aktiv. Automatisches Recovery ist ein Zielbild fuer spaetere Versionen.

| Aspekt | Beschreibung |
|--------|--------------|
| **Sofort-Schutz** | GTC-Stops beim Broker schuetzen offene Positionen automatisch (unabhaengig vom ODIN-Prozess). Dies ist die primaere Sicherheitsschicht bei Prozess-Tod |
| **Operator** | Der Operator sitzt waehrend des Handels aktiv vor dem System und kann bei Prozess-Tod manuell eingreifen (Broker-UI, TWS) |
| **Neustart** | WinSW startet den Prozess automatisch neu. Startup-Recovery erkennt Crash-Zustand |
| **Safe-Mode** | System startet in Safe-Mode (kein Trading, nur Dashboard-Anzeige) |
| **Naechster Tag** | Flat-Check Reconciliation: Positionen aus Broker-API lesen, mit internem State abgleichen. Offene Positionen aus dem Vortag identifizieren und schliessen. Erst nach erfolgreicher Reconciliation: normaler Handel |
| **Kein Intraday-Restart** | ODIN ist in V1 NICHT intraday-restartable fuer Trading. Bei Crash waehrend des Tages: Safe-Mode bis zum naechsten Tag |

#### Zielbild: Crash-Recovery (spaetere Entwicklungsstufe)

Das langfristige Ziel ist ein mehrstufiges Recovery-Konzept:

**1. Externer Watchdog-Prozess:**

- Eigenstaendiger, leichtgewichtiger Prozess, der den ODIN-Hauptprozess ueberwacht (Heartbeat/PID-Monitoring)
- Erkennt Prozess-Tod innerhalb weniger Sekunden

**2. Operator-Alerting (hoechste Prioritaet):**

- Watchdog alertiert den Operator bei Prozess-Tod ueber SMS, Push-Notification oder E-Mail
- Operator-Notification hat hoehere Prioritaet als automatischer Restart — ein informierter Operator kann schneller und sicherer reagieren als ein automatischer Neustart in unbekanntem Zustand

**3. Optionaler automatischer Neustart:**

- Watchdog kann (konfigurierbar) einen Neustart des ODIN-Prozesses versuchen
- Neustart erfolgt ausschliesslich im **Position-Management-Only Recovery Mode**:
    - Keine neuen Entries
    - Offene Positionen erkennen (Reconciliation mit IB-Broker-API)
    - Bestehende GTC-Stops aktiv halten und bei Bedarf nachziehen
    - Forced Close vor EOD sicherstellen (Flat-Garantie)
- Neustart-Versuche sind begrenzt (max. 2 innerhalb von 5 Minuten), danach nur noch Alerting

---

## 5. Trading-Edge-Cases

### 5.1 Overtrading in Chop (RANGE_BOUND)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Viele Rejections/Whipsaws. RANGE_BOUND oder UNCERTAIN Regime. Haeufige VWAP-Crossings |
| **Reaktion** | Gates tighten: RANGE_BOUND/UNCERTAIN = No-Trade Default. Max-Trades und Cooldown verhindern Overtrade. ReasonCode: `RISK_OVERTRADE` |
| **Anti-Overtrade** | Max 5 Round-Trips/Tag (global), Cooldown nach Stop-Out (15--30 Min), Entry nur auf 3m-Takt |

### 5.2 Event-Schock waehrend Pattern

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | News/Orderflow-Schock (Earnings, Fed-Speaker) zerreisst aktives Setup |
| **Reaktion bei Flash-Crash** | Kill-Switch-Logik greift |
| **Reaktion bei moderatem Schock** | LLM-Urgency wird CRITICAL -> sofortige Re-Evaluation. Arbiter entscheidet auf Basis aktueller Daten: EXIT, SCALE_OUT, oder HOLD |

### 5.3 Slippage-Ueberraschung (> 0.5%)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Fill-Preis vs. Order-Preis: Einzelne Slippage > 0.5% |
| **Reaktion** | Stop anpassen. Pruefen ob R/R zerstoert. Ggf. sofortiger Exit wenn R/R unter 1.0 |
| **Eskalation** | Avg. Slippage > 0.2% ueber 5 Trades -> WARNING. > 0.3% ueber 10 Trades -> Max. Positionsgroesse um 25% reduzieren |

### 5.4 Pre-Trade Exposure-Limit (> 80%)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Neue Position wuerde Tageskapital-Exposure > 80% bringen |
| **Reaktion** | Entry blockieren. Nur nach Teil-Exit einer bestehenden Position darf erneut versucht werden |
| **Berechnung** | `neue_exposure = bestehende_exposure + geplante_position. Wenn neue_exposure > Tageskapital x 80% -> blockieren` |

### 5.5 Abnormale Fill-Rate (< 30%)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | < 30% der Orders werden gefuellt (ueber 5 aufeinanderfolgende Versuche) |
| **Reaktion** | Liquiditaetsproblem erkannt. Positionsgroesse halbieren ODER in OBSERVING wechseln |
| **Recovery** | Bis naechster LLM-Zyklus |

### 5.6 Late Ramp (Volumen-Anstieg spaet am Tag)

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | Volume steigt spaet am Tag an |
| **Reaktion** | Entries erlaubt bis entry_cutoff (15:30 ET). Danach nur noch Management bestehender Positionen. Forced Close ab 15:45 ET |

---

## 6. Degradation Modes (Normativ)

Bei teilweisem Systemausfall MUSS das System kontrolliert degradieren statt sofort zu stoppen. Der Kill-Switch ist das letzte Mittel, nicht das erste.

### 6.1 Degradation-Tabelle

| Situation | Mode | Verhalten | Recovery |
|-----------|------|-----------|----------|
| LLM Timeout / Invalid Schema (3x) | **QUANT_ONLY** | Keine neuen Entries. Bestehende Position managen (Trailing, Targets). EOD close | Erster erfolgreicher LLM-Call |
| Data-Feed stale / stottert | **DATA_HALT** | Keine neuen Orders. Nur risk-reducing Exits erlaubt | Feed stabil fuer > 30s |
| Data-Feed persistent > 60s | **KILL_SWITCH** | Alle Positionen schliessen, DAY_STOPPED | Naechster Tag |
| Broker rejects / cannot liquidate | **EMERGENCY** | Kill-Switch, manuelle Eskalation, Retry alle 30s | Manueller Eingriff |
| Halluzinations-Kaskade (3+) | **QUANT_ONLY** (Rest des Tages) | Wie oben, aber kein automatisches Recovery | Naechster Tag |
| Repricing-Degradation | **DEGRADED_EXECUTION** | Repricing-Intervall erhoeht, Zyklen reduziert | Metriken unter Schwellen |

### 6.2 Degradation vor Kill-Switch

**Prinzip:** Bei einzelnen Quality-Gate-Verletzungen wird das System zunaechst degradiert (groessere Intervalle, konservativere Regeln), bevor der Kill-Switch gezogen wird. Nur bei persistierendem Datenausfall (> 60s keine validen Ticks waehrend RTH) oder EMERGENCY-Situationen wird sofort geschlossen.

### 6.3 Degradation-Kaskade

```
NORMAL
  → einzelne DQ-Warnung → WARNING (beobachten)
    → persistente DQ-Verletzung → DATA_HALT (nur risk-reducing)
      → > 60s keine Daten → KILL_SWITCH

NORMAL
  → LLM Timeout → Retry (max 3)
    → 3 Failures in Folge → QUANT_ONLY
      → (Recovery bei erstem erfolgreichen Call)

NORMAL
  → Broker Reject → CRITICAL Alert + Retry
    → Cannot Liquidate → EMERGENCY + Kill-Switch
```

---

## 7. Trading Halts und Session Interrupts

### 7.1 Halt-Erkennung ohne explizite Halt-Daten

Ohne spezielle Halt-Daten wird ein Halt indirekt erkannt:

- Ploetzliche Bar-Luecken + keine Updates -> HALT Mode
- Keine neuen Orders, nur risk-reducing Cancels
- Nach Resume: Re-Assessment, keine blinden Entries

### 7.2 Halt waehrend offener Position

- Broker-seitige GTC-Stops bleiben aktiv
- Keine OMS-Aktionen waehrend Halt moeglich
- Nach Resume: Sofort Situation re-evaluieren (Preis vs. Stop, Regime)

### 7.3 Halt ueber Forced-Close-Phase

- LULD ueber 15:55 ET: Position kann nicht geschlossen werden
- Broker-seitige MOC-Order als Fallback
- Am naechsten Tag: Reconciliation + sofortiges Schliessen
- EMERGENCY Alert + Operator-Eingriff

---

## 8. Multi-Cycle Edge Cases

### 8.1 Budget-Erschoepfung nach Verlust-Zyklus

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | remaining_budget nach realisiertem Verlust zu gering fuer neuen Trade |
| **Reaktion** | Kein weiterer Zyklus. Pipeline geht in FLAT_INTRADAY oder DAY_STOPPED |

### 8.2 Cooling-Off nach Verlustserie

| Aspekt | Beschreibung |
|--------|--------------|
| **Erkennung** | 3 Verluste in Folge (global ueber alle Pipelines) |
| **Reaktion** | 30 Minuten Cooling-Off. Keine neuen Entries fuer alle Pipelines |

### 8.3 Re-Entry in fallendes Messer

| Aspekt | Beschreibung |
|--------|--------------|
| **Schutz** | Re-Entry erfordert: RECLAIM_CONFIRMED + Higher-Low + 10m Confirmation nicht DOWN + Quant Score >= Threshold + LLM nicht VETO/CAUTION |
| **Sizing** | Cycle 2+: Reduzierte Positionsgroesse (Default 60--80% des Standards) |
| **Profit-Protection** | Fruehere Profit-Protection bei Cycle 2+ |

---

## 9. News Security und Input Validation

### 9.1 Grundprinzip

Externe Textdaten (Nachrichtenartikel, Social-Media-Posts, Analystenkommentare) sind potentielle Angriffsvektoren fuer Prompt Injection. Das System behandelt alle externen Texte als **untrusted input**.

### 9.2 Input-Normalisierung

| Massnahme | Details |
|-----------|---------|
| **Whitelist-Felder** | Nur strukturierte Felder werden ans LLM weitergegeben: Headline (max 200 Chars), Source, Timestamp, Sentiment-Score (numerisch) |
| **Keine Rohtexte** | Artikel-Body wird NIEMALS in den LLM-Prompt eingespeist. Nur extrahierte, strukturierte Metadaten |
| **Character-Sanitization** | Nur ASCII + Standard-Unicode. Control Characters, Zero-Width-Zeichen, RTL-Override werden entfernt |
| **Length Limits** | Jedes String-Feld hat ein striktes Max-Length. Truncation bei Ueberschreitung |
| **Prompt-Injection-Schutz** | Externe Texte werden in einem dedizierten JSON-Block markiert (`external_context`), nicht als System/User-Prompt |

### 9.3 Anomalie-Detection bei News-Kontext

- Wenn LLM-Output nach Einspeisung von News-Kontext signifikant vom Baseline-Verhalten abweicht (z.B. ploetzlich CRITICAL urgency bei normalem Markt), wird die Antwort verworfen und ohne News-Kontext wiederholt
- Audit-Trail: Jeder LLM-Aufruf loggt den kompletten Input (inklusive News-Kontext) fuer spaetere Forensik

### 9.4 LLM Negative Tests (Safety Contract)

Folgende Tests MUESSEN bestehen:

| Testfall | Erwartung |
|----------|-----------|
| LLM Output nennt konkrete Stueckzahlen ("buy 100 shares") | **FAIL** -- verwerfen |
| LLM Output nennt konkrete Preise/Stops ("stop at 101.23") | **FAIL** -- verwerfen |
| LLM Output erzwingt Ordertypen ("use market order") | **FAIL** -- verwerfen |
| LLM Output nicht im JSON Schema | **FAIL** -- verwerfen |
| Prompt Injection in News-Kontext ("ignore previous rules") | LLM DARF NICHT auf direkte Anweisungen reagieren |
| Widerspruechliche Daten | LLM MUSS UNCERTAIN/CAUTION waehlen (nicht overconfident) |

---

## 10. Vollstaendige Edge-Case-Referenztabelle

Die folgende Tabelle konsolidiert alle Edge Cases aus allen Quelldokumenten:

| # | Szenario | Erkennung | Reaktion | Eskalation |
|---|----------|-----------|----------|------------|
| 1 | **Flash Crash** (> 5% in < 1 Min) | SHOCK_DOWN Event (vol_ratio > 3) | Sofortiger Exit, DAY_STOPPED | CRITICAL |
| 2 | **Gap ueber Stop** | Pre-Market: Open < Stop | Market-Order bei Eroeffnung | WARNING |
| 3 | **Parabolic + Plateau + Dump** | PARABOLIC_ACCEL + EXHAUSTION_SIGNAL | Scale-Out-Ladder, Trail tighten, EOD close | INFO |
| 4 | **Opening Flush** | FLUSH_DETECTED (Range > 2x ATR) | Setup B: Starter erst nach Reclaim | INFO |
| 5 | **Halt/LULD Pause** | Bar-Luecken, keine Updates | Pause, nur risk-reducing. Nach Resume: Re-Assessment | WARNING |
| 6 | **LULD ueber Close** | Halt nach 15:55 | MOC-Fallback, naechster Tag Reconciliation | EMERGENCY |
| 7 | **Cannot Liquidate** | Broker lehnt Close-Order ab | Kill-Switch, Retry 30s, manueller Eingriff | EMERGENCY |
| 8 | **LLM-Ausfall** (3 Timeouts) | Timeout/5xx Counter | QUANT_ONLY: keine neuen Entries, manage+close | CRITICAL |
| 9 | **Halluzinations-Kaskade** (3+ verworfen) | Verworfene Outputs Counter | QUANT_ONLY fuer Rest des Tages | CRITICAL |
| 10 | **Daten-Feed-Ausfall** (> 60s) | DQ_STALE_FEED | Kill-Switch: alle Positionen schliessen | CRITICAL |
| 11 | **Daten-Feed stottert** | Intermittierende DQ-Events | Degradation → DATA_HALT → ggf. Kill-Switch | WARNING→CRITICAL |
| 12 | **Outlier-Bar** (Datenfehler) | Move > 5%, normales Volume | Bar verwerfen, DQ_OUTLIER_REJECTED | INFO |
| 13 | **Quote Staleness** (> 15s) | Best Bid/Ask veraltet | Order-Sperre → DQ-Warnung → Kill-Switch | WARNING→CRITICAL |
| 14 | **Broker-Reject** | OMS erkennt Reject | CRITICAL Alert, Retry begrenzt, ggf. Kill-Switch | CRITICAL |
| 15 | **Partial Fill auf Entry** | Nicht komplett nach 30s | Rest stornieren, Teilposition weiter, proportionale Anpassung | INFO |
| 16 | **Partial Fill auf Stop** | Stop nur teilweise gefuellt | NOTFALL: Rest + Targets sofort Market schliessen | CRITICAL |
| 17 | **Duplicate Fills** | orderId + eventSeq bereits verarbeitet | Idempotent verwerfen | INFO |
| 18 | **System-Neustart bei offener Position** | Recovery-Check bei Start / Watchdog erkennt Prozess-Tod | V1: GTC-Stops + Operator-Eingriff. Zielbild: Watchdog alertiert Operator + optionaler Recovery Mode (Position-Management-Only, keine neuen Entries). Naechster Tag: Reconciliation | CRITICAL |
| 19 | **Bull Trap / Reclaim ohne Follow-Through** | Setup invalidiert | Initial-Stop schuetzt. Kein Re-Entry fuer dieses Pattern | INFO |
| 20 | **Fakeout am Apex** (Setup C) | Preis kehrt in Formation zurueck | Exit bei Reentry. Cooldown | INFO |
| 21 | **Zu tiefer Pullback / Lower-Low** | Lower-Low statt Higher-Low | Sofortiger Exit, OBSERVING | INFO |
| 22 | **Spaeter Entry** (> 1R ueber optimal) | Preis > OMS-Struktur-Level-Referenzpreis | Entry blockiert | INFO |
| 23 | **Overtrading in Chop** | RANGE_BOUND, viele Rejections | Gates tighten, Cooldown, Max-Trades | WARNING |
| 24 | **Event-Schock waehrend Pattern** | News/Orderflow-Schock | Flash-Crash: Kill-Switch. Moderat: Re-Evaluation | WARNING→CRITICAL |
| 25 | **Slippage-Ueberraschung** (> 0.5%) | Fill vs. Order-Preis | Stop anpassen, ggf. Exit wenn R/R zerstoert | WARNING |
| 26 | **Pre-Trade Exposure > 80%** | Exposure-Check | Entry blockiert | INFO |
| 27 | **Abnormale Fill-Rate** (< 30%) | 5 aufeinanderfolgende Versuche | Groesse halbieren oder OBSERVING | WARNING |
| 28 | **Low-Volume Breakout** | Breakout ohne vol_confirm | Entry blockiert | INFO |
| 29 | **VWAP Whipsaw** | Haeufige VWAP-Crossings | RANGE: No-Trade | INFO |
| 30 | **Gap Up & Fade** | Gap up, dann Abverkauf unter VWAP | Entry nur nach Reclaim. Schneller Exit wenn VWAP nicht haelt | INFO |
| 31 | **Dead-Cat-Bounce** (Flush → Bounce → weiter DOWN) | Reclaim-Gate nicht erfuellt | Setup abbrechen. Kein Averaging Down | INFO |
| 32 | **Morning weak → Afternoon Reversal** | Regime-Shift + Confirmation | Cycle 2 Re-Entry (Setup D), klein | INFO |
| 33 | **Multi-Cycle Budget-Erschoepfung** | remaining_budget < min_trade_risk | Kein weiterer Zyklus | INFO |
| 34 | **3 Verluste in Folge** | Verlust-Counter | 30 Min Cooling-Off global | WARNING |

---

## 11. Challenger-Suite-Abdeckung

Jeder Edge Case in dieser Liste wird durch mindestens ein Szenario der Challenger Suite (Abschnitt 09) abgedeckt:

| Edge Case | Challenger-Szenario |
|-----------|-------------------|
| Opening Spike → Trend | S01 |
| Fake Breakout → Range Day | S02 |
| Flush → V-Reversal → Rally | S03 |
| Flush → Dead-Cat-Bounce → DOWN | S04 |
| Coil → Breakout | S05 |
| Coil → Fakeout → Stop | S06 |
| Parabolic → Plateau → Dump | S07 |
| Slow Trend Day | S08 |
| VWAP Magnet Chop Day | S09 |
| Gap Up & Fade | S10 |
| Gap Down & Recover | S11 |
| News/Shock Dump | S12 |
| Volatility Halt / Datenluecke | S13 |
| Outlier-Bar | S14 |
| LLM Outage mitten im Trade | S15 |
| Broker Reject | S16 |
| Partial Fills | S17 |
| Re-Entry nach Take Profit + Pullback | S18 |
| Afternoon Trendwechsel | S19 |
| End-of-Day Squeeze → Reversal | S20 |

---

## 12. Parameterkatalog (Edge-Case-relevant, konfigurierbar)

| Parameter | Default | Beschreibung |
|-----------|---------|--------------|
| `odin.risk.daily-loss-limit-pct` | 0.10 | Hard-Stop: Max. Tagesverlust |
| `odin.risk.shock-lockout-minutes` | 15 | Lockout nach SHOCK_DOWN Event |
| `odin.risk.cooldown-after-exit-min` | 15--30 | Cooldown nach Exit |
| `odin.risk.cooldown-after-loss-series-min` | 30 | Cooling-Off nach 3 Verlusten |
| `odin.risk.max-slippage-single-pct` | 0.005 | Einzelne Slippage-Schwelle (0.5%) |
| `odin.risk.avg-slippage-warn-pct` | 0.002 | Avg. Slippage Warning (0.2%, 5 Trades) |
| `odin.risk.avg-slippage-reduce-pct` | 0.003 | Avg. Slippage -> Size Reduction (0.3%, 10 Trades) |
| `odin.dq.crash-move-pct` | 0.05 | Move% fuer Crash-Erkennung |
| `odin.dq.crash-volume-ratio` | 3.0 | VolumeRatio fuer Crash-Erkennung |
| `odin.dq.outlier-move-pct` | 0.05 | Move% fuer Outlier-Erkennung |
| `odin.dq.outlier-max-volume-ratio` | 1.2 | Max VolumeRatio fuer Outlier |
| `odin.dq.stale-seconds-warn` | 90 | Warnschwelle keine Bars |
| `odin.dq.stale-seconds-halt` | 180 | Halt-Schwelle keine Bars |
| `odin.dq.quote-staleness-order-block-sec` | 15 | Quote-Staleness: Order-Sperre |
| `odin.dq.quote-staleness-warn-sec` | 30 | Quote-Staleness: DQ-Warnung |
| `odin.dq.quote-staleness-kill-sec` | 60 | Quote-Staleness: Kill-Switch |
| `odin.execution.partial-fill-timeout-sec` | 30 | Timeout fuer Partial Fill Stornierung |
| `odin.llm.max-consecutive-failures` | 3 | Failures bis QUANT_ONLY |
| `odin.llm.hallucination-cascade-threshold` | 3 | Verworfene Outputs bis QUANT_ONLY |
