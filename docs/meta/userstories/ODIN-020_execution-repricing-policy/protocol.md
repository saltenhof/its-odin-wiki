# ODIN-020 Protocol: Repricing Policy fuer Limit-Orders

## Status: COMPLETED

---

## 1. Implementierungsschritte

### Phase 1: Analyse & Design

Gelesen:
- `story.md` — Scope, Akzeptanzkriterien, DoD
- `docs/concept/07-oms-execution.md` — Abschnitt 8: Repricing-Policy (normativ)
- `docs/concept/06-risk-management.md` — Abschnitt 8: Anti-Overtrade / Cancel-Replace-Rate
- `docs/backend/architecture/07-oms.md` — OMS-Architektur
- `OrderManagementService.java`, `ExecutionProperties.java`, `OrderTracker.java`, `TrackedOrder.java`

### Phase 2: Implementierung

**Neue Dateien:**
- `RepricingManager.java` — Kern-Implementierung
- `RepricingManagerTest.java` — 31 Unit-Tests
- `RepricingManagerIntegrationTest.java` — Integrationstests (compile-verified)

**Geaenderte Dateien:**
- `ExecutionProperties.java` — `RepricingProperties` Nested-Record hinzugefuegt
- `OrderManagementService.java` — `RepricingManager` integriert
- `odin-execution.properties` — Defaults fuer Repricing-Properties
- `OrderManagementServiceTest.java`, `RiskGateTest.java`, `ParameterOverrideApplierTest.java`,
  `ParameterOverrideApplier.java`, `PipelineFactoryTest.java` — Breaking change durch neuen Record-Parameter

---

## 2. Design-Entscheidungen

### 2.1 Timer-Implementierung mit MarketClock (nicht Thread.sleep)

**Entscheidung:** `RepricingManager` ist ein passiver POJO, der extern auf jedem Market-Tick via `onMarketTick()` getriggert wird. Keine eigenen Threads oder `Instant.now()`.

**Begruendung:** Sim-Kompatibilitaet erfordert `MarketClock` als einzige Zeit-Quelle. Im Sim-Modus wird `SimClock` verwendet, die jederzeit vorgesprungen werden kann. Ein eigener Thread wuerde den Test-Setup verkomplizieren und die Single-Thread-Garantie pro Pipeline brechen.

### 2.2 Cancel + Replace (nicht Modify)

**Entscheidung:** Jeder Repricing-Zyklus cancelt die bestehende Order und submitted eine neue.

**Begruendung:** IB TWS API unterstuetzt kein Modify. Konzept schreibt Cancel+Replace explizit vor.

### 2.3 Preisberechnung: min(currentAsk, originalLimitPrice)

**Entscheidung:** Der neue Preis ist `min(currentAsk, originalLimitPrice)`.

**Begruendung:** `originalLimitPrice` ist die bei Entry-Submission vom OMS berechnete OMS-Limit-Obergrenze (naechstes Struktur-Level + ATR-Puffer). Repricing darf diese Obergrenze nicht ueberschreiten (Konzept Abschnitt 8.1). `currentAsk` wird als Mindestpreis verwendet, um marktnahe Preise zu erzielen.

### 2.4 Funktionales Interface fuer Client-Order-ID-Generierung

**Entscheidung:** `ClientOrderIdSupplier` Functional Interface, das von OMS mit `this::generateClientOrderId` bespielt wird.

**Begruendung:** Entkoppelt `RepricingManager` von OMS-internem Zustand. RepricingManager bleibt ein reines POJO ohne Abhaengigkeit auf den OMS-Sequenzzaehler.

### 2.5 Interval-Doubling via `intervalDoubled` Flag

**Entscheidung:** `updateIntervalDegradationState()` wird auf jedem Tick aufgerufen (nach Timestamp-Pruning), damit auch eine Erholung ohne neue Events registriert wird. Die Flag-Aktualisierung nach jedem `performCancelReplace()` setzt den Zustand fuer den naechsten Zyklus.

