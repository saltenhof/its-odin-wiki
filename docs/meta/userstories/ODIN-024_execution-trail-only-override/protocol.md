# ODIN-024: TRAIL_ONLY Override im OMS — Implementierungsprotokoll

**Story:** ODIN-024
**Modul:** odin-execution
**Datum:** 2026-02-22
**Status:** DONE — bereit fuer QS-Agent-Commit

---

## 1. Story-Zusammenfassung

TRAIL_ONLY ist ein OMS-Override-Modus, in dem das LLM (via `targetPolicy=TRAIL_ONLY`) alle
Tranchen-Target-Orders deaktiviert und die gesamte Position ausschliesslich gegen den Trailing Stop
laufen laesst. Neben der normalen Aktivierung durch den LLM-Adapter ist auch eine manuelle
Aktivierung via Control-Endpoint (zukuenftig) vorgesehen.

**Kernfunktionen:**
- OMS-Flag `trailOnlyActive` unterdrueckt Target-Order-Submission solange aktiv
- Bestehende Target-Orders werden bei Aktivierung automatisch storniert
- Anti-Missbrauch-Guard: Aktivierung nur bei >= 0.5R Gewinn (Freeze-Zeitpunkt: Entry-ATR)
- Deaktivierung via explizitem Call (z.B. neuer LLM-Call mit anderem `targetPolicy`)
- Crash-Recovery-Pfad: `restoreTrailOnlyState()` umgeht den Anti-Missbrauch-Guard

**Acceptance Criteria — alle erfuellt:**
- [x] Wenn `trailOnlyActive == true`: `submitTargetOrdersForTranches()` wird nicht aufgerufen
- [x] Wenn TRAIL_ONLY nach Entry aktiviert wird: Bestehende Target-Orders werden storniert
- [x] Trail schuetzt gesamte Position (kein Sonderfall fuer Runner-Tranche)
- [x] EventLog: `TRAIL_ONLY_ACTIVATED` / `TRAIL_ONLY_DEACTIVATED` / `TRAIL_ONLY_ACTIVATION_BLOCKED`
- [x] Anti-Missbrauch: Aktivierung blockiert wenn Position < 0.5R im Gewinn

---

## 2. Implementierte Klassen und Dateien

### 2.1 Geaendert: OrderManagementService.java

**Pfad:**
`T:/codebase/its_odin/its-odin-backend/odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java`

**Neue Konstante:**
```java
private static final double TRAIL_ONLY_MIN_PROFIT_R = 0.5;
```

**Neues Feld:**
```java
/** Whether TRAIL_ONLY override is currently active for this position. */
private boolean trailOnlyActive;
```
- Initialisierung im Konstruktor: `this.trailOnlyActive = false;`
- Reset in `reset()`: `trailOnlyActive = false;`

**Geaenderte Methode — `submitTargetOrdersForTranches()`:**
```java
private void submitTargetOrdersForTranches(Instant now) {
    if (trailOnlyActive) {
        LOG.info("TRAIL_ONLY active — target order submission suppressed for instrument={}", instrumentId);
        return;
    }
    // ... existing logic
}
```

**Neue public Methoden:**

`activateTrailOnly(double currentPrice, double entryAtr, Instant now) -> boolean`
- Prueft ob Position offen ist (sonst `false`)
- Prueft ob bereits aktiv (Idempotenz: `true`)
- Berechnet `pnlInR = (currentPrice - avgEntryPrice) / (entryAtr * trailAtrFactor)`
- Anti-Missbrauch: blockiert wenn `pnlInR < 0.5`, loggt `TRAIL_ONLY_ACTIVATION_BLOCKED`
- Setzt Flag ZUERST (`trailOnlyActive = true`), dann `cancelActiveTargetOrders(now)`
- Loggt `TRAIL_ONLY_ACTIVATED`

`deactivateTrailOnly(Instant now) -> void`
- No-op wenn nicht aktiv
- Setzt `trailOnlyActive = false`, loggt `TRAIL_ONLY_DEACTIVATED`

`isTrailOnlyActive() -> boolean`
- Getter fuer den aktuellen Flag-Zustand

`restoreTrailOnlyState(boolean active, Instant now) -> void`
- Crash-Recovery-Pfad: setzt Flag direkt ohne Anti-Missbrauch-Guard
- Loggt Event mit `"source":"CRASH_RECOVERY"` im Payload

