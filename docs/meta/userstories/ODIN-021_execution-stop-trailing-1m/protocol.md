# Protokoll: ODIN-021 — Stop-Trailing auf 1-Minute-Bar-Close

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (33 Tests in TrailingStopManagerTest)
- [x] ChatGPT-Sparring für Test-Edge-Cases (2 Runden)
- [x] Integrationstests geschrieben (11 Tests in TrailingStopManagerIntegrationTest)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push (commit e044d75, QS-Agent Runde 1)

**Testergebnis:** 169 Tests in odin-execution, alle grün.

## Design-Entscheidungen

### D1: TrailingStopManager als POJO in odin-execution (nicht odin-core)
Die Story schreibt vor: `TrailingStopManager` in odin-execution, `TradingPipeline` (odin-core) routing the 1m-callback.
However, odin-core already has `positionEntryAtr` (frozen) and `trailingStopHighwaterMark`. The `TrailingStopManager`
sits in odin-execution, is created by `OrderManagementService`, and gets called via `onOneMinuteBarClose()`.

### D2: rFloor Parameter Design
The `onOneMinuteBarClose()` method accepts `rFloor` as a parameter because it is computed by the
`ProfitProtectionEngine` (odin-brain, ODIN-014) which lives outside odin-execution. The caller
(TradingPipeline in odin-core) holds both the entry ATR and can compute/pass the rFloor.
This avoids a Fachmodul→Fachmodul dependency.

If ODIN-014 (ProfitProtectionEngine) is not yet implemented, rFloor can default to 0.0 (no floor),
making ODIN-021 work independently.

### D3: Frequency Limiting — lastUpdateMinute Tracking
The spec requires max. 1x per minute. Since `MarketClock.now()` returns an `Instant`,
we truncate to the minute epoch (Instant.truncatedTo(ChronoUnit.MINUTES)) and compare.
This is deterministic and compatible with both Live and Sim modes.

### D4: Cancel+New (not Modify) for Stop Updates
Per story acceptance criteria: "Stop-Order-Anpassung: Cancel + New (nicht Modify)".
This is implemented in `OrderManagementService.adjustStopLoss()` which already performs Cancel+New.
`TrailingStopManager.evaluateOnBarClose()` returns an `Optional<Double>` indicating the new stop level,
then OMS delegates to adjustStopLoss.

### D5: highwaterMark lives in TrailingStopManager
The highwater mark for the *trailing stop computation* lives in `TrailingStopManager` (initialized
at position open). This is distinct from `trailingStopHighwaterMark` in `TradingPipeline` which
tracks the stop level for the 3m decision loop. Both must be consistent.
Decision: `TrailingStopManager.highwaterMark` is the authoritative value for 1m trailing.
`TradingPipeline.trailingStopHighwaterMark` is updated on 3m bars. Both enforce the same principle.

### D6: Integration into OMS vs. Standalone
`TrailingStopManager` is created and owned by `OrderManagementService`. OMS exposes
`onOneMinuteBarClose(intradayHigh, entryAtr, rFloor, now)` which delegates to `TrailingStopManager`
and calls `adjustStopLoss` if an update is warranted. OMS is the single point of broker interaction.

### D7: TrailingStopManager Return Value
Returns an `Optional<Double>` representing the new effective stop level, or empty if no update needed.
This keeps logic pure/testable: caller (OMS) decides to cancel+replace the broker stop.

### D8: Configuration namespace
Per story: `odin.execution.trailing.*`
Added: `trail-atr-factor` (default 2.2 per concept) to `ExecutionProperties.TrailingProperties`.

### D9: Rate-Limit Semantik — Minute bei jeder Evaluierung konsumieren
**Entscheidung:** Das Minuten-Token wird bei JEDER gültigen Evaluierung konsumiert (nicht nur bei
echtem Broker-Update). Das verhindert mehrere Cancel+New-Events innerhalb einer Minute, falls
`onOneMinuteBarClose` irrtümlich mehrfach pro Minute aufgerufen wird.

**Begründung von ChatGPT bestätigt:** Da die Methode konzeptionell nur 1x pro Minute aufgerufen
werden soll (Bar-Close), ist das kein funktionales Problem. In einem Tick-basierten System würde
es legitime Updates innerhalb einer Minute blockieren, aber das ist hier nicht der Anwendungsfall.

### D10: Input Validation — NaN/Infinity Guards
**Problem von ChatGPT identifiziert:** Ohne `Double.isFinite()`-Checks würde `Math.max()` mit NaN
den Highwater-Mark mit NaN poisonen → alle folgenden Berechnungen liefern NaN → Stille Failures.

**Fix:** `Double.isFinite()`-Prüfung für alle numerischen Inputs vor jeder Berechnung.
Invalide Inputs (NaN, Infinity) werden rejected WITHOUT consuming the rate-limit minute.

## Offene Punkte

- ODIN-014 (ProfitProtectionEngine) ist als Dependency gelistet. `rFloor` defaults auf 0.0
  wenn nicht implementiert → backward-compatible.
- TradingPipeline routing (odin-core): Die Story erwähnt TradingPipeline-Callback. Das ist
  out-of-scope für odin-execution — die `onOneMinuteBarClose()` Methode auf OMS ist der
  Integrationspunkt. Das TradingPipeline-Routing ist odin-core scope (ODIN-025).