**Begruendung:** Doppelter Aufruf von `pruneOldTimestamps` + `updateIntervalDegradationState` ist idempotent und stellt sicher, dass die Rate-Erholung auch dann erfasst wird, wenn keine neuen Events eintreffen (z.B. lange Pause nach dem Erreichen der Schwelle).

### 2.6 cycleStartTime Reset bei ungueltigem Ask

**Entscheidung:** Wenn `currentAsk <= 0`, wird `cycleStartTime = now` gesetzt, bevor WAITING zurueckgegeben wird.

**Begruendung:** Ohne Reset wuerde nach Erholung der Daten sofort ein Repricing ausgefuehrt (da `elapsedMs` bereits >= Intervall). Das wuerde einen unkontrollierten Burst nach einem Datengltch erzeugen. Mit Reset startet der volle Intervall nach dem letzten gueltigen Tick neu.

---

## 3. Gemini-Review (Drei Dimensionen)

### Dimension 1: Code-Review (Bugs & Implementierungsfehler)

**Finding:** `currentAsk > 0` Guard fehlte — bei ungueltigem Ask (z.B. Datenfeed-Gltch) koennte `Math.min(0, originalLimitPrice)` eine 0-Preis-Order erzeugen.

**Action:** Guard implementiert (`currentAsk <= 0.0` → WAITING + Timer-Reset).

**Finding:** NPE moeglich wenn `TrackedOrder.getLimitPrice()` null — durch Null-Check in `resolveCurrentLimitPrice()` bereits abgefangen.

### Dimension 2: Konzepttreue-Review (KRITISCH)

**Finding KRITISCH:** Konzept Abschnitt 8.2 fordert explizit: "Cancel/Replace-Rate > 5:1 (ueber 30 Min) → Repricing-Intervall von 5s auf 10s erhoehen | Bis Rate < 3:1". Die urspruengliche Implementierung logte nur einen Alert, verdoppelte das Intervall aber nicht.

**Action:** `intervalDoubled`-Flag implementiert mit:
- `cancelReplaceRateAlertThreshold` (Default 5): Schwelle zum Aktivieren der Verdopplung
- `cancelReplaceRateRecoveryThreshold` (Default 3): Schwelle zur Rueckkehr zu normalem Intervall
- Neues Konfig-Property: `odin.execution.repricing.cancel-replace-rate-recovery-threshold=3`
- `computeEffectiveIntervalMs()`: gibt `intervalMs * 2` oder `intervalMs` zurueck
- `updateIntervalDegradationState()`: verwaltet den Zustandsuebergang

**Finding:** Verifikation der Repricing-Parameter: Alert-Payload aktualisiert, um `intervalDoubled`-Status zu enthalten.

### Dimension 3: Praxis-Review (Offene Punkte)

**Finding:** Was passiert nach 3 gescheiterten Repricing-Zyklen — soll auf Market-Order umgestellt werden?

**Action:** Konzept sagt klar "Abandon" (keine Market-Order fuer Entry). Im `protocol.md` als offener Punkt dokumentiert.

**Finding:** Was wenn der Markt sich waehrend Repricing schnell gegen uns bewegt?

**Action:** Der `originalLimitPrice` als Ceiling verhindert, dass wir mit dem Markt mitlaufen. Dokumentiert als bewusstes Design.

---

## 4. ChatGPT-Sparring (Edge Cases)

### 4.1 Was vorgeschlagen wurde

1. **Cancel fails / order already filled at broker:** Wenn die alte Order beim Broker gefuellt wird, bevor unser Cancel ankommt, submitted der Code trotzdem eine neue Order. Die neue Order koennte erneut gefuellt werden → Doppel-Position.
   - **Implementiert:** `notifyFilled()` cancelt jetzt proaktiv `activeBrokerOrderId` vor dem State-Clear. Das verhindert den Double-Fill, wenn der Fill-Event fuer die alte Order eintrifft und eine neue Order bereits aktiv ist.