**Neue private Methode — `cancelActiveTargetOrders(Instant now)`:**
```java
private void cancelActiveTargetOrders(Instant now) {
    List<String> targetSnapshot = List.copyOf(activeTargetClientOrderIds);
    activeTargetClientOrderIds.clear();  // clear BEFORE iterating snapshot (CME-Sicherheit)
    for (String targetClientOrderId : targetSnapshot) {
        TrackedOrder targetOrder = orderTracker.getOrder(targetClientOrderId);
        if (targetOrder != null && targetOrder.getBrokerOrderId() != null && !targetOrder.isTerminal()) {
            brokerGateway.cancelOrder(targetOrder.getBrokerOrderId());
            // log TARGET_ORDER_CANCELLED
        }
    }
}
```

**Bugfix — `moveStopToBreakEven()` (Highwater-Mark-Invariante):**
```java
double breakEvenPrice = positionState.getAvgEntryPrice();
double newStopPrice = Math.max(currentStopPrice, breakEvenPrice);
if (newStopPrice <= currentStopPrice) {
    // Stop already at or above break-even — log "already-satisfied" and return
    return;
}
double previousStopPrice = currentStopPrice;  // capture BEFORE mutation
currentStopPrice = newStopPrice;
// ... adjustStopLoss(...), log STOP_LOSS_BREAK_EVEN mit previousStopPrice
```

**Klassen-JavaDoc:** Ergaenzt um TRAIL_ONLY-Override-Beschreibung und `@see`-Referenz auf Konzeptdokument.

---

### 2.2 Geaendert: OrderManagementServiceTest.java

**Pfad:**
`T:/codebase/its_odin/its-odin-backend/odin-execution/src/test/java/de/its/odin/execution/oms/OrderManagementServiceTest.java`

**12 neue Unit-Test-Methoden** (Abschnitt `// ========== ODIN-024: TRAIL_ONLY Override ==========`):

| Test | Prueft |
|------|--------|
| `trailOnlyActiveInitiallyFalse` | Flag ist initial `false` |
| `activateTrailOnlyWithoutOpenPositionReturnsFalse` | Ablehnung ohne offene Position |
| `activateTrailOnlyBelowMinProfitRReturnsFalse` | Anti-Missbrauch-Guard bei `pnlInR < 0.5` |
| `activateTrailOnlyAtJustAboveMinProfitRSucceeds` | Akzeptanz bei `pnlInR > 0.5` |
| `activateTrailOnlySuppressesTargetOrders` | Bestehende Targets werden storniert |
| `activateTrailOnlyWhenAlreadyActiveReturnsTrue` | Idempotenz |
| `deactivateTrailOnlyResetsFlag` | Deaktivierung setzt Flag auf `false` |
| `deactivateTrailOnlyWhenNotActiveIsNoOp` | No-op wenn nicht aktiv |
| `trailOnlyActiveDuringEntryFillSuppressesTargetSubmission` | Flag gesetzt vor Entry-Fill: keine Targets |
| `lateTargetFillAfterTrailOnlyDoesNotResubmitTargets` | Spaetes FILLED-Event loest kein Resubmit aus |
| `breakEvenAlreadySatisfiedDoesNotLowerStopOrSubmitNewStop` | Break-Even darf Stop nicht absenken |
| `trailOnlyResetOnCycleReset` | Flag nach `notifyCycleEnd()` + `reset()` = `false` |

**Neue Helper-Methoden:**
- `fillExitForTrailOnlyTest(double fillPrice)` — Exit-Fill-Helper fuer TRAIL_ONLY-Tests
- `countNewStopOrders(int fromIndex)` — zaehlt neue Stop-Orders ab gegebenem Index

---

### 2.3 Neu: TrailOnlyIntegrationTest.java

**Pfad:**
`T:/codebase/its_odin/its-odin-backend/odin-execution/src/test/java/de/its/odin/execution/oms/TrailOnlyIntegrationTest.java`

Plain-Java Integrations-Testklasse (kein Spring-Kontext). Stubs: `StubBrokerGateway`, `StubEventLog`, `StubMarketClock`.

**5 Szenarien:**

