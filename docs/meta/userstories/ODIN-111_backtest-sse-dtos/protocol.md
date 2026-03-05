# Protokoll: ODIN-111 — Backtest-SSE DTOs und EventType-Erweiterung

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases (Runde 2: 2026-03-05)
- [x] Integrationstests geschrieben (n.a. — story-spezifische DoD-Ausnahme: reine DTOs, Jackson-Serialisierungstests ausreichend)
- [x] Gemini-Review Dimension 1 (Code) (Runde 2: 2026-03-05)
- [x] Gemini-Review Dimension 2 (Konzepttreue) (Runde 2: 2026-03-05)
- [x] Gemini-Review Dimension 3 (Praxis) (Runde 2: 2026-03-05)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

---

## Design-Entscheidungen

### BigDecimal fuer monetaere Werte
Alle finanziellen Felder (currentPrice, dayPnl, overallPnl, cumulativePnl, maxDrawdown, stopLevel) verwenden `BigDecimal` gemaess Story-Notiz und CLAUDE.md-Vorgabe. Dies vermeidet Floating-Point-Praezisionsverluste bei Geldbetraegen.

### double fuer Raten und Scores
`winRate`, `score` und `rrRatio` verwenden `double`, da sie Prozentwerte/Verhaeltnisse im Bereich 0.0-1.0 darstellen. Per Story-Notiz ist BigDecimal hier nicht erforderlich.

### LocalDate vs Instant
- `tradingDate` als `LocalDate`: Repraesentiert den abstrakten Handelstag ohne Zeitzone
- `marketTime` als `Instant`: Repraesentiert den exakten Marktzeitpunkt fuer zeitkritische Events (QuantScore, GateResult)

### Factory-Methoden (`of()`)
Jeder Record bietet eine statische `of()`-Methode, die den `type`-Discriminator automatisch aus dem `SseEventType`-Enum setzt. Dies verhindert Fehler bei der manuellen Discriminator-Vergabe. Der Canonical Constructor bleibt fuer Jackson-Deserialisierung offen.

### Keine Validierung in DTOs
Bewusste Entscheidung: DTOs sind reine Datentraeger ohne Validierungslogik. Validierung gehoert in die Publisher-Schicht (Folgestory). Dies folgt dem CLAUDE.md-Grundsatz "keine Abstraktionsschichten ohne Mandat".

### Defensive List-Kopien (Runde 2 — Review-Finding)
Nach ChatGPT- und Gemini-Review: `List.copyOf()` in allen `of()`-Factory-Methoden eingebaut.
- Verhindert stille Mutation nach Record-Erstellung (Record-Immutabilitaet war bisher nur flach)
- Null-Normalisierung: `null`-Listen werden auf `List.of()` normalisiert, um garantiert `[]` in JSON zu erzeugen
- Verhindert Frontend-`TypeError` wenn `event.instruments.map(...)` aufgerufen wird und die Liste `null` waere

---

## Erstellte/Geaenderte Dateien

### Neue Dateien
| Datei | Beschreibung |
|---|---|
| `odin-app/.../dto/sse/BacktestBarProgressEvent.java` | DTO Record fuer Bar-Level-Fortschritt (10 Felder) |
| `odin-app/.../dto/sse/BacktestDaySummaryEvent.java` | DTO Record fuer Tagesabschluss-Zusammenfassung (10 Felder) |
| `odin-app/.../dto/sse/BacktestQuantScoreEvent.java` | DTO Record fuer Quant-Score-Bewertung (9 Felder) |
| `odin-app/.../dto/sse/BacktestGateResultEvent.java` | DTO Record fuer Gate-Evaluierungsergebnis (10 Felder) |

### Geaenderte Dateien
| Datei | Aenderung |
|---|---|
| `odin-app/.../sse/SseEventType.java` | 4 neue Enum-Werte mit JavaDoc |
| `odin-app/.../dto/sse/SseEventDtoSerializationTest.java` | 30 Tests gesamt (15 bestehend + 8 Runde-1 + 7 Runde-2) |
| `odin-app/.../dto/sse/BacktestDaySummaryEvent.java` | List.copyOf() + null-Normalisierung (Runde 2) |
| `odin-app/.../dto/sse/BacktestQuantScoreEvent.java` | List.copyOf() + null-Normalisierung (Runde 2) |
| `odin-app/.../dto/sse/BacktestGateResultEvent.java` | List.copyOf() + null-Normalisierung (Runde 2) |

