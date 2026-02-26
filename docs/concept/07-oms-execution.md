# 07 -- OMS, Execution Policy, Order-Management, Fill-Handling

> **Quellen:** Fachkonzept v1.5 (Kap. 15: Execution Policy), Master-Konzept v1.1 (OMS, Multi-Tranchen, Stop-Nachfuehrung, Fill-Handling), Build-Spec v3.0 (Kap. 11: Execution/OMS, OHLCV-basierte Execution Policy), Strategie-Sparring (Repricing-Policy, Market-Order-Policy, L2-Execution-Enhancement, TRAIL_ONLY Override)

---

## 1. Grundprinzip (Normativ)

- OMS ist **deterministisch und reproduzierbar** -- gleicher Input fuehrt zum gleichen Output
- Keine Mikrosekunden-Optimierung; Orders werden auf **Decision-Bar-Takt** (3m oder 5m) gesteuert
- LLM liefert maximal **Dringlichkeit** und **Bias** (bounded Enums), aber keine Orderdetails
- OMS operiert **single-threaded pro Pipeline**
- Trailing-Stops und Scaling-Out sind **OMS-Maintenance**, keine TradeIntents

---

## 2. Order State Machine (Normativ)

Orders durchlaufen folgende Zustaende:

```
NEW → SUBMITTED → (PARTIAL_FILLED)* → FILLED
NEW/SUBMITTED → CANCELED
SUBMITTED → REJECTED
```

**Regeln:**

- Jeder Broker-Event wird **idempotent** verarbeitet (`dedupe by orderId + eventSeq`)
- `PARTIAL_FILLED` aktualisiert Stop/Targets proportional
- `REJECTED` ist **CRITICAL**, wenn Position ungeschuetzt ist (sofortige Eskalation)
- Duplicate Fill-Events DUERFEN KEINE doppelte Position erzeugen

---

## 3. Order-Typen nach Situation (Normativ)

### 3.1 Execution Policy

| Situation | Order-Typ | Details |
|-----------|----------|---------|
| **Normal Entry** | Limit Order | Repricing alle 5s (max. 3 Zyklen). Preis = OMS-berechnetes Struktur-Level (siehe Abschnitt 3.2) |
| **Entry nach 15s ohne Fill** | Cancel/Replace | Neuer Limit-Preis = min(aktueller Ask, OMS-Limit-Obergrenze). Max. 3 Cancel/Replace-Zyklen |
| **Entry nach 3 Zyklen ohne Fill** | Abandon | Kein Entry. State zurueck auf SEEKING_ENTRY |
| **Normal Exit (Tranchen)** | Limit Order | Preis = aktueller Bid. Repricing alle 5s |
| **Forced Close (15:45+ ET)** | Market Order | Position MUSS geschlossen werden |
| **Kill-Switch** | Market Order | Notfall-Schliessung, keine Limit-Logik |
| **Stop-Loss** | Stop-Limit | Stop-Trigger = Stop-Level, Limit = Stop - 0.1%. Fallback: Stop-Market nach 5s |

### 3.2 Entry-Preis-Bestimmung durch OMS (Normativ)

Das LLM liefert **keine** konkreten Preisfelder (`entry_price_zone`, `target_price`). Stattdessen liefert es:

- **Setup-Typ** (Enum)
- **Taktik** (`WAIT_PULLBACK` / `BREAKOUT_FOLLOW` / `NO_TRADE`)
- **Urgency** (Enum)

Das OMS bestimmt den Entry-Preis **deterministisch** aus Struktur-Levels:

| Struktur-Level | Quelle |
|---------------|--------|
| Pre-Market High / Low | Aus Pre-Market-Daten (odin-data) |
| Prior Day Close / High / Low | Historische Tagesdaten |
| Erste 1m-Bar High / Low | Nach Close der ersten RTH-Kerze |
| Gap-Fill-Level | Berechnet aus Prior Close vs. Open |
| Runde Zahlen / psychologische Levels | Deterministisch berechnet (z.B. 100, 150, 200) |

**Taktik-zu-Level-Mapping:**

| LLM-Taktik | OMS-Entry-Logik |
|------------|----------------|
| `WAIT_PULLBACK` | Entry bei Ruecklauf zum naechstgelegenen Support-Level (z.B. VWAP, Prior Close, Pre-Market Low) |
| `BREAKOUT_FOLLOW` | Entry ueber dem Breakout-Level (z.B. Pre-Market High, Prior Day High) mit Limit knapp ueber Ausbruchsniveau |
| `NO_TRADE` | Kein Entry — Intent wird nicht erzeugt |