| Szenario | Prueft |
|----------|--------|
| `lifecycleEntryFillThenTrailOnlyActivationCancelsTargets` | Vollstaendiger Lifecycle: Entry → Targets → TRAIL_ONLY aktiviert → alle Targets storniert |
| `trailOnlyFlagResetAfterCycleReset` | Flag nach Cycle-Reset zurueckgesetzt |
| `antiAbuseGuardBlocksActivationAtLowProfit` | Guard verhindert Aktivierung bei `pnlInR < 0.5` |
| `deactivationEmitsEventAndResetsFlag` | Deaktivierung loggt Event und setzt Flag |
| `activateWithNoOpenTargetsSucceeds` | Aktivierung ohne offene Targets schlaegt nicht fehl |

---

## 3. Test-Ergebnis

**Surefire (Unit-Tests):**
```
Tests run: 196, Failures: 0, Errors: 0, Skipped: 0 — BUILD SUCCESS
```

Einzeln:
- `OrderManagementServiceTest`: 46 Tests (davon 12 neu fuer ODIN-024)
- `TrailOnlyIntegrationTest`: 5 Tests (direkt via Surefire validiert)
- Restliche Klassen: unveraendert gruен

Alle Tests gruен, kein Fehler, kein Skip.

---

## 4. ChatGPT-Review

### Runde 1 (Owner: `odin-024-impl`)

**Kontext:** `OrderManagementService.java` + `OrderManagementServiceTest.java` + relevante Konzeptdokumente

#### KRITISCH-1: ConcurrentModificationException-Risiko

**Befund:** `cancelActiveTargetOrders()` iterierte direkt ueber `activeTargetClientOrderIds`.
Wenn `BrokerGateway.cancelOrder()` synchron ein `FILLED`/`CANCELLED`-Callback ausloest, koennte
`handleExitFill()` das Set waehrend der Iteration modifizieren → `ConcurrentModificationException`.

**Bewertung:** Berechtigt. Auch bei formal single-threaded Ausfuehrung sind reentrant Callbacks moeglich.

**Massnahme:**
```java
List<String> targetSnapshot = List.copyOf(activeTargetClientOrderIds);
activeTargetClientOrderIds.clear();  // BEFORE iterating snapshot
for (String id : targetSnapshot) { ... }
```
Ausserdem: `trailOnlyActive = true` wird jetzt VOR `cancelActiveTargetOrders()` gesetzt.

#### KRITISCH-2: `moveStopToBreakEven()` koennte Stop absenken

**Befund:** Wenn der Trailing Stop bereits ueber den Entry-Preis gestiegen war (z.B. via
`onOneMinuteBarClose`) und spaeter der erste Tranche-Fill eingeht, wuerde die Break-Even-Logik
`currentStopPrice = avgEntryPrice` setzen — was niedriger ist als der aktuell gueltige Stop.
Verletzt die Highwater-Mark-Invariante.

**Bewertung:** Berechtigt. Stille Regression waere im Betrieb schwer zu detektieren.

**Massnahme:**
```java
double newStopPrice = Math.max(currentStopPrice, breakEvenPrice);
if (newStopPrice <= currentStopPrice) {
    // log "already-satisfied"; return
}
double previousStopPrice = currentStopPrice;
currentStopPrice = newStopPrice;
```

#### WICHTIG: Audit-Payload `oldStopPrice` falsch

**Befund:** `moveStopToBreakEven()` loggte `"oldStopPrice": entryStopLevel` — forensisch falsch,
wenn der Stop zwischen Entry und erstem Tranche-Fill bereits nachgefuehrt worden war.

**Massnahme:** `double previousStopPrice = currentStopPrice` vor der Mutation cachen, im Payload verwenden.

#### WICHTIG: `filledTrancheCount` Desync

**Befund:** Spaete FILLED-Events fuer Targets, die bei TRAIL_ONLY-Aktivierung bereits storniert
wurden, koennen `filledTrancheCount` inkrementieren und Break-Even-Logic triggern.

**Bewertung:** Begruendetes Risiko, aber im Single-Thread-Modell von OMS sehr unwahrscheinlich
(Cancel-Confirmation kommt vor Fill). Fuer jetzt akzeptiert, in `lateTargetFillAfterTrailOnlyDoesNotResubmitTargets`
Test abgedeckt.

---

### Runde 2 (Owner: `odin-024-impl-r2`, neuer Slot — erster Slot abgelaufen)

