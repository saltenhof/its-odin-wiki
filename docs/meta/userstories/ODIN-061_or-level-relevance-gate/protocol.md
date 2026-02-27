# Protokoll: ODIN-061 — OR-Level Relevance Gate

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (21 Tests in OrRelevanceGateTest)
- [x] Integrationstests geschrieben (3 Tests in OrRelevanceGateIntegrationTest)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [ ] Playwright E2E-Test (Out of Scope dieser Agent-Session — separates Ticket noetig)
- [ ] Commit & Push

---

## Design-Entscheidungen

### 1. Status-Speicherung pro SrLevelSource (nicht pro Level-Instanz)

**Entscheidung:** `statusMap` ist ein `EnumMap<SrLevelSource, OrRelevanceStatus>` — gekeyt nach SrLevelSource, nicht nach Level-Instanz.

**Begruendung:** OR-Level haben immer die gleiche SrLevelSource (OR5_HIGH, OR5_LOW, etc.) pro Session. Level-Instanzen koennen sich durch Band-Updates aendern, aber die Source bleibt konstant. `EnumMap` ist maximal speichereffizient — die Groesse ist auf die Anzahl der SrLevelSource-Enum-Konstanten gedeckelt, kein Memory-Leak moeglich.

### 2. MID-Check in addOrUpdateStaticLevels (nicht als nachtraeglicher Filter)

**Entscheidung:** Die MID-Konditionierung greift in der Erzeugungsphase: OR_MID wird gar nicht erst erzeugt, wenn die OR-Range zu breit ist. Wenn ein MID-Level existiert und die Bedingung nicht mehr gilt, wird es via `activeLevels.removeIf(...)` entfernt.

**Begruendung:** Das Konzept verlangt explizit, dass MID nicht nachtraeglich gefiltert wird ("nicht als nachtraeglicher Filter"), da nachtraegliche Filterung Spuren in der TouchRegistry hinterlassen koennte. Die `removeIf`-Zeile ist notwendig fuer den seltenen Fall, dass ein Level existiert und die Bedingung wegfaellt.

### 3. PARKED-Level bleiben in activeLevels

**Entscheidung:** PARKED OR-Level werden aus der Zone-Fusion-Liste herausgefiltert, verbleiben aber in `activeLevels`. Sie werden in `computeOrRelevanceStatus()` gefiltert (gibt `levelsForFusion` zurueck), nicht aus `activeLevels` entfernt.

**Begruendung:** PARKED-Level muessen auf zukuenftige Preisannaeherungen reagieren koennen (PARKED→ACTIVE-Transition via Hysterese). Wenn sie aus `activeLevels` entfernt wuerden, wuerden sie beim naechsten Snapshot neu als initiale Level (ohne Hysterese-Status) behandelt — was das Hysterese-Konzept bricht.

### 4. Initialer Status: activateThreshold-basiert, nicht parkThreshold-basiert

**Entscheidung:** Beim ersten Snapshot fuer eine Source gilt: ACTIVE wenn `distance <= activateThreshold`, sonst PARKED. Der parkThreshold wird fuer den initialen Status nicht verwendet.

**Begruendung:** Diese Entscheidung entspricht dem Konzept ("ACTIVE wenn innerhalb activateThreshold, sonst PARKED"). Konsequenz: Level im Dead-Zone-Bereich beim ersten Snapshot starten als PARKED (nicht als ACTIVE). Das ist bewusst konservativ — besser zu wenig ACTIVE-Level als zu viele.

### 5. PROTECTED → 0 Touches: Behandlung wie ACTIVE (nicht wie NULL)

**Entscheidung:** Wenn ein PROTECTED-Level seine Touches verliert (z.B. durch Registry-Pruning), wird es wie ein ACTIVE-Level behandelt: erst PARKED wenn `distance > parkThreshold`.

**Begruendung:** Die Alternative (wie NULL behandeln: PARKED wenn `distance > activateThreshold`) wuerde ein Level, das kurz zuvor Touches hatte und bewiesen hat, dass es ein relevantes Level ist, zu schnell parken. Die parkThreshold-basierte Fallback-Logik verhindert unnoetiges Flickering.

### 6. isOrLevel() Guard in computeStatus()

**Entscheidung:** `computeStatus()` wirft `IllegalArgumentException` wenn der Source kein OR-Level ist.

**Begruendung:** Fail-fast ist besser als silent corruption des statusMap. Upstream-Code (SrLevelEngine.computeOrRelevanceStatus) filtert via `OrRelevanceGate.isOrLevel()` bereits vor dem Aufruf — dieser Guard ist defensive programming.

---

## Eingearbeitete Review-Findings

### ChatGPT-Findings

| Finding | Prioritaet | Eingearbeitet | Begruendung |
|---------|-----------|---------------|-------------|
| SLF4J Format Strings `{:.4f}` → `{}` | HIGH | Ja | War ein echter Bug — SLF4J ignoriert silently nicht-unterstuetzte Formatstrings. Alle 5 Stellen korrigiert (OrRelevanceGate + SrLevelEngine). |
| PROTECTED→0 touches: treat wie ACTIVE | MEDIUM | Ja | Behandlung via parkThreshold statt activateThreshold verhindert unnoetiges PARKED-Flickering wenn Touches wegfallen. |
| currentPrice NaN/invalid Guard | MEDIUM | Ja | Guard am Anfang von computeStatus() — preserved previous status oder ACTIVE als Default. |
| Non-OR source Guard | LOW | Ja | IllegalArgumentException in computeStatus() — Fail-fast statt silent statusMap-Corruption. |
| ATR=0 high-priced instruments: wide pctCap | INFO | Nein | ODIN traded Intraday-Aktien $10-$200, nicht Berkshire A ($600k). 3% von $200 = $6 Schwelle — akzeptabel. Kein Handlungsbedarf. |
| Dead-zone stability wenn Thresholds aendern | INFO | Nein | Thresholds aendern sich mit currentPrice — theoretisch moeglich. In der Praxis bei Intraday-Trading von RTH bis Close negligible. Kein Handlungsbedarf. |

