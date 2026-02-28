# ODIN-076: Correlation Check Multi-Instrument

**Modul:** odin-core, odin-api
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** S

---

## Kontext

ODIN handelt bis zu 2-3 Instrumente parallel. Der `GlobalRiskManager` verwaltet Account-Limits (Gesamtexposure, maximale Verluste), prueft aber nicht, ob die offenen Positionen hochkorreliert sind. Wenn alle 3 Instrumente im gleichen Sektor liegen (z.B. Tech), kann ein sektorweiter Einbruch alle Positionen gleichzeitig stoppen und die Account-Risk-Limits schlagartig reißen. Ein Korrelations-Check vor neuem Entry reduziert dieses Klumpenrisiko.

## Scope

**In Scope:**

- Neue Klasse `CorrelationChecker` in `de.its.odin.core.service` zur Berechnung der Rolling-Korrelation
- Erweiterung von `GlobalRiskManager` um Korrelationspruefung bei neuem Entry
- Korrelationsberechnung auf Basis der letzten N 5-Minuten-Returns (konfigurierbar, Default: 30)
- Blockierung oder SizeModifier-Reduktion bei Korrelation ueber Schwelle
- Konfigurationsparameter in `CoreProperties` (Namespace `odin.core.global-risk.correlation.*`)

**Out of Scope:**

- Sektor-Klassifizierung der Instrumente (rein preisbasierte Korrelation)
- Historische Korrelationsberechnung ueber mehrere Tage
- Dynamische Anpassung der Korrelationsschwelle
- Visualisierung der Korrelationsmatrix im Frontend

## Akzeptanzkriterien

- [ ] Neue Klasse `CorrelationChecker` berechnet Pearson-Korrelation zwischen zwei Return-Serien
- [ ] Returns werden als `(close[t] - close[t-1]) / close[t-1]` auf 5m-Bars berechnet
- [ ] Rolling-Fenster konfigurierbar (Default: 30 Bars = 150 Minuten)
- [ ] `GlobalRiskManager.checkCorrelation(String newInstrumentId, List<Double> newInstrumentReturns)` prueft gegen alle offenen Positionen
- [ ] Bei Korrelation > 0.7 (konfigurierbar): `AccountRiskState` enthaelt neues Flag `correlationRiskElevated=true` und `correlationSizeMultiplier=0.5`
- [ ] Bei Korrelation > 0.9: Entry wird blockiert (SizeMultiplier = 0.0)
- [ ] Unit-Test: Perfekt korrelierte Returns (r=1.0) → Entry blockiert
- [ ] Unit-Test: Unkorrelierte Returns (r≈0.0) → kein Flag, SizeMultiplier = 1.0
- [ ] Unit-Test: Korrelation = 0.75 → SizeMultiplier = 0.5
- [ ] Unit-Test: Nur 1 offene Position → kein Korrelationscheck noetig, SizeMultiplier = 1.0
- [ ] Unit-Test: Weniger als 10 Returns verfuegbar → Korrelation nicht berechenbar, Default SizeMultiplier = 1.0

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `GlobalRiskManager` | `odin-core/.../service/GlobalRiskManager.java` | Neue Methode `checkCorrelation()`, Speicherung von Return-Serien pro Pipeline |
| `AccountRiskState` | `odin-api/.../dto/AccountRiskState.java` | Neue Felder `correlationRiskElevated`, `correlationSizeMultiplier` |
| `CoreProperties` | `odin-core/.../config/CoreProperties.java` | Neues `CorrelationProperties` in `GlobalRiskProperties` |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `CorrelationChecker` | `de.its.odin.core.service` | Pearson-Korrelationsberechnung mit Rolling-Fenster |

### Algorithmus: Pearson-Korrelation

Standard-Pearson-Formel auf den letzten N Returns:
```
r = Σ((xi - x̄)(yi - ȳ)) / sqrt(Σ(xi - x̄)² * Σ(yi - ȳ)²)
```
Keine externen Libraries noetig — einfache Implementierung mit Schleifen.

### Konfiguration

```properties
odin.core.global-risk.correlation.enabled=true
odin.core.global-risk.correlation.window-bars=30
odin.core.global-risk.correlation.reduce-threshold=0.70
odin.core.global-risk.correlation.block-threshold=0.90
odin.core.global-risk.correlation.reduce-size-multiplier=0.50
odin.core.global-risk.correlation.min-data-points=10
```

### Datenfluss

1. `TradingPipeline` sendet bei jedem 5m-Bar-Close den Return an `GlobalRiskManager.recordReturn(instrumentId, return)`
2. Bei neuem Entry-Intent: `GlobalRiskManager.getAccountRiskState()` enthaelt die Korrelationsinformation
3. `RiskGate` liest `correlationSizeMultiplier` aus `AccountRiskState` und multipliziert auf PositionSize

## Konzept-Referenzen

- `theme-backlog.md` Thema 14: "Korrelations-Check fuer Multi-Instrument-Risiko" — IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/00-system-overview.md` — Multi-Instrument-Architektur, GlobalRiskManager
- `docs/backend/architecture/07-oms.md` — Position Sizing, RiskGate

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — odin-core Singleton-Services, Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints
- `CLAUDE.md` — Coding-Regeln (keine Magic Numbers, Records fuer DTOs)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core,odin-api`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.core.global-risk.correlation.*`

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `CorrelationChecker` (perfekte, keine, partielle Korrelation, Edge Cases)
- [ ] Unit-Tests fuer erweiterten `GlobalRiskManager`
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: GlobalRiskManager mit CorrelationChecker und 2+ instrumenten
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen, Akzeptanzkriterien, Tests
- [ ] Edge Cases abgefragt (NaN-Returns, identische Instrumente, negative Korrelation)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Numerik-Praezision, Division-by-Zero)
- [ ] Dimension 2: Konzepttreue-Review
- [ ] Dimension 3: Praxis-Review (Spurious Correlation bei kurzen Fenstern)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Pearson-Korrelation ist instabil bei kurzen Fenstern** (<10 Datenpunkte). Deshalb der `min-data-points` Parameter: bei weniger als 10 Returns wird keine Korrelation berechnet und der Default-Multiplikator 1.0 zurueckgegeben. Das ist konservativ (lieber handeln als unberechtigt blockieren).
- **Return-Speicherung:** Die 5m-Returns muessen im `GlobalRiskManager` pro Instrument in einem ringbasierten Puffer (oder `LinkedList` mit size-cap) gespeichert werden. Max-Groesse = `window-bars` (Default 30).
- **Thread-Safety:** `GlobalRiskManager` ist ein Singleton, aber die Pipelines sind single-threaded pro Pipeline. Die Methoden `recordReturn()` und `getAccountRiskState()` werden aus verschiedenen Pipeline-Threads aufgerufen → Synchronisation noetig (z.B. `synchronized` auf der Return-Map oder `ConcurrentHashMap` + `CopyOnWriteArrayList`).
- **Negative Korrelation (z.B. Long-Short-Hedge):** ODIN ist Long-Only, daher ist negative Korrelation theoretisch diversifizierend. Die aktuelle Logik prueft nur `|correlation| > threshold` — der Implementierer soll entscheiden ob `abs()` oder nur positive Korrelation relevant ist. Empfehlung: nur positive Korrelation, da negative Korrelation bei Long-Only wuenschenswert ist.