Die OMS-Limit-Obergrenze ergibt sich aus dem naechsthoeherem Struktur-Level plus konfigurierbarem Puffer. Repricing darf diese Obergrenze nicht ueberschreiten.

### 3.3 Market-Order-Policy (Normativ)

Market Orders sind **AUSSCHLIESSLICH** erlaubt bei:

1. **Forced Close** (15:45+ ET) -- Position MUSS geschlossen werden
2. **Kill-Switch** -- Notfall-Schliessung
3. **Stop-Market-Fallback** -- wenn Stop-Limit nach 5s nicht gefuellt
4. **Partial-Fill-Notfall auf Stop** -- Restmenge + alle Targets sofort Market schliessen
5. **Gap-Opening unter Stop** -- Markteroeffnung bei der der Preis unter dem Stop-Level liegt

Jeder Market-Order-Einsatz wird im **Audit-Log** mit Ausnahmegrund klassifiziert. Kein Market-Order fuer regulaere Entries oder planmaessige Exits.

---

## 4. Stop-Market-Fallback-Kriterien (Normativ)

Der Stop-Loss ist als Stop-Limit konfiguriert (Stop-Trigger mit 0.1% Limit-Offset). Wenn der Stop-Limit-Order nicht gefuellt wird:

| Bedingung | Aktion |
|-----------|--------|
| Stop-Limit nicht gefuellt nach 5 Sekunden | Cancel Stop-Limit, ersetze durch Stop-Market |
| Gap-Opening unter Stop-Level (Pre-Market-Check) | Sofort Market-Order bei Markteroeffnung |
| LULD-Halt aktiv | Keine Aktion moeglich, nach Resume sofort Market-Close |

---

## 5. Entry-Order und Post-Fill Exit Split (Normativ)

### 5.1 Single Entry Order

Der Entry erfolgt als **EINE einzelne Order** (Limit Order, siehe Abschnitt 3.1/3.2). Es wird NICHT vorab in Tranchen aufgeteilt.

### 5.2 Post-Fill Exit Split (OCA-Gruppen pro Tranche)

Erst **nach dem Entry-Fill** splittet das OMS die Position in separate **OCA-Exit-Gruppen pro Tranche**. Jede OCA-Gruppe besteht aus genau einem Target und einem Stop -- bei einem Fill storniert IB serverseitig automatisch die andere Order der Gruppe. Dadurch gibt es keine Race Condition.

**Beispiel (3-Tranchen-Modell, 100 Shares gesamt):**

| OCA-Gruppe | Target (Limit Sell) | Stop (Stop-Market) | Shares |
|------------|--------------------|--------------------|--------|
| OCA-1 | Target 1 @ 1R | Stop 1 @ -1R | 40 (40%) |
| OCA-2 | Target 2 @ 2R | Stop 2 @ -1R | 40 (40%) |
| OCA-3 | Runner-Target @ weit (z.B. 5R) | Trailing Stop 3 | 20 (20%) |

**Ablauf nach Entry-Fill:**

1. OMS berechnet Tranchen-Aufteilung (gemaess Profil aus Abschnitt 6)
2. OMS platziert **atomar** alle OCA-Gruppen beim Broker
3. Bei **Target-Fill** einer OCA-Gruppe: IB storniert serverseitig den zugehoerigen Stop — keine manuelle Synchronisation noetig
4. Bei **Stop-Trigger** einer OCA-Gruppe: IB storniert serverseitig das zugehoerige Target
5. Trailing-Policy wird fuer die Runner-Tranche (OCA-3) aktiviert

> "Atomar" heisst: im EventLog als zusammenhaengender Schritt; bei Fehlern Recovery-Mechanismus.

**Vorteile gegenueber einem globalen Stop:**

- **Keine Race Condition:** Broker-seitige OCA-Logik garantiert, dass bei einem Target-Fill nur der zugehoerige Stop storniert wird (nicht der Stop fuer andere Tranchen)
- **Unabhaengige Tranchen:** Jede Tranche hat ihr eigenes R/R-Profil. Tranche 1 kann bei 1R geschlossen werden, waehrend der Runner weiterlaeuft
- **Crash-Sicherheit:** Alle Orders sind GTC beim Broker — ueberleben einen ODIN-Crash

### 5.3 Stop-Nachfuehrung bei OCA-Gruppen

Wenn der Trail-Stop nachgefuehrt wird (Abschnitt 10), MUSS das OMS die Stops aller noch offenen OCA-Gruppen konsistent aktualisieren. Der effektive Stop-Level wird zentral berechnet, aber per Modify-Order in jeder OCA-Gruppe angepasst.

