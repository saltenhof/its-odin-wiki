# ODIN-032 — Trade Intent HMAC Signing: Implementierungsprotokoll

**Story:** ODIN-032 — Audit: TradeIntent HMAC Signing
**Datum:** 2026-02-22
**Agent:** impl-odin032
**Status:** BEREIT FÜR QS

---

## 1. Umgesetzte Akzeptanzkriterien

| AC | Beschreibung | Status |
|----|-------------|--------|
| AC-1 | `TradeIntent` Record erhält neues Feld `String hmacSignature` | ✅ |
| AC-2 | Signing Payload: alle execution-kritischen Felder (inkl. targetPrice, quantScore, regime, runId) | ✅ |
| AC-3 | Secret Key ausschließlich aus ENV `ODIN_HMAC_SECRET` | ✅ |
| AC-4 | `RiskGate` validiert HMAC vor Order-Weiterleitung | ✅ |
| AC-5 | Invalide Signatur: Order REJECT + ALERT Event `HMAC_INVALID` | ✅ |
| AC-6 | Fehlender Secret: Unsigned Mode (Warning, kein Exception-Crash) | ✅ |

---

## 2. Geänderte Dateien

### odin-api
- **`TradeIntent.java`** — 13. Record-Komponente `String hmacSignature` hinzugefügt; neue Methode `toSigningPayload()` als Single-Source-of-Truth für den canonischen Signing-Payload (versioned, `ODIN_TI|v=1|` Prefix)

### odin-brain
- **`IntentSigner.java`** (NEU) — HmacSHA256-Signing via `intent.toSigningPayload()`, Unsigned-Mode-Fallback, `Objects.requireNonNull`, defensive key copy
- **`IntentSignerFactory.java`** (NEU) — Factory liest `ODIN_HMAC_SECRET` aus Environment
- **`DecisionArbiter.java`** — `IntentSigner` als Dependency injiziert; `buildTradeIntent()` signiert Intent nach Erstellung
- **`pom.xml`** — `odin-execution` als `test`-scope Dependency für Cross-Modul-Integrationstests

### odin-execution
- **`IntentVerifier.java`** (NEU) — HmacSHA256-Verification via `intent.toSigningPayload()`, Constant-Time-Compare via `MessageDigest.isEqual`, Längenvalidierung (64 Hex-Chars), `Objects.requireNonNull`, Unsigned-Mode-Fallback
- **`IntentVerifierFactory.java`** (NEU) — Factory liest `ODIN_HMAC_SECRET` aus Environment
- **`RiskGate.java`** — `IntentVerifier` als 5. Konstruktorparameter; Check-Sequenz: HMAC-Verify → Context-Invariant (`instrumentId`-Match) → sonstige Risk-Checks

### odin-core
- **`PipelineFactory.java`** — `IntentSignerFactory.create()` und `IntentVerifierFactory.create()` werden aufgerufen; beide werden in `DecisionArbiter` bzw. `RiskGate` injiziert

### Tests (alle Dateien aktualisiert/neu)
- **`IntentSignerTest.java`** (NEU) — 16 Unit-Tests inkl. 64-Char-Längen-Check, Null-Guard, Determinismus, alle payload-relevanten Felder
- **`IntentVerifierTest.java`** (NEU) — 17 Unit-Tests inkl. Malformed-Length-Reject, Null-Guard, VerificationResult-Factories
- **`IntentSignerVerifierIntegrationTest.java`** (NEU, odin-brain) — 11 End-to-End-Tests: Happy Path, falsches Secret, 6 Tamper-Tests (entryPrice, stopLevel, targetPrice, quantScore, runId, instrumentId), Unsigned-Mode
- **`RiskGateHmacIntegrationTest.java`** (NEU, odin-execution) — 4 End-to-End-Tests: korrekte Signatur → Approval, null Signatur → HMAC_INVALID Event, Tamper → HMAC_INVALID Event, Unsigned Mode → kein HMAC_INVALID

---

## 3. Architektur-Entscheidungen

### Payload-Canonicalization in odin-api
**Problem:** `buildPayload()` war ursprünglich in `IntentSigner` (odin-brain) **und** `IntentVerifier` (odin-execution) dupliziert. Jede Feldänderung hätte beide Stellen kaputt machen können.

**Lösung:** `TradeIntent.toSigningPayload()` in `odin-api` als Single-Source-of-Truth. Beide Fachmodule delegieren dorthin. Kein Krypto-Code in `odin-api`, nur canonische Stringaufbereitung.

### Erweiterter Payload (v=1)
**Problem:** Ursprünglicher Payload enthielt nur 6 Felder (`instrumentId|direction|intentType|entryPrice|stopLevel|marketTime`). `targetPrice`, `quantScore`, `regime`, `runId` beeinflussten Execution-Entscheidungen, waren aber **nicht** in der Signatur.

**Lösung:** Payload erweitert auf alle execution-kritischen Felder:
```
ODIN_TI|v=1|instrumentId|runId|direction|intentType|regime|quantScore|entryPrice|stopLevel|targetPrice|marketTime
```