---

## Test-Ergebnisse

### Unit-Tests (Surefire) — Stand Runde 2
- **Testklasse:** `SseEventDtoSerializationTest`
- **Anzahl Tests:** 30 (15 bestehend + 8 Runde 1 + 7 Runde 2)
- **Ergebnis:** Alle 30 Tests PASS
- **Gesamt odin-app:** 394 Tests PASS, 0 Failures

### Neue Tests in Runde 2
| Test | Motivation |
|---|---|
| `backtestQuantScoreEventShouldSerializeMarketTimeAsIso8601String` | Regression-Guard: Jackson-Konfiguration explizit verifizieren |
| `backtestGateResultEventShouldSerializeMarketTimeAsIso8601String` | Regression-Guard: Jackson-Konfiguration explizit verifizieren |
| `backtestDaySummaryEventShouldNormalizeNullInstrumentsToEmptyArray` | Null-Normalisierung: garantiert `[]` im JSON |
| `backtestQuantScoreEventShouldNormalizeNullListsToEmptyArrays` | Null-Normalisierung: reasonCodes und vetos |
| `backtestGateResultEventShouldNormalizeNullRejectReasonsToEmptyArray` | Null-Normalisierung: rejectReasons |
| `backtestDaySummaryEventListShouldBeImmutable` | Defensive-Copy: Mutation nach Erstellung verhindert |
| `backtestQuantScoreEventListsShouldBeImmutable` | Defensive-Copy: Mutation nach Erstellung verhindert |

### Build
- `mvn clean install -DskipTests` — BUILD SUCCESS (26.199 s)
- `mvn test -pl odin-app` — BUILD SUCCESS, 394/394 Tests PASS

---

## Offene Punkte

Gemini Dimension 3 hat systemische Punkte identifiziert, die AUSSERHALB des Scopes dieser Story liegen:

1. **backtestId / runId** (CRITICAL laut Gemini): Wenn ein Backtest abgebrochen und ein neuer gestartet wird, koennen trailing Events aus dem alten Run den UI-State korrumpieren. Loessung erfordert Publisher-Schicht (Folgestory). Im aktuellen DTO-Scope nicht loesbar ohne Scope-Erweiterung.

2. **eventId fuer React Keys** (MAJOR laut Gemini): Ohne eindeutige Event-ID muessen Frontend-Entwickler synthetische Keys bilden. Kollisionsrisiko bei gleicher `marketTime + instrumentId`. Ebenfalls Publisher-Schicht-Thema.

3. **NaN/Infinity fuer double-Felder**: Jackson wirft standardmaessig bei NaN/Infinity. Upstream-Komponenten (Publisher, SimulationRunner) MUESSEN sicherstellen, dass `winRate`, `score`, `rrRatio` keine NaN/Infinity-Werte liefern. Alternativ: Jackson-Konfiguration mit `ALLOW_NON_NUMERIC_NUMBERS`. Dies ist eine Architekturentscheidung fuer den Stakeholder.

---

## ChatGPT-Sparring

**Datum:** 2026-03-05
**Fokus:** Serialization Edge Cases, Null-Safety, Missing Test Scenarios

### Identifizierte Risiken (ChatGPT)

| Finding | Schwere | Status |
|---------|---------|--------|
| Flache Immutabilitaet: `List<String>`-Parameter koennen nach Erstellung mutiert werden | MAJOR | **Eingearbeitet** — `List.copyOf()` in allen `of()`-Methoden |
| Null-Listen serialisieren als `null` statt `[]` | MAJOR | **Eingearbeitet** — Null-Normalisierung in `of()` |
| `marketTime`-Assertion pruefte nur "isNotNull", nicht den exakten Format-String | MINOR | **Eingearbeitet** — 2 neue Regression-Guard-Tests |
| NaN/Infinity bei `double`-Feldern (winRate, score, rrRatio) | MINOR | **Dokumentiert** als Offener Punkt — Validierung in Publisher-Schicht |
| Canonical Constructor erlaubt Umgehung des `of()`-Discriminators | INFO | Akzeptiert — Jackson-Deserialisierung benoetigt Canonical Constructor |
| `result`/`intent`/`setupType`/`gateName` als freie Strings — Tippfehlerrisiko | INFO | Akzeptiert — Enum-Umstieg waere Scope-Erweiterung, nicht pure DTO-Story |