**Sonderfall TRAIL_ONLY:** Bei aktiver TRAIL_ONLY-Policy (Abschnitt 7) werden alle Target-Orders storniert. Die OCA-Gruppenlogik wird aufgeloest — es verbleibt ein einzelner Trailing-Stop auf die gesamte Restposition.

### 5.4 Stop/Target-Konsistenz-Invariante

Es MUSS zu jedem Zeitpunkt gelten:

> "Wenn Position > 0, dann existiert ein wirksamer Schutzmechanismus" (Broker-Stop in mindestens einer aktiven OCA-Gruppe oder interner Stop + sofortige Exit-Policy)

---

## 6. Multi-Tranchen-Modell (Normativ)

### 6.1 Tranchen-Profile

| Variante | Bedingung | Tranchen | Aufteilung |
|----------|----------|----------|------------|
| **Kompakt (3T)** | < 5.000 USD | 3 | 40% @ 1R, 40% @ 2R, 20% Runner |
| **Standard (4T)** | 5.000 -- 15.000 USD | 4 | 30% @ 1R, 30% @ Key-Level, 25% @ 2R, 15% Runner |
| **Voll (5T)** | > 15.000 USD | 5 | 25% @ 1R, 25% @ Key-Level, 25% @ 2R, 15% @ 3R, 10% Runner |

Schwellenwerte und Prozentaufteilungen sind als **Properties konfigurierbar**.

### 6.2 Tranchen-Ausloesung

- **R-Multiples:** 1R, 2R, 3R (konfigurierbar)
- **Parabolic/Exhaustion Events:** Scale-Out-Ladder triggert sofort
- **LLM `scale_out_profile`:** Waehlt Profil (OFF/CONSERVATIVE/STANDARD/AGGRESSIVE)

---

## 7. TRAIL_ONLY Override (Stakeholder-Entscheidung)

Wenn `target_policy = TRAIL_ONLY` vom LLM geliefert wird, MUSS das OMS:

1. Alle offenen Target-Orders **stornieren**
2. Ausschliesslich den **Trailing-Stop** als Exit-Mechanismus verwenden
3. Die **gesamte Restposition** per Trail managen

**Nebenwirkung:** TRAIL_ONLY deaktiviert implizit das Scaling-Out ueber Target-Orders. Dies ist beabsichtigt fuer Trend-Days, bei denen partielle Gewinnmitnahmen den Gesamtprofit schmaelern wuerden.

---

## 8. Repricing-Policy (Normativ)

### 8.1 Standard-Repricing

- Repricing alle **5 Sekunden** in Richtung NBBO
- Max. **3 Cancel/Replace-Zyklen** pro Order
- Repricing DARF die OMS-Limit-Obergrenze (naechstes Struktur-Level + Puffer) NICHT ueberschreiten
- Wenn nach max. Zyklen kein Fill: **Abandon** (Intent zurueck an Arbiter)

### 8.2 Repricing-Degradation

Bei hoher Cancel/Replace-Rate MUSS das Repricing-Verhalten degradiert werden:

| Bedingung | Reaktion | Dauer |
|-----------|----------|-------|
| Cancel/Replace-Rate > 5:1 (ueber 30 Min) | Repricing-Intervall von 5s auf 10s erhoehen | Bis Rate < 3:1 |
| Order/Trade-Ratio > 10:1 (ueber 60 Min) | Repricing auf 1 Zyklus (statt 3), kein Abandon-Retry | Bis Ratio < 6:1 |
| Fill-Rate < 30% (ueber 5 Versuche) | Positionsgroesse halbieren ODER in OBSERVING | Bis naechster LLM-Zyklus |

Die Degradation wird im Audit-Log protokolliert. Parameter normalisieren sich automatisch, wenn die Ratio-Metriken wieder unter den Schwellenwerten liegen.

---

## 9. Fill-Event-Handling (Normativ)

### 9.1 Standard-Fill-Events

| Event | OMS-Reaktion |
|-------|-------------|
| **Target-Tranche gefuellt** | IB storniert serverseitig den zugehoerigen Stop der OCA-Gruppe. OMS aktualisiert interne Position. Verbleibende OCA-Gruppen bleiben unveraendert aktiv |
| **Stop einer OCA-Gruppe getriggert** | IB storniert serverseitig das zugehoerige Target der OCA-Gruppe. OMS aktualisiert interne Position. Wenn alle OCA-Gruppen geschlossen: Position = FLAT |
| **Alle Stops getriggert** | Position ist vollstaendig geschlossen. Alle verbleibenden Target-Orders werden geprueft (sollten durch OCA bereits storniert sein) |
| **Partial Fill auf Target** | OMS trackt gefuellte vs. offene Menge innerhalb der OCA-Gruppe. Kein neuer Target-Order fuer Restbetrag der Tranche |
| **Partial Fill auf Stop** | Verbleibende Stop-Menge + alle Targets sofort als **Market-Order** schliessen (Notfall-Fall) |
| **Partial Fill auf Entry** | Position mit Teilmenge weiterverwalten. Rest stornieren nach 30s. OCA-Exit-Gruppen proportional auf Teilmenge anpassen |