### Gemini-Findings

| Finding | Dimension | Eingearbeitet | Begruendung |
|---------|-----------|---------------|-------------|
| Fehlender isOrLevel() Guard | Dim 1 (Code) | Ja | Jetzt: IllegalArgumentException wenn non-OR source. |
| Logging korrekt (nach ChatGPT-Fix) | Dim 1 (Code) | n/a | Bereits durch ChatGPT-Sparring gefixed. |
| EnumMap max size gedeckelt | Dim 3 (Praxis) | n/a | Bestaetigt: kein Memory-Leak moeglich. |
| PROTECTED→0 Touches: korrekt gehandhabt | Dim 2/Dim 3 | n/a | Bereits durch ChatGPT-Sparring gefixed und von Gemini bestaetigt. |
| Gap-up/down: korrekt (PARKED initial) | Dim 3 (Praxis) | n/a | Architektonisch korrekt. Kein Handlungsbedarf. |
| Volatility compression: MID suppression | Dim 3 (Praxis) | Nein | Bei sehr komprimierten Sessions koennte MID unnoetig suppressed werden. Akzeptables konservatives Verhalten — besser kein falsches MID als ein storendes MID. |
| High-priced asset: pctCap wird $120 | Dim 3 (Praxis) | Nein | Irrelevant fuer ODIN-Handelsuniverse ($10-$200 Aktien). Eskaliert an Stakeholder: kein Handlungsbedarf. |

---

## ChatGPT-Sparring (2026-02-27)

**Fragestellung:** Grenzfaelle fuer OrRelevanceGate:
1. ATR=0 frueh in der Session
2. Preis genau auf Schwellenwert
3. Alle OR-Level PARKED
4. Nur OR5_HIGH, kein OR5_LOW
5. Preis UNTER OR-Level (negative Distanz)
6. OR-Range genau 0 (HIGH==LOW)
7. EnumMap zukunftssicher?
8. Weitere Hysterese-Edge-Cases

**Wesentliche Erkenntnisse:**
- SLF4J `{:.4f}` Format-Bug behoben (HIGH — war silent data loss in Logs)
- PROTECTED→0-touches: behandle wie ACTIVE fuer sanfteren Uebergang
- currentPrice NaN Guard hinzugefuegt
- isOrLevel() Guard in computeStatus()
- ATR=0 Fallback mit pctCap ist ausreichend fuer ODIN-Handelsuniverse
- Math.abs() fuer distance korrekt — Symmetrie ist gewuenscht

---

## Gemini-Review (2026-02-27)

### Dimension 1: Code-Review

- **EnumMap**: Korrekt und effizient. Max-Size durch Enum-Konstanten gedeckelt.
- **Hysterese-Richtung**: Korrekt. activateThreshold (enger) fuer PARKED→ACTIVE, parkThreshold (weiter) fuer ACTIVE→PARKED.
- **Logging (nach Fix)**: Korrekt mit `{}` Placeholdern.
- **Finding**: Fehlender `isOrLevel()` Guard in `computeStatus()` — behoben.

### Dimension 2: Konzepttreue

- Drei-Status-Lifecycle: vollstaendig implementiert.
- Touch-Praezedenz: Sofortige PROTECTED-Transition bei erstem Touch — korrekt.
- Dead Zone: korrekt implementiert — kein Zustandswechsel zwischen Schwellenwerten.
- MID-Konditionierung: Daily-ATR-Proxy-Formel exakt implementiert.
- kein keepAlways fuer OR-Level: bestaetigt.
- **Fazit: Vollstaendig konzepttreu.**

### Dimension 3: Praxis

- Memory: kein Leak moeglich (EnumMap).
- Gap-up/down Tage: OR startet korrekt als PARKED.
- Volatility compression: MID koennte suppressed werden. Akzeptables konservatives Verhalten.
- High-price instruments: irrelevant fuer ODIN.
- PROTECTED→0 Touches: bereits behoben (Gemini bestaetigt).

---

## Offene Punkte / Eskalationen

1. **Playwright E2E-Test (ODIN-061 DoD 2.8)**: Steht noch aus. Erfordert laufendes Backend + Frontend + IREN 23.02 Backtest. Separater Agent erforderlich. Story ist noch nicht vollstaendig abgeschlossen bis dieser Test vorliegt.

2. **High-price instrument pctCap**: Fuer ODIN-Handelsuniverse ($10-$200) irrelevant. Falls ODIN kuenftig in andere Asset-Klassen expandiert, muss ein absolutes Max-Cap fuer den pctCap-Fallback erwaegt werden.

3. **Volatility compression MID suppression**: Bei sehr engen Handelstagen (z.B. Pre-Holiday) koennte MID faelschlicherweise suppressed werden. Akzeptables Verhalten — kein Handlungsbedarf ohne konkreten Regressionsfall.