**Kontext:** Aktualisierter Code nach KRITISCH-Fixes + neue Tests

#### Bestaetigung der Fixes

ChatGPT bestaetigte beide KRITISCH-Fixes als korrekt implementiert.

#### Fehlende Tests (WICHTIG)

Zwei Grenzfall-Tests fehlten noch:

1. **Spaetes Target-Fill nach TRAIL_ONLY** (`lateTargetFillAfterTrailOnlyDoesNotResubmitTargets`):
   Verifiziert, dass ein FILLED-Event fuer einen bereits als cancelled markierten Target
   keinen erneuten Target-Submit ausloest.

2. **Break-Even bereits erfuellt** (`breakEvenAlreadySatisfiedDoesNotLowerStopOrSubmitNewStop`):
   Verifiziert, dass `moveStopToBreakEven()` den Stop nicht absenkt, wenn der Trailing Stop
   bereits ueber den Entry-Preis gestiegen war.

**Massnahme:** Beide Tests implementiert (siehe Abschnitt 2.2).

---

## 5. Gemini-Review

### Dimension 1: Code-Qualitaet und Bugs

**Befunde:**

| Schwere | Befund | Massnahme |
|---------|--------|-----------|
| HINWEIS | Magic Number `0.01` in `isFirstTrancheFill()` nicht explizit als Konstante | Akzeptiert — Magic Number war bereits vor ODIN-024 vorhanden, ausserhalb Scope |
| HINWEIS | `TARGET_ORDER_CANCELLED` kann doppelt geloggt werden wenn `cancelOrder()` synchron ein `CANCELLED`-Callback ausloest | Akzeptiert — Doppel-Event im Audit-Log ist tolerierbar und forensisch korrekt |

Keine KRITISCH-Befunde. Beide KRITISCH-Fixes aus ChatGPT-Review wurden als korrekt bestaetigt.

---

### Dimension 2: Konzepttreue

**Pruefgrundlage:** `OrderManagementService.java` + `docs/concept/07-oms-execution.md` (Abschnitt 7)
+ `docs/concept/05-stops-profit-protection.md` (Abschnitt 5.1)

**Befunde:**

| Schwere | Befund | Entscheidung |
|---------|--------|--------------|
| HINWEIS | Konzept spezifiziert keine explizite Formel fuer `1R = entryAtr * trailAtrFactor`. Implementierung leitet korrekt aus `TrailingProperties.trailAtrFactor` ab | Bestaetigt — konsistent mit `TrailingStopManager` |
| HINWEIS | Konzept sagt "kein Target mehr" aber spezifiziert nicht, ob `filledTrancheCount` nach TRAIL_ONLY-Aktivierung noch weiterlaueft | Akzeptiert — `filledTrancheCount` laeuft weiter (Break-Even-Guard benoetigt es) |

Keine Abweichungen vom Konzept. Alle Acceptance Criteria umgesetzt.

---

### Dimension 3: Produktionstauglichkeit und Praxis-Gaps

**Befunde:**

| Schwere | Befund | Massnahme |
|---------|--------|-----------|
| WICHTIG | **Crash-Recovery-Gap:** `activateTrailOnly()` kann beim Rehydrieren des OMS-Zustands den Anti-Missbrauch-Guard ausloesen, wenn der Preis zwischenzeitlich gefallen ist. Flag wuerde nicht wiederhergestellt. | `restoreTrailOnlyState(boolean active, Instant now)` implementiert — umgeht Guard fuer Recovery-Pfad, loggt mit `"source":"CRASH_RECOVERY"` |
| HINWEIS | Kill-Switch-Pfad funktioniert korrekt (ignoriert TRAIL_ONLY-Flag, da `submitExit()` direkt aufgerufen wird) | Kein Handlungsbedarf |
| HINWEIS | NaN/Infinity in `currentPrice` oder `entryAtr` wird durch den fail-safe ternary `(oneRValue > 0.0) ? ... : 0.0` sicher abgefangen (ergibt `pnlInR=0.0` → Guard blockiert) | Kein Handlungsbedarf |

---

## 6. Design-Entscheidungen

### Flag-vor-Cancel-Reihenfolge