Versioniertes Prefix ermöglicht spätere Payload-Evolution ohne Silent-Breaks.

### Test-Scope-Dependency odin-brain → odin-execution
**Rationale:** Cross-Modul-Integration-Tests (`IntentSignerVerifierIntegrationTest`) benötigen beide Klassen. Da Fachmodul→Fachmodul in Production verboten ist, wurde `odin-execution` als `<scope>test</scope>` Dependency in `odin-brain/pom.xml` aufgenommen. Keine zyklische Production-Abhängigkeit, Maven Reactor-Order bleibt stabil.

---

## 4. ChatGPT Review — Round 1 (Findings und Umsetzung)

| Finding | Severity | Aktion |
|---------|----------|--------|
| Payload umfasst nicht alle execution-kritischen Felder (`quantScore`, `targetPrice`, `regime`, `runId`) | CRITICAL | ✅ Umgesetzt — alle 4 Felder in Payload aufgenommen |
| Fail-open Unsigned Mode ist Operationsrisiko | IMPORTANT | ✅ Akzeptiert — Logging-Warning bleibt, kein Fail-Closed in v0.1 (Kill-Switch vorhanden) |
| `instrumentId`-Invariant-Check fehlt in RiskGate | IMPORTANT | ✅ Umgesetzt — Check 0b in `RiskGate.evaluate()` |
| Hex-Längenvalidierung fehlt im Verifier | IMPORTANT | ✅ Umgesetzt — 64-Chars-Check vor HMAC-Vergleich |
| Payload-Duplikation brain↔execution | IMPORTANT | ✅ Umgesetzt — nach odin-api verlagert |
| `isUnsignedMode`-Warning nicht einmal-only | IMPORTANT | Akzeptiert — Warnung per Instanz ist korrekt für Pro-Pipeline POJOs; Factory warnt einmalig per JVM |
| `Objects.requireNonNull` für `sign()`/`verify()` | HINT | ✅ Umgesetzt |
| Test-Abdeckung für jetzt-signierte Felder | HINT | ✅ Umgesetzt — Tamper-Tests für alle 6 payload-relevanten Felder |

## 5. ChatGPT Review — Round 2 (Nachfragen)

- **Payload-Reihenfolge**: Empfehlung Kontext/Identität zuerst, dann Typ, dann Preise, dann Zeit → umgesetzt
- **Fail-open vs. Fail-closed**: Pragmatisches Minimum für v0.1 ist akzeptiert (Warning + Doku). Property-based Fail-Closed als mögliches v0.2-Feature dokumentiert
- **Payload-Duplikation**: Lösung via `TradeIntent.toSigningPayload()` in odin-api bestätigt als beste Option
- **Hex-Validierung**: IMPORTANT für interne Systeme als defensiver Check — umgesetzt

## 6. Gemini Review — 3 Dimensionen (Findings und Umsetzung)

| Dimension | Finding | Severity | Aktion |
|-----------|---------|----------|--------|
| Security | `targetPrice` fehlt im Payload | CRITICAL | ✅ Umgesetzt |
| Security | `runId` nicht in Signatur → Cross-Pipeline-Replay möglich | IMPORTANT | ✅ Umgesetzt |
| Architektur | Payload-Duplikation brain↔execution | IMPORTANT | ✅ `toSigningPayload()` in TradeIntent |
| Robustheit | Silent unsigned-mode Degradation | HINT | Akzeptiert — Logging-Warning bleibt |
| Robustheit | Tamper-Tests für neue Payload-Felder fehlen | HINT | ✅ Umgesetzt |

---

## 7. Testergebnis

```
Tests run: 451, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

- Unit-Tests (odin-brain): 245 Tests (alle grün)
- Unit-Tests (odin-execution): 206 Tests (alle grün)
- Integration-Tests `IntentSignerVerifierIntegrationTest`: 11 Tests (alle grün)
- Integration-Tests `RiskGateHmacIntegrationTest`: 4 Tests (alle grün)

---

## 8. Nicht umgesetzt (bewusste Entscheidungen)

| Feature | Begründung |
|---------|------------|
| Fail-Closed Property `hmac-signing-required` | v0.1-Scope zu groß; Kill-Switch + Warning mitigiert das Risiko ausreichend |
| Key-ID / Version im Intent-Header | Payload-Version (`ODIN_TI|v=1`) ermöglicht spätere Key-Rotation ohne Breaking-Change; Key-ID ist v0.2-Feature |
| Replay-Freshness-Check (Clock-Skew) | `marketTime` + `runId` im Payload begrenzen Replay auf identische Run-Kontexte; volles Freshness-Window ist v0.2-Feature |

---

## 9. Offene Punkte für QS

Keine bekannten Blocker. QS kann auf Basis dieser Dateien geprüft werden:

- Alle neuen Klassen: JavaDoc vollständig ✅
- Alle Tests: grün ✅
- Keine Spring-Annotations in Pipeline-POJOs ✅
- Keine Magic Numbers ✅
- Kein `var` ✅
- Kein `Instant.now()` im Trading-Codepfad ✅
- Kein Secret in Properties/Code ✅ (ausschließlich ENV-Variable)
