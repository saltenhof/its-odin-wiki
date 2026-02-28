# Protokoll: ODIN-006 -- Cycle-Tracking-Modell im API-Modul

## Working State
- [x] Story + Konzepte gelesen
- [x] Bestehenden Code analysiert
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (25 Tests nach R2)
- [x] Integrationstests geschrieben (CycleTrackingIntegrationTest: 7 Tests)
- [x] Kompilierung erfolgreich
- [x] Gemini-Review (3 Dimensionen) -- R1 initial, R2 vollstaendig
- [x] Review-Findings eingearbeitet (Cross-Field-Validation, Null-Checks, cycleNumber>max Validierung)
- [x] ChatGPT-Sparring (R2 durchgefuehrt)
- [x] GlobalRiskManager: onCycleCompleted() implementiert (Bug-Fix R2)
- [x] Commit & Push

## Design-Entscheidungen

### 1. AccountRiskState um totalCyclesCompleted erweitert
Neuer int-Parameter am Ende des Records. GlobalRiskManager trackt und liefert den Wert ueber die neu
implementierte Methode `onCycleCompleted(instrumentId)`. 8 AccountRiskState-Konstruktoraufrufe mussten
um `, 0` ergaenzt werden.

### 2. CycleContext mit strenger Validation
Compact Constructor validiert:
- cycleNumber >= 1
- maxCyclesPerInstrument >= 1
- maxCyclesGlobal >= 1
- cycleNumber <= maxCyclesPerInstrument (neu in R2: verhindert ungueltige Kontexte)
- cycleNumber == 1 → previousCyclePnl == null (neu in R2: Symmetrie-Invariant)
- cycleNumber > 1 → previousCyclePnl != null (aus Gemini-Review R1)
- cycleStartTime != null (neu in R2: Pflicht-Null-Check)
- entryReason != null (neu in R2: Pflicht-Null-Check)

### 3. maxCyclesPerInstrument vs maxCyclesGlobal
Kein Invariant maxCyclesPerInstrument <= maxCyclesGlobal. Bei 3 Instrumenten mit max 3 Cycles pro
Instrument und max 5 global ist das kein Widerspruch -- die Limits operieren auf verschiedenen Ebenen.
Eine Konfig-Warnung bei maxCyclesPerInstrument > maxCyclesGlobal waere sinnvoll, wurde aber als
zu frueh (ODIN-006 ist reines API-Modell, keine Config-Logik) eingestuft.

### 4. Vier CycleEntryReasons
INITIAL (erster Trade), RE_ENTRY_AFTER_TARGET, RE_ENTRY_AFTER_STOP, RE_ENTRY_LLM_APPROVED.
Direkt aus Konzept 03-strategy-logic.md (Abschnitt 8) und 00-overview.md (Abschnitt 9).

### 5. previousCyclePnl als Double (nullable)
Boxed Double statt primitive double, weil der erste Cycle keinen Vorgaenger hat.
Null = kein vorheriger Cycle, nicht "PnL war 0".

### 6. onCycleCompleted(instrumentId) als explizite Methode
GlobalRiskManager erhaelt `onCycleCompleted(String instrumentId)`. Damit wird `totalCyclesCompleted`
korrekt inkrementiert. instrumentId wird geloggt, aber der Zaehler ist global (pro Tag, alle
Pipelines). Per-Instrument-Tracking bleibt in PipelineRiskState (roundTrips), wird aber nicht via
AccountRiskState exportiert -- das ist ein dokumentiertes Open Point.

## Dateien

### Neu erstellt
- `odin-api/src/main/java/de/its/odin/api/model/CycleEntryReason.java` -- 4-Wert-Enum
- `odin-api/src/main/java/de/its/odin/api/dto/CycleContext.java` -- Cycle-Tracking-Record mit erweiterter Validation
- `odin-api/src/test/java/de/its/odin/api/dto/CycleContextTest.java` -- 25 Unit-Tests
- `odin-api/src/test/java/de/its/odin/api/cycle/CycleTrackingIntegrationTest.java` -- 7 Integrationstests