### Umgesetzte Vorschlaege
- `List.copyOf()` + Null-Normalisierung in `BacktestDaySummaryEvent.of()`, `BacktestQuantScoreEvent.of()`, `BacktestGateResultEvent.of()`
- 5 neue Null/Immutabilitaets-Tests
- 2 neue Regression-Guard-Tests fuer ISO-8601-Zeitformatierung

---

## Gemini-Review

**Datum:** 2026-03-05
**Slot Owner:** ODIN-111-rework

### Dimension 1: Code-Bugs und Qualitaet

**Ergebnis:** Keine CRITICAL-Findings. Ein MAJOR-Finding, zwei MINOR-Findings.

| Finding | Schwere | Status |
|---------|---------|--------|
| List Mutability: `of()`-Methoden uebergeben Listreferenzen ohne defensive Kopie | MAJOR | **Eingearbeitet** — `List.copyOf()` in allen betroffenen `of()`-Methoden |
| Keine expliziten Null-Annotations (`@NonNull`/`@Nullable`) | MINOR | Akzeptiert — keine static-analysis-Bibliothek im Scope des Projekts konfiguriert |
| `BigDecimal`-Serialisierung koennte scientific notation produzieren bei extremen Werten | MINOR | Dokumentiert — `WRITE_BIGDECIMAL_AS_PLAIN` als Option fuer ObjectMapper-Konfiguration |

**Positive Bewertung:** "Strong Pattern Enforcement", "Excellent naming guardrails", "Perfect documentation bridges code and architecture"

### Dimension 2: Konzepttreue

**Ergebnis:** "Exceptionally aligned" — keine Abweichungen.

- Alle Feldnamen stimmen exakt mit Concept 11 §3.3.9–§3.3.12 ueberein
- Alle Feldtypen korrekt (BigDecimal/double/LocalDate/Instant/List<String>)
- Alle 4 Discriminator-Strings korrekt
- Keine fehlenden Felder, keine Extra-Felder
- Feldreihenfolge im JSON entspricht dem Konzept

### Dimension 3: Praxis-Gaps

**Ergebnis:** 2 CRITICAL, 2 MAJOR, 2 MINOR — alle ausserhalb des aktuellen Story-Scopes (reine DTOs).

| Finding | Schwere | Sofortmassnahme | Status |
|---------|---------|-----------------|--------|
| Kein `backtestId` in DTOs — trailing Events koennen UI-State korrumpieren | CRITICAL | Publisher-Schicht (Folgestory) | Als Offener Punkt dokumentiert |
| Multi-Instrument-Progress-Ambiguitaet in `BacktestBarProgressEvent` | MAJOR | Konzept-Klaerung | Als Offener Punkt dokumentiert |
| Frontend Main-Thread-Freezing bei 84k Events (Throttling fehlt) | CRITICAL | Throttling in Publisher-Schicht | Ausserhalb Scope — bereits in SseProperties konfigurierbar |
| Frontend React-Key-Kollision (kein `eventId`) | MAJOR | Publisher-Schicht | Als Offener Punkt dokumentiert |
| NaN/Infinity bei `double`-Feldern | MINOR | Upstream-Validierung oder Jackson-Konfiguration | Als Offener Punkt dokumentiert |
| Kein Versionsfeld in DTOs | MINOR | Zukuenftige Erweiterung | Akzeptiert als bekannte Limitierung |

**Bewertung:** Die drei Sofortmassnahmen-Findings betreffen allesamt die Publisher-Schicht, nicht die DTO-Schicht. Die DTOs sind fuer den definierten Scope vollstaendig korrekt implementiert.