2. **Race condition zwischen Cancel und Submit:** Fill-Event auf alte Order-ID, waehrend neue Order schon submitted.
   - **Implementiert:** Durch den proaktiven Cancel in `notifyFilled()` (gleiche Fix wie #1). Minimum containment fuer V1.

3. **Broker rejects replacement submission (null return):** Code setzte `SUBMITTED`-Status auch bei `null`-Rueckgabe. Zukuenftige Cancels wuerden dann nicht ausgefuehrt (`activeBrokerOrderId != null`-Guard).
   - **Implementiert:** Null-Guard nach `submitOrder()`: wenn `null`, TrackedOrder als `REJECTED` markieren, `REPRICE_SUBMIT_FAILED`-Event loggen, `ABANDON` zurueckgeben.

4. **Concurrent `onMarketTick()` und Fill-Notification:** Bei echter Parallelitaet NPE moeglich (`cycleStartTime` wird zu `null` geraeumt waehrend tick-Thread sie liest).
   - **Nicht implementiert in V1 — dokumentiert als Known Limitation.** Die Architektur garantiert Single-Thread-Zugriff pro Pipeline. Fill-Events muessen auf den Pipeline-Thread marshaled werden. Diese Invariante wird nicht in RepricingManager selbst durchgesetzt.

5. **cycleStartTime nicht zurueckgesetzt bei ungueltigem Ask:** Nach einem Datengltch wuerde sofort repriced.
   - **Implementiert:** `cycleStartTime = now` bei `currentAsk <= 0` (Timer-Reset, kein Burst nach Recovery).

### 4.2 Was verworfen wurde

- **`notifyFilled(String orderId)` — order-aware fill notification:** ChatGPT schlug vor, `notifyFilled()` mit Order-ID zu parametrisieren. Verworfen fuer V1: Die OMS-Integration wird den Fill-Event auf den korrekten Client-Order-ID pruefen (idempotenz auf OMS-Ebene). RepricingManager bleibt einfach.
- **`CANCEL_REQUESTED` Status:** Als Zwischenstatus zwischen Cancel und Broker-Bestaetigung. Verworfen: zu viel Komplexitaet fuer V1, OMS-Status ist bereits OMS-Verantwortlichkeit.
- **Thread-Ownership-Assertion:** Laufzeitpruefung welcher Thread `onMarketTick()` aufruft. Verworfen: Architektur-Invariante, nicht RepricingManager-Verantwortung. Kann spaeter im Integration-Test-Layer verifiziert werden.

---

## 5. Offene Punkte (V2+)

1. **Market-Order nach gescheitertem Repricing:** Konzept sagt Abandon → kein Entry. In V2 koennte eine Market-Order als letzter Entry-Versuch erwaegt werden (Stakeholder-Entscheidung).
2. **Exponentielle Backoff-Strategie:** Statt einfacher Verdopplung koennte ein exponentieller Backoff bei wiederholten Fehlern implementiert werden.
3. **Order/Trade-Ratio > 10:1:** Konzept Abschnitt 8.2, zweite Zeile — "Max. 1 Zyklus statt 3, kein Abandon-Retry". Noch nicht implementiert, erfordert globalen Order/Trade-Counter (ausserhalb RepricingManager).
4. **Fill-Rate < 30%:** Konzept Abschnitt 8.2, dritte Zeile — Positionsgroesse halbieren. Erfordert Cross-Activation-Tracking.

---

## 6. Test-Uebersicht

| Test | Typ | Abdeckung |
|------|-----|-----------|
| `inactiveByDefault` | Unit | isActive() = false initial |
| `activeAfterActivate` | Unit | activate() setzt active = true |
| `returnsWaitingBeforeIntervalElapses` | Unit | Kein Repricing vor Intervall |
| `repricedAfterIntervalElapsed` | Unit | REPRICED nach Intervall |
| `cancelAndReplaceIsPerformedOnRepricing` | Unit | Cancel+Submit je Zyklus |
| `newPriceIsMinOfAskAndOriginalCeiling` | Unit | Preis = min(ask, ceiling) |
| `newPriceIsClampedToOriginalCeilingWhenAskIsHigher` | Unit | Preis nie > ceiling |
| `repricingEventLoggedOnEachCycle` | Unit | REPRICING_ATTEMPT Event |
| `repricingEventContainsOldAndNewPrice` | Unit | Payload oldPrice+newPrice |
| `threeRepricingCyclesThenAbandon` | Unit | 3 Zyklen → ABANDON |
| `abandonCancelsRemainingOrder` | Unit | Letzter Cancel bei Abandon |
| `abandonLogsEntryAbandonedEvent` | Unit | ENTRY_ABANDONED Event |
| `stopLevelNotAffectedByRepricing` | Unit | Nur ENTRY LIMIT BUY Orders |
| `notifyFilledStopsRepricing` | Unit | Fill stoppt Repricing |
| `returnsFilledAfterNotifyFilled` | Unit | FILLED nach notifyFilled |
| `noCancelReplaceAfterFill` | Unit | Kein weiterer Zyklus nach Fill |
| `fillAfterFirstRepricingCycleStopsImmediately` | Unit | Fill nach Zyklus 1 stoppt sofort |
| `notifyFilledCancelsActiveReplacementToPreventDoubleFill` | Unit | Protective Cancel bei Fill |
| `abandonOnNullBrokerOrderIdFromSubmit` | Unit | Null-Rueckgabe → ABANDON |
| `cancelReplaceRateAlertTriggeredAtThreshold` | Unit | Alert bei Schwelle |
| `cancelReplaceRateNotAlertedBelowThreshold` | Unit | Kein Alert unter Schwelle |
| `cancelReplaceCountWindowPrunesOldEntries` | Unit | Rolling-Window-Pruning |
| `intervalIsDoubledWhenRateExceedsAlertThreshold` | Unit | intervalDoubled = true |
| `doubledIntervalDelaysNextRepricingCycle` | Unit | Verdoppeltes Intervall wirkt |
| `intervalRestoredWhenRateDropsBelowRecoveryThreshold` | Unit | Erholung des Intervalls |
| `zeroAskSkipsRepricingAndReturnsWaiting` | Unit | currentAsk=0 → kein Repricing |
| `negativeAskSkipsRepricingAndReturnsWaiting` | Unit | negativer Ask → kein Repricing |
| `invalidAskResetsTimerToPreventImmediateBurstOnRecovery` | Unit | Timer-Reset bei ungueltigem Ask |
| `returnsInactiveWhenNotActivated` | Unit | INACTIVE ohne activate() |
| `notifyCancelledDeactivatesManager` | Unit | External Cancel deaktiviert Manager |
| Integrationstests (RepricingManagerIntegrationTest) | Integration | End-to-End mit OMS + OrderTracker |

---

## 7. Konfiguration

```properties
odin.execution.repricing.interval-ms=5000
odin.execution.repricing.max-cycles=3
odin.execution.repricing.cancel-replace-rate-alert-threshold=5
odin.execution.repricing.cancel-replace-rate-recovery-threshold=3
```

---

## 8. Test-Ergebnis

- **Unit-Tests (odin-execution):** 111 passed, 0 failures
- **odin-core Tests:** 79 passed, 0 failures
- **odin-app Tests:** 219 passed, 0 failures
- **Compile:** SUCCESS (odin-execution, odin-core, odin-app)
- **Integration-Tests:** Kompiliert, konnten wegen Sandbox-Einschraenkungen nicht via Failsafe ausgefuehrt werden

---

## 9. QA-Review Round 1 — Remediation (2026-02-21)

### F-001 — Fehlschlagender Integrationstest (KRITISCH) — BEHOBEN

**Root Cause:** `fillStopsRepricing_noDanglingTimer` rief `onRepricingTick()` zweimal auf demselben Clock-Stand auf. Der erste Aufruf lieferte korrekt REPRICED und setzte `cycleStartTime = now` zurueck. Der zweite Aufruf (identischer Zeitpunkt, `elapsedMs = 0`) lieferte korrekt WAITING — aber der Test erwartete REPRICED.

**Fix:** Der doppelte Aufruf wurde zu einem einzelnen Aufruf mit Capture in eine lokale Variable umgebaut. Das Ergebnis wird nach dem ersten Aufruf geprueft. Der Test-Intent (kein Dangling-Timer nach Fill) bleibt vollstaendig erhalten: Nach der Fill-Simulation prueft der Test weiterhin, dass:
- `RepricingManager.isActive() == false`
- Weitere Ticks liefern FILLED
- Keine zusaetzlichen Cancels oder Submits entstehen

**Geaenderte Datei:** `RepricingManagerIntegrationTest.java`, Methode `fillStopsRepricing_noDanglingTimer`

### F-002 — Namespace-Abweichung (MEDIUM) — Option B gewaehlt

**Entscheidung:** `odin.execution.repricing.*` bleibt unveraendert. Die in Story-DoD (2.1) dokumentierte Konvention `odin.execution.oms.repricing.*` wird als akzeptierte Abweichung markiert.

**Begruendung:**

1. **Konzept-Dokument ist inkonsistent:** Das normative Konzept (`07-oms-execution.md`, Abschnitt 16 "Parameterkatalog") schreibt selbst eine Flat-Notation ohne `oms`-Ebene vor: `odin.execution.repricing-interval-ms`. Weder die Story noch das Konzept sind eindeutig.

2. **Architektonische Konsistenz:** Alle anderen `ExecutionProperties`-Sub-Records verwenden `odin.execution.<name>.*` ohne `oms`-Zwischenebene: `risk`, `limits`, `tranchen`, `stop`, `costs`. Ein `odin.execution.oms.repricing.*` wuerde eine inkonsistente Tiefen-Asymmetrie erzeugen.

3. **Modulgrenzen:** Der Konfigurationsprefix `odin.execution.*` ist der Modul-Namespace fuer `odin-execution`. Die Einfuehrung von `oms` als Sub-Namespace wuerde implizieren, dass zukuenftig auch `odin.execution.risk-gate.*` oder `odin.execution.order-tracker.*` moeglich waeren — das waere ein Praezedenzfall fuer inkonsistente Tiefen-Strukturen.

4. **Kein Mehrwert:** Da RepricingManager Teil des OMS ist (Package `de.its.odin.execution.oms`), ist die OMS-Zuordnung bereits durch den Java-Package-Schnitt klar. Ein zusaetzliches `oms`-Segment in der Properties-Hierarchie bringt keinen Informationsmehrwert.

**Fazit:** `odin.execution.repricing.*` ist die konzept-konforme und architektonisch konsistente Wahl. Story-DoD-Abweichung akzeptiert und begruendet dokumentiert.

---

## 10. QA-Review Round 2 — Abschluss (2026-02-21)

**Ergebnis: PASS**

- F-001: BEHOBEN — `fillStopsRepricing_noDanglingTimer` auf Single-Call mit lokalem Result umgebaut. 34 Integrationstests GRUEN, BUILD SUCCESS (Primary: `mvn verify`).
- F-002: AKZEPTIERT — Namespace-Entscheidung begruendet dokumentiert (Abschnitt 9).
- F-003: BEHOBEN — Integrationstests laufen und sind gruen.
- F-004: AKZEPTIERT — `MIN_EVENTS_FOR_RATE_ALERT` korrekt deklariert, kein Fix erforderlich.

**ODIN-020 ist vollstaendig abgeschlossen.**