### Geaendert
- `odin-api/src/main/java/de/its/odin/api/dto/AccountRiskState.java` -- totalCyclesCompleted hinzugefuegt
- `odin-core/src/main/java/de/its/odin/core/service/GlobalRiskManager.java` -- onCycleCompleted() + 4 neue Tests
- Alle AccountRiskState-Caller in odin-execution, odin-core Tests

## ChatGPT-Sparring

### Durchfuehrung
Slot: odin-006-review. Dateien uebermittelt: CycleContext.java, CycleEntryReason.java,
AccountRiskState.java, GlobalRiskManager.java.

### Findings und Bewertung

**Finding C1 (CRITICAL): cycleStartTime + entryReason koennen null sein**
- Status: UMGESETZT. Null-Checks fuer beide Felder im Compact Constructor hinzugefuegt.

**Finding C2 (CRITICAL): cycleNumber == 1 sollte previousCyclePnl == null erzwingen**
- Status: UMGESETZT. Symmetrie-Invariant implementiert: Cycle 1 darf keinen previousCyclePnl haben.

**Finding C3 (CRITICAL): Kein Guard gegen cycleNumber > maxCyclesPerInstrument**
- Status: UMGESETZT. Validation: `cycleNumber > maxCyclesPerInstrument` wirft IllegalArgumentException.

**Finding C4 (IMPORTANT): maxCyclesGlobal in per-Pipeline CycleContext semantisch fragwuerdig**
- Status: DOKUMENTIERT als Open Point. maxCyclesGlobal im CycleContext dient als Referenzwert fuer
  pre-trade-Entscheidungen in der Pipeline. Die Durchsetzung des globalen Limits obliegt dem
  GlobalRiskManager (Singleton in odin-core). Der CycleContext spiegelt nur die Konfiguration wider.

**Finding C5 (CRITICAL): contractCurrency vs accountCurrency Benennung**
- Status: DOKUMENTIERT als Open Point. Pre-existing Design, ausserhalb ODIN-006 Scope.

**Finding C6 (CRITICAL): totalCyclesCompleted nicht idempotent**
- Status: DOKUMENTIERT als Open Point. onCycleCompleted ist ein One-way-Signal ohne Idempotenz.
  Robustere Alternative (Map<instrumentId, lastCompletedCycleNumber>) ist ein valider Vorschlag,
  geht aber ueber den ODIN-006 Scope hinaus.

**Finding C7 (CRITICAL): Multi-Instrument Race Condition bei globalem Limit**
- Status: DOKUMENTIERT als Open Point. Wird in odin-core/RiskGate addressiert, nicht in odin-api.

## Gemini-Review (3 Dimensionen)

### Dimension 1: Code-Bugs und technische Korrektheit

**Finding G1 (CRITICAL): onEntryFill ueberschreibt Exposure bei Partial Fills**
- Status: DOKUMENTIERT als Open Point. Pre-existing Bug in GlobalRiskManager, ausserhalb ODIN-006 Scope.

**Finding G2 (CRITICAL): consecutiveLosses global falsch bei Multi-Instrument**
- Status: DOKUMENTIERT als Open Point. Pre-existing Architektur-Problem, ausserhalb ODIN-006 Scope.

**Finding G3 (IMPORTANT): onCycleCompleted nutzt instrumentId nur fuer Logging**
- Status: BEWERTET und AKZEPTIERT. Per-Instrument-Zykluszaehlung ist Out-of-Scope fuer ODIN-006.
  Der globale Zaehler `totalCyclesCompleted` ist das spezifizierte Ziel.

**Finding G4 (MINOR): isCoolingOffActive() hat Side-Effects (State-Mutation in Getter)**
- Status: DOKUMENTIERT als Open Point. Pattern-Anti-Pattern, pre-existing.