### 9.2 Idempotenz-Regel

OMS MUSS **idempotent** sein: Doppelte Fill-Events duerfen keine doppelte Position erzeugen. Deduplizierung ueber `orderId + eventSeq`.

### 9.3 Broker-Rejections

Broker-Rejections werden als **CRITICAL Alert** behandelt, insbesondere wenn die Position ungeschuetzt ist (kein aktiver Stop). In diesem Fall: sofortige Eskalation, Retry begrenzt, ggf. Kill-Switch.

### 9.4 Partial Fills im Backtest (Bekannte Limitation)

Partial Fills sind im Backtest **nicht modelliert**. Dies ist eine bewusste Entscheidung fuer V1:

- **Praktische Relevanz:** Bei den typischerweise gehandelten Instrumenten (populaere High-Beta-Aktien zu regulaeren Handelszeiten / RTH) sind Partial Fills praktisch irrelevant. Die Liquiditaet dieser Aktien uebersteigt die ODIN-Positionsgroessen um Groessenordnungen
- **Live-Monitoring:** Im Live-Betrieb werden Partial-Fill-Events ueber das Audit-Log erfasst und koennen im Monitoring ausgewertet werden (Haeufigkeit, betroffene Instrumente, Auswirkung auf Execution-Qualitaet)
- **Backtest-Modellierung:** Keine Backtest-Modellierung fuer V1. Sollte sich im Live-Betrieb zeigen, dass Partial Fills bei bestimmten Instrumenten oder Marktphasen relevant werden, kann ein Partial-Fill-Modell nachgeruestet werden

---

## 10. Stop-Nachfuehrung (OMS-Maintenance)

### 10.1 Update-Cadence

Stops werden aktualisiert:

- Auf jedem **1m-Bar-Close** (wenn Trailing aktiv)
- Bei **CRITICAL Monitor Event** (z.B. Blow-off → tighten)
- Nach jedem **Target-Fill** (Quantity-Anpassung)

### 10.2 Berechnungslogik

```
trailCandidate     = currentHigh - (trail_atr_factor x entryAtr)
mfeLockStop        = entry + (mfe_lock_pct x MFE)
runnerTrailCandidate = min(ema9_close, last_higher_low) - buffer
effectiveStop      = max(prevStop, stop0, trailCandidate, mfeLockStop, runnerTrailCandidate)
```

Der `effectiveStop` wird per **Modify-Order** an den Broker gesendet. Die Highwater-Mark-Invariante MUSS eingehalten werden.

---

## 11. Forced Close Policy (Normativ)

- Ab `forcedCloseStart` (Default: 15:45 ET) werden **keine neuen Entries** angenommen
- Offene Positionen werden sukzessive geschlossen:
  - Bevorzugt **aggressive Limit** (nahe Bid)
  - Fallback **Market** (wenn nicht liquidierbar)
- "Cannot Liquidate" → **EMERGENCY Alert** + Kill-Switch + Eskalation

---

## 12. Begriffshierarchie und Multi-Cycle OMS-Handling (Normativ)

### 12.0 Begriffshierarchie: Trade, Cycle, Tranche

| Begriff | Definition | Beispiel |
|---------|-----------|---------|
| **Trade** | Erste Eroeffnung bis vollstaendige Schliessung einer Position in einem Instrument | Long AAPL 10:15 -- 14:30 |
| **Cycle** | Neuer Trade im selben Instrument am selben Tag (nach komplettem Exit und Re-Entry) | Trade 1 (10:15--12:00), Trade 2 = Cycle 2 (13:00--14:30) |
| **Tranche** | Teilstueck innerhalb eines Trades -- sowohl beim Aufstocken (Add) als auch beim Teilverkauf (Scale-Out) | 100 Shares Entry + 50 Shares Add = 2 Tranchen |

**Limits:**

- `maxCyclesPerInstrumentPerDay` — begrenzt Re-Entries im selben Instrument
- `maxTranchesPerTrade` — begrenzt Aufstockungen innerhalb eines Trades

### 12.1 Multi-Cycle Regeln