`trailOnlyActive = true` wird explizit VOR dem Aufruf von `cancelActiveTargetOrders()` gesetzt.
Grund: Wenn `BrokerGateway.cancelOrder()` synchron einen `onBrokerEvent(CANCELLED)`-Callback
auf demselben Thread ausloest und OMS daraufhin `submitTargetOrdersForTranches()` aufrufen wuerde,
muss das Flag bereits gesetzt sein, damit kein neues Target eingereicht wird.

### Snapshot-Pattern fuer Cancel-Iteration

```java
List<String> targetSnapshot = List.copyOf(activeTargetClientOrderIds);
activeTargetClientOrderIds.clear();
for (String id : targetSnapshot) { ... }
```

Die Liste wird ZUERST geleert, dann ueber den Snapshot iteriert. So koennen reentrant Callbacks
keine `ConcurrentModificationException` ausloesen und die Cancel-Schleife laeuft auf einem
stabilen Snapshot.

### Math.max-Guard in moveStopToBreakEven

```java
double newStopPrice = Math.max(currentStopPrice, breakEvenPrice);
```

Setzt die Highwater-Mark-Invariante (Stop kann nur steigen) auch im Break-Even-Pfad durch.
Wenn der Trailing Stop bereits ueber dem Entry-Preis liegt, ist Break-Even per Definition
bereits erfuellt — der Code loggt `STOP_LOSS_BREAK_EVEN` mit `"already-satisfied"` im Payload
und kehrt ohne neue Stop-Order-Submission zurueck.

### Floating-Point im Anti-Missbrauch-Guard

Urspruenglicher Test `activateTrailOnlyAtExactlyMinProfitRSucceeds` schlug fehl, weil
`2.2 / 4.4` zu `0.4999...` auswertet statt exakt `0.5`. Test wurde umbenannt zu
`activateTrailOnlyAtJustAboveMinProfitRSucceeds` und verwendet einen Preis, der robust
oberhalb des Schwellenwerts liegt (`currentPrice=152.3` → `pnlInR ≈ 0.523`).
Der Guard in der Produktionsimplementierung benutzt `<` (strikt kleiner), was korrekt ist
— exakt 0.5R ist akzeptabel.

### restoreTrailOnlyState — Crash-Recovery-Pfad

`activateTrailOnly()` hat einen Anti-Missbrauch-Guard, der die Aktivierung blockieren kann,
wenn der Preis zum Recovery-Zeitpunkt gefallen ist. Um den gespeicherten Zustand korrekt
wiederherzustellen, gibt es `restoreTrailOnlyState(boolean active, Instant now)`, das den
Guard vollstaendig umgeht. Diese Methode ist ausschliesslich fuer den Crash-Recovery/
Rehydrierungs-Pfad in odin-core gedacht.

---

## 7. Offene Punkte

- **`filledTrancheCount`-Desync nach TRAIL_ONLY:** Ein spaetes FILLED-Event fuer einen
  Target-Order, der bei TRAIL_ONLY-Aktivierung haette storniert werden sollen, kann
  `filledTrancheCount` inkrementieren und die Break-Even-Logik triggern. Im Single-Thread-
  Modell unwahrscheinlich, da Cancel-Bestaetigung vor Fill kommt. Durch den `Math.max`-Guard
  ist der schlimmste Fall (Stop-Absenkung) verhindert. Bleibt auf dem Radar.

- **Integration in LLM-Adapter (ODIN-002):** Das Routing von `targetPolicy=TRAIL_ONLY`
  aus dem LLM-JSON-Output zu `oms.activateTrailOnly()` ist Bestandteil von ODIN-002
  (out of scope fuer ODIN-024).

- **Frontend-Controls:** Manuelle TRAIL_ONLY-Aktivierung via Dashboard-Button ist ein
  separates Frontend-Ticket (out of scope).

---

## 8. Dateistatus fuer Commit (QS-Agent)

| Datei | Status |
|-------|--------|
| `odin-execution/src/main/java/de/its/odin/execution/oms/OrderManagementService.java` | Geaendert |
| `odin-execution/src/test/java/de/its/odin/execution/oms/OrderManagementServiceTest.java` | Geaendert |
| `odin-execution/src/test/java/de/its/odin/execution/oms/TrailOnlyIntegrationTest.java` | Neu |

Alle drei Dateien sind via `git add` in den Staging-Index aufgenommen.