- Cancel+New "Naked Position" Fenster: Bekannte Lücke < 50ms zwischen Cancel und neuer Order.
  Mit globalem Kill-Switch und IB-Monitoring akzeptable Trade-off gegenüber `modifyOrder`
  (das bei IBKR Race-Conditions bei Trigger+Modify produzieren kann). Für ein separates ADR vormerken.

## ChatGPT-Sparring (2 Runden)

### Runde 1 — Edge Cases für Tests

**Findings:**
1. **NaN/Infinity Guard fehlend:** `Math.max()` mit NaN poisoned den Highwater-Mark dauerhaft.
   → Fix: `Double.isFinite()` für alle numerischen Inputs, + `entryAtr <= 0` Guard.
2. **Rate-Limit-Semantik:** Soll Minute bei Evaluierung konsumiert werden (nicht nur bei Broker-Update)?
   → Entscheidung: JA, konsumieren bei jeder gültigen Evaluierung (Design-Entscheidung D9).
3. **Null 'now' Guard fehlend:** NPE bei `Instant.truncatedTo()` wenn `now == null`.
   → Fix: Null-Check als erste Aktion.

**Umgesetzte Änderungen:**
- TrailingStopManager.java: Input-Validation-Block hinzugefügt (null, non-finite, entryAtr <= 0)
- TrailingStopManagerTest.java: 9 neue Edge-Case-Tests hinzugefügt

### Runde 2 — Rate-Limit-Konsequenzen bei Tick-basierten Systemen

**Finding:** Rate-Limit auf Evaluierung (nicht Broker-Update) ist korrekt für Bar-Close-System,
wäre aber problematisch in einem Tick-System. Da ODIN explizit Bar-Close-basiert ist: accepted.

## Gemini-Review (2 Runden, 3 Dimensionen)

### Dimension 1: Code-Qualität und Korrektheit

**Positiv:**
- Input-Validation gegen NaN/Infinity vorbildlich.
- Single-Threaded-Garantie im JavaDoc dokumentiert.

**Finding (mittleres Risiko):**
- `adjustStopLoss()` feuert Cancel und dann New OHNE auf Cancel-Bestätigung zu warten (async IB).
  Bei Margin-Limit kann IB die neue Stop-Order als "Margin Violation" abweisen während der Cancel
  noch pending ist. Ergebnis: Alte Order cancelled, neue rejected = Naked Position.
  Workaround: Kill-Switch wird bei Order-Rejection-Events ausgelöst (vorhandenes REJECTED-Handling).
  Hinweis: `modifyOrder` wurde bewusst verworfen wegen IB-Race-Conditions bei Trigger+Modify.
- **Entscheidung:** Cancel+New beibehalten (per Story-Anforderung). REJECTED-Event-Handling
  im OMS auf Kill-Switch-Eskalation prüfen — separate Task (außerhalb ODIN-021 Scope).

### Dimension 2: Konzepttreue

**Positiv:**
- Trail-Formel `intradayHigh - trailAtrFactor * entryAtr` korrekt implementiert.
- Highwater-Mark `Math.max(highwater, candidateTrail)` korrekt.
- R-Floor `Math.max(effectiveTrail, rFloor)` korrekt an richtiger Stelle.
- Rate-Limit-Semantik für Bar-Close-System korrekt.

**Keine Abweichungen vom Konzept identifiziert.**

### Dimension 3: Praxis-Lücken (IB-Kontext)

**Finding 1 (umgesetzt): Test-Payload-Assertions zu oberflächlich**
Die `CapturingEventLog` im IntegrationTest hat den Payload komplett verworfen.
→ Fix: `RecordedEvent` um `payload`-Feld erweitert. Test `trailingStopEventIsLoggedOnUpdate`
  prüft jetzt `"reason":"1m-trail"`, `"newStopPrice":%.4f`, `"intradayHigh":%.4f`.

**Finding 2 (umgesetzt): Uninitialized-State Test fehlend**
In `TrailingStopManagerTest` wird in `@BeforeEach` immer `initializeForEntry()` aufgerufen.
Der `HIGHWATER_UNINITIALIZED`-Pfad (ohne vorherige Initialisierung) war nicht explizit getestet.
→ Fix: Test `uninitializedManagerUsesFirstCandidateAsEffectiveTrail` hinzugefügt.

**Finding 3 (nicht umgesetzt — außerhalb Scope): "Crossing the Market"-Validierung**
Kein Check ob `effectiveStop` nahe an oder über dem aktuellen Bid/Ask liegt.
Bei Flash-Crash könnte Stop sofort triggern oder als "Invalid Price" abgelehnt werden.
→ Vorgemerkt für IB-Praxis-Hardening (separates ADR / odin-core scope).

**Finding 4 (akzeptiert): Guard vor `onOneMinuteBarClose` ohne initializeForEntry**
Bestätigt: Der Guard `!positionState.isPositioned() || !initialStopSubmitted` verhindert
Aufrufe korrekt. Der `HIGHWATER_UNINITIALIZED`-Pfad im Manager ist zusätzliche Robustheit.