| Aspekt | Regelung |
|--------|---------|
| **Cycle-Counter** | Pro Pipeline, `cycleNumber` in Events |
| **Unabhaengige Trades** | Jeder Cycle = eigenstaendiger Trade mit eigenen Stops, Targets, P&L |
| **Positionsgroessen** | Cycle 2+: Sizing beruecksichtigt verbrauchtes Tagesbudget |
| **Re-Entry** | Neuer Trade (= neuer Cycle) nach komplettem Exit (FLAT_INTRADAY → SEEKING_ENTRY) |
| **Aufstocken (Scale-Up)** | Add-to-Position bei bestehender Position. Zaehlt als **neue Tranche** im selben Trade, NICHT als neuer Cycle |

### 12.2 Scale-Up OMS-Implikationen (Tranchen-basiert)

| Aspekt | Verhalten |
|--------|-----------|
| **Order-Typ** | Add-to-Position (keine neue Trade-Eroeffnung, sondern neue Tranche im bestehenden Trade) |
| **Entry-Preis** | Aufgestockte Tranche hat eigenen Entry-Preis |
| **Entry-ATR** | Aufgestockte Tranche verwendet ATR zum Zeitpunkt des Aufstockens |
| **Trailing-Stop** | Neue Shares: eigener Entry-ATR als Trail-Basis. Bestehende Shares (Runner): unveraenderter Entry-ATR aus der urspruenglichen Tranche |
| **P&L-Berechnung** | Pro Tranche (unterschiedliche Entry-Preise) |
| **Risk-Budget** | Realisierter P&L aus vorherigen Cycles wird beruecksichtigt |
| **Cycle-Counter** | Aufstocken veraendert den Cycle-Counter **NICHT** — nur ein Re-Entry nach komplettem Exit erzeugt einen neuen Cycle |

---

## 13. Pre-Market-Regeln (Normativ)

| Regel | Wert |
|-------|------|
| Max. Positionsgroesse | 25% Tageskapital (halbe Standardgroesse) |
| Mindest-Spread | < 0.5% (Bid-Ask relativ zum Midpoint) |
| Mindest-Volumen | > 10.000 Stueck in den letzten 30 Min Pre-Market |
| Repricing | Kein Repricing — ein Limit-Order-Versuch (Override der Standard-Execution-Policy) |

---

## 14. Slippage-Management (Normativ)

### 14.1 Per-Trade Tracking

Jeder Fill wird gegen den Intent-Preis verglichen: `slippage = fill_price - intent_price`.

### 14.2 Eskalation

| Schwelle | Aktion |
|----------|--------|
| Avg. Slippage > 0.2% ueber 5 Trades | Warnung an Monitoring |
| Avg. Slippage > 0.3% ueber 10 Trades | Max. Positionsgroesse um 25% reduzieren |
| Einzelne Slippage > 0.5% | Stop anpassen, ggf. sofortiger Exit wenn R/R zerstoert |

---

## 15. L2-Execution-Enhancement (Stakeholder-Entscheidung)

**Kompromiss fuer L2-OrderBook-Daten:**

- **Alpha/Regime/Setups:** Ausschliesslich aus OHLCV (backtestbar)
- **Execution-Verbesserung:** Optional mit L2 live (nicht backtest-kritisch). Spread-Monitoring, Liquiditaets-Check
- **Backtest:** Konservatives Kosten-/Fill-Modell + Stress-Szenarien (kein L2 noetig)

L2-Daten werden nur fuer die Execution-Qualitaet verwendet, nie fuer Signalgebung.

---

## 16. Parameterkatalog (OMS-relevant, konfigurierbar)

| Parameter | Default | Beschreibung |
|-----------|---------|-------------|
| `odin.execution.repricing-interval-ms` | 5000 | Repricing-Intervall in Millisekunden |
| `odin.execution.max-repricing-cycles` | 3 | Maximale Cancel/Replace-Zyklen |
| `odin.execution.stop-limit-offset-pct` | 0.001 | Limit-Offset fuer Stop-Limit-Orders (0.1%) |
| `odin.execution.stop-market-fallback-ms` | 5000 | Timeout fuer Stop-Market-Fallback |
| `odin.execution.forced-close-start` | 15:45 | Beginn der Forced-Close-Phase (ET) |
| `odin.execution.forced-close-end` | 15:55 | Ende der Forced-Close-Phase (ET) |
| `odin.risk.max-position-pct` | 0.50 | Max. Positionsgroesse als Anteil am Tageskapital |
| `odin.risk.cancel-replace-rate-warn` | 5.0 | Cancel/Replace-Rate Warnschwelle |
| `odin.risk.order-trade-ratio-warn` | 10.0 | Order/Trade-Ratio Warnschwelle |