### Dimension 2: Konzepttreue

**Finding G5 (IMPORTANT): AccountRiskState exportiert keine Per-Instrument-Daten**
- Status: DOKUMENTIERT als Open Point. Per-Instrument maxCyclesPerInstrument-Enforcement ist
  Aufgabe der Pipeline (CycleContext liegt pro Pipeline vor), nicht des globalen AccountRiskState.

**Finding G6 (IMPORTANT): Round-Trips und Cycles sind redundante Konzepte**
- Status: BEWERTET. Ein Cycle ist konzeptionell eine "entry-to-flat sequence". Round-Trips koennen
  auch innerhalb eines Cycles liegen (wenn Tranchen einzeln geschlossen werden). Die Trennung ist
  semantisch korrekt und intentional.

**Finding G7 (MINOR): Keine Long-Only Validierung in onEntryFill**
- Status: DOKUMENTIERT als Open Point. Validierung sinnvoll, aber pre-existing Gap.

### Dimension 3: Praxis-Gaps

**Finding G8 (IMPORTANT): Race Condition bei globalem Cycle-Limit**
- Wenn maxCyclesGlobal z.B. = 1 ist und 2 Pipelines gleichzeitig starten, sehen beide
  totalCyclesCompleted=0 und erhalten beide Erlaubnis. Das globale Limit wird gebrochen.
- Status: DOKUMENTIERT als Open Point. Loesung erfordert atomare Reservierung im GlobalRiskManager
  (z.B. `reserveCycleSlot(instrumentId)` vor Entry-Erlaubnis). Scope: odin-core/RiskGate-Erweiterung.

**Finding G9 (IMPORTANT): Reset() prueft nicht ob totalExposure == 0**
- Ein Reset waehrend offener Positionen verliert den Kontext zu offenen Positionen.
- Status: DOKUMENTIERT als Open Point. Crash-Recovery ist ein eigenes Architektur-Konzept.

**Finding G10 (MINOR): synchronized auf Methodenebene als Performance-Bottleneck**
- Status: DOKUMENTIERT als Open Point. Erst bauen, dann messen, dann optimieren (Projektprinzip).
  Bei 2-3 parallelen Pipelines ist das kein praktisches Problem.

## Offene Punkte

| ID | Beschreibung | Schwere | Empfohlene Story |
|----|-------------|---------|-----------------|
| OP-01 | Race Condition: maxCyclesGlobal Enforcement bei parallelen Pipeline-Starts (atomare Reservierung im GlobalRiskManager noetig) | HOCH | odin-core / RiskGate |
| OP-02 | contractCurrency vs accountCurrency Benennung in AccountRiskState ist inconsistent | MITTEL | Technische Schuld |
| OP-03 | onCycleCompleted() ist nicht idempotent -- kein Schutz gegen Double-Count bei Retries | MITTEL | GlobalRiskManager-Haertung |
| OP-04 | onEntryFill() ueberschreibt statt addiert Exposure bei Partial Fills (pre-existing Bug) | HOCH | GlobalRiskManager Bug-Fix |
| OP-05 | consecutiveLosses ist global falsch bei Multi-Instrument (Gewinn Pipeline A resettet Verlustserie Pipeline B) | HOCH | GlobalRiskManager Refactoring |
| OP-06 | Per-Instrument Cycle-Zaehlung fehlt im AccountRiskState (RiskGate kann maxCyclesPerInstrument nicht validieren) | MITTEL | AccountRiskState-Erweiterung |
| OP-07 | reset() prueft nicht ob totalExposure == 0 (kann offene Positionen vergessen) | MITTEL | GlobalRiskManager-Haertung |
| OP-08 | maxCyclesPerInstrument > maxCyclesGlobal sollte eine Konfig-Warnung ausloesen | GERING | Config-Validation |
