# 07 -- OMS, Execution Policy, Order-Management, Fill-Handling

---

## 1. Grundprinzip (Normativ)

- OMS ist **deterministisch und reproduzierbar** -- gleicher Input fuehrt zum gleichen Output
- Keine Mikrosekunden-Optimierung; Orders werden auf **1m/3m-Takt** gesteuert
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
| **Normal Entry** | Limit Order | Repricing alle 5s (max. 3 Zyklen). Preis = `entry_price_zone.ideal` |
| **Entry nach 15s ohne Fill** | Cancel/Replace | Neuer Limit-Preis = min(aktueller Ask, `entry_price_zone.max`). Max. 3 Cancel/Replace-Zyklen |
| **Entry nach 3 Zyklen ohne Fill** | Abandon | Kein Entry. State zurueck auf SEEKING_ENTRY |
| **Normal Exit (Tranchen)** | Limit Order | Preis = aktueller Bid. Repricing alle 5s |
| **Forced Close (15:45+ ET)** | Market Order | Position MUSS geschlossen werden |
| **Kill-Switch** | Market Order | Notfall-Schliessung, keine Limit-Logik |
| **Stop-Loss** | Stop-Limit | Stop-Trigger = Stop-Level, Limit = Stop - 0.1%. Fallback: Stop-Market nach 5s |

### 3.2 Market-Order-Policy (Normativ)

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

## 5. Bracket-Logik (Normativ)

Nach erfolgreichem Entry-Fill MUSS das OMS **atomar** ausfuehren:

1. **Schutzstop** platzieren (gesamte Restposition)
2. **Target-Orders** platzieren (je Tranche eine separate Limit-Sell-Order)
3. **Trailing-Policy** aktivieren

> "Atomar" heisst: im EventLog als zusammenhaengender Schritt; bei Fehlern Recovery-Mechanismus.

### 5.1 Kein OCA fuer Stop/Target-Kombination

- **Stop-Loss-Order:** Immer als eigenstaendige Stop-Order auf die **gesamte Restposition**. NICHT in einer OCA-Gruppe mit Targets. GTC beim Broker (ueberlebt Crash). Wird bei jedem Teil-Exit auf die verbleibende Stueckzahl angepasst (Reduce-Only)
- **Target-Orders:** Jede Tranche als separate Limit-Sell-Order mit exakter Stueckzahl. Targets sind untereinander unabhaengig (keine OCA). Werden durch OMS-Logik verwaltet, nicht durch Broker-seitige Gruppenlogik

**Begruendung:** OCA wuerde bei einem Target-Fill den Stop canceln -- das waere fatal. Die Synchronisation zwischen Stop und Targets erfolgt ausschliesslich ueber die interne OMS-Logik, die nach jedem Fill-Event atomar die Restposition und offene Orders abgleicht.

### 5.2 Stop/Target-Konsistenz-Invariante

Es MUSS zu jedem Zeitpunkt gelten:

> "Wenn Position > 0, dann existiert ein wirksamer Schutzmechanismus" (Broker-Stop oder interner Stop + sofortige Exit-Policy)

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
- Repricing DARF `entry_price_zone.max` NICHT ueberschreiten
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
| **Target-Tranche gefuellt** | Stop-Quantity auf Restposition reduzieren. Stop-Preis ggf. anpassen (Tranche 1 → Break-Even). Alle verbleibenden Targets behalten ihre Stueckzahl |
| **Stop getriggert** | Alle offenen Target-Orders sofort stornieren. Position ist geschlossen |
| **Partial Fill auf Target** | OMS trackt gefuellte vs. offene Menge. Stop-Quantity auf Restposition anpassen. Kein neuer Target-Order fuer Restbetrag der Tranche |
| **Partial Fill auf Stop** | Verbleibende Stop-Menge + alle Targets sofort als **Market-Order** schliessen (Notfall-Fall) |
| **Partial Fill auf Entry** | Position mit Teilmenge weiterverwalten. Rest stornieren nach 30s. Stops und Targets proportional anpassen |

### 9.2 Idempotenz-Regel

OMS MUSS **idempotent** sein: Doppelte Fill-Events duerfen keine doppelte Position erzeugen. Deduplizierung ueber `orderId + eventSeq`.

### 9.3 Broker-Rejections

Broker-Rejections werden als **CRITICAL Alert** behandelt, insbesondere wenn die Position ungeschuetzt ist (kein aktiver Stop). In diesem Fall: sofortige Eskalation, Retry begrenzt, ggf. Kill-Switch.

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

## 12. Multi-Cycle OMS-Handling (Normativ)

| Aspekt | Regelung |
|--------|---------|
| **Cycle-Counter** | Pro Pipeline, `cycleNumber` in Events |
| **Unabhaengige Trades** | Jeder Zyklus = eigenstaendiger Trade mit eigenen Stops, Targets, P&L |
| **Positionsgroessen** | Zyklus 2+: Sizing beruecksichtigt verbrauchtes Tagesbudget |
| **Re-Entry** | Neuer Trade nach komplettem Exit (FLAT_INTRADAY → SEEKING_ENTRY) |
| **Aufstocken (Scale-Up)** | Add-to-Position bei bestehender Position. Aufgestockter Teil hat eigenen Entry-Preis und Entry-ATR |

### 12.1 Scale-Up OMS-Implikationen

| Aspekt | Verhalten |
|--------|-----------|
| **Order-Typ** | Add-to-Position (keine neue Trade-Eroeffnung) |
| **Entry-Preis** | Aufgestockter Teil hat eigenen Entry-Preis |
| **Entry-ATR** | Aufgestockter Teil verwendet ATR zum Zeitpunkt des Aufstockens |
| **Trailing-Stop** | Neue Shares: eigener Entry-ATR als Trail-Basis. Bestehende Shares (Runner): unveraenderter Entry-ATR aus Zyklus 1 |
| **P&L-Berechnung** | Pro Tranche (unterschiedliche Entry-Preise) |
| **Risk-Budget** | Realisierter P&L aus Zyklus 1 wird beruecksichtigt |
| **Cycle-Counter** | Aufstocken zaehlt als eigener Zyklus |

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
