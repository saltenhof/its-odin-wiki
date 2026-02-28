# ODIN-080: Execution Policy Layer

**Modul:** odin-execution, odin-api
**Phase:** 1
**Abhaengigkeiten:** ODIN-067 (Execution Quality Feedback / TCA — liefert die Daten um den Mehrwert dieser Schicht zu messen)
**Geschaetzter Umfang:** L

---

## Kontext

Das OMS erzeugt aktuell Bracket-Orders mit festem Order-Typ (Limit fuer Entry, Stop fuer Absicherung, Limit fuer Targets). Es gibt keine adaptive Selektion des Order-Typs basierend auf Liquiditaetsbedingungen. Bei engen Spreads und hohem Volumen waere ein passives Limit-Order besser (geringere Slippage), bei weiten Spreads und dringenden Exits ein aggressiveres Marketable Limit. Die Execution-Policy-Schicht sitzt zwischen Arbiter/RiskGate und OMS und entscheidet deterministisch ueber Order-Typ, Preis-Offset und maximale Slippage basierend auf aktuellen Marktbedingungen.

## Scope

**In Scope:**

- Neues Interface `ExecutionPolicy` in `de.its.odin.api.port` (oder `de.its.odin.execution.policy`)
- Neue Klasse `DefaultExecutionPolicy` in `de.its.odin.execution.policy` mit deterministischer Order-Typ-Selektion
- Neues Record `ExecutionDirective` mit Order-Typ, Preis-Offset, Max-Slippage
- Integration in `OrderManagementService`: ExecutionPolicy wird vor Order-Submission konsultiert
- 4 Order-Typ-Strategien: PASSIVE_LIMIT, MARKETABLE_LIMIT, AGGRESSIVE_LIMIT, MARKET
- Konfigurationsparameter in `ExecutionProperties` (Namespace `odin.execution.policy.*`)
- Urgency-Bewertung: Regime-Exit und Stop-Update sind dringender als regulaere Entry

**Out of Scope:**

- Slicing/Splitting grosser Orders (erst relevant bei signifikant groesseren Positionsgroessen)
- Vollstaendige Almgren-Chriss-Implementierung (nur vereinfachte Prinzipien)
- Adaptive Policies die sich basierend auf historischer Performance anpassen
- VWAP-/TWAP-Algorithmen
- Repricing-Policy-Aenderungen (bestehende ODIN-020 bleibt unveraendert)

## Akzeptanzkriterien

- [ ] Neues Interface `ExecutionPolicy` mit Methode `ExecutionDirective evaluate(ExecutionContext context)`
- [ ] `ExecutionContext` Record enthaelt: `double spread, double spreadPercent, double volumeRatio, Regime regime, OrderUrgency urgency, double lastPrice, double atr`
- [ ] Neues Enum `OrderUrgency` mit Stufen: `LOW` (regulaerer Entry), `NORMAL` (Target), `HIGH` (Regime-Exit), `CRITICAL` (Stop, Kill-Switch, Forced Close)
- [ ] `ExecutionDirective` Record enthaelt: `OrderType orderType, double priceOffset, double maxSlippage`
- [ ] `DefaultExecutionPolicy` implementiert folgende Logik:
  - `spreadPercent < 0.05%` UND `volumeRatio > 1.5` UND `urgency == LOW` → PASSIVE_LIMIT (priceOffset = 0, am Bid)
  - `spreadPercent < 0.15%` UND `urgency <= NORMAL` → MARKETABLE_LIMIT (priceOffset = 1 Tick ueber Bid)
  - `spreadPercent < 0.30%` ODER `urgency == HIGH` → AGGRESSIVE_LIMIT (priceOffset = halber Spread)
  - Sonst → MARKET (urgency CRITICAL oder Spread zu weit fuer Limits)
- [ ] `OrderManagementService.submitEntry()` konsultiert ExecutionPolicy und verwendet den resultierenden OrderType
- [ ] Bestehende Stop- und Target-Orders bleiben unveraendert (eigene Logik, nicht ueber Policy)
- [ ] Unit-Test: Enger Spread (0.03%) + hohes Volumen + LOW urgency → PASSIVE_LIMIT
- [ ] Unit-Test: Moderater Spread (0.10%) + NORMAL urgency → MARKETABLE_LIMIT
- [ ] Unit-Test: Weiter Spread (0.25%) + HIGH urgency → AGGRESSIVE_LIMIT
- [ ] Unit-Test: CRITICAL urgency → immer MARKET, unabhaengig von Spread
- [ ] Integrationstest: OMS mit DefaultExecutionPolicy erzeugt korrekte Order-Typen fuer verschiedene Marktszenarien
- [ ] Alle Schwellen konfigurierbar via Properties

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `OrderManagementService` | `odin-execution/.../oms/OrderManagementService.java` | Neuer Konstruktor-Parameter `ExecutionPolicy`, Aufruf in `submitEntry()` |
| `ExecutionProperties` | `odin-execution/.../config/ExecutionProperties.java` | Neues `PolicyProperties` Nested Record |
| `PipelineFactory` | `odin-core/.../pipeline/PipelineFactory.java` | Erzeugt `DefaultExecutionPolicy` und reicht es an OMS weiter |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `ExecutionPolicy` | `de.its.odin.execution.policy` | Interface mit `evaluate(ExecutionContext)` Methode |
| `DefaultExecutionPolicy` | `de.its.odin.execution.policy` | Deterministische Implementierung |
| `ExecutionDirective` | `de.its.odin.execution.policy` | Immutable Record: OrderType, priceOffset, maxSlippage |
| `ExecutionContext` | `de.its.odin.execution.policy` | Immutable Record: Marktbedingungen als Input fuer Policy |
| `OrderUrgency` | `de.its.odin.api.model` | Enum: LOW, NORMAL, HIGH, CRITICAL |

### Konfiguration

```properties
# Execution Policy thresholds
odin.execution.policy.enabled=true
odin.execution.policy.passive-limit-max-spread-pct=0.05
odin.execution.policy.passive-limit-min-volume-ratio=1.50
odin.execution.policy.marketable-limit-max-spread-pct=0.15
odin.execution.policy.aggressive-limit-max-spread-pct=0.30
odin.execution.policy.tick-size=0.01
```

### Design-Pattern

- `ExecutionPolicy` ist ein **Strategy Pattern**: Austauschbar ohne OMS-Aenderung. Default: `DefaultExecutionPolicy`. Spaeter moeglich: `SimulationExecutionPolicy` (immer Mid-Price fuer deterministische Backtests).
- Die Policy sitzt als **deterministische Schicht** zwischen Arbiter-Entscheidung und OMS-Submission. Sie veraendert nicht die Trade-Entscheidung (Entry ja/nein), nur die Art der Ausfuehrung.
- `OrderUrgency` wird vom Aufrufer bestimmt (TradingPipeline oder OMS-interner Code) und nicht von der Policy selbst.

## Konzept-Referenzen

- `theme-backlog.md` Thema 19: "Execution-Policy-Schicht (Order-Typ-Selektion)" — IST-Zustand, SOLL-Zustand, Almgren-Chriss-Prinzipien
- `theme-backlog.md` Thema 3: "TCA / Implementation Shortfall" — Messung der Execution-Qualitaet (Abhaengigkeit ODIN-067)
- `docs/backend/architecture/07-oms.md` — OMS-Architektur, Order-Lifecycle, Bracket-Orders
- `docs/backend/architecture/03-broker-integration.md` — IB TWS API, Order-Typen

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen (odin-execution), Abhaengigkeitsregeln
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln (Port-Abstraktion, Records fuer DTOs, ENUM statt String)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution,odin-api,odin-core`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs (ExecutionDirective, ExecutionContext)
- [ ] ENUM statt String (OrderUrgency)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.execution.policy.*`
- [ ] Port-Abstraktion: ExecutionPolicy als Interface

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `DefaultExecutionPolicy` (alle 4 Order-Typ-Strategien + Edge Cases)
- [ ] Unit-Tests fuer `ExecutionContext`-Validierung
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: OMS + DefaultExecutionPolicy End-to-End
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest pro Order-Typ-Strategie

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen, Akzeptanzkriterien, Tests
- [ ] Edge Cases: Was passiert bei Spread=0? Bei NaN-Volumen? Bei fehlender ATR?
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Schwellen-Logik, Off-by-One bei Tick-Offsets)
- [ ] Dimension 2: Konzepttreue-Review (Vergleich mit Thema 19)
- [ ] Dimension 3: Praxis-Review (IB TWS API Order-Typ-Beschraenkungen, Penny-Pilot-Tick-Sizes)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Fuer ODINs Groessenordnung (2-3 Aktien, moderate Positionen) ist der Mehrwert begrenzt** — die Story ist P2 und wird erst relevant wenn TCA (ODIN-067) zeigt, dass Slippage ein signifikantes Problem ist. Die Implementierung sollte daher **lean** sein: keine uebertriebene Abstraktion.
- **BrokerSimulator:** Der `BrokerSimulator` in odin-broker simuliert Fills vereinfacht. Die ExecutionPolicy sollte im Sim-Modus ignoriert oder auf einen simplen Default gesetzt werden (`SimulationExecutionPolicy` die immer MARKET zurueckgibt), um deterministische Backtests nicht zu verkomplizieren.
- **IB TWS API Beschraenkungen:** Nicht alle Order-Typen sind fuer alle Instrumente verfuegbar. Pruefen ob `IbBrokerGateway` die relevanten Order-Typen (LMT, MKT, REL) unterstuetzt. PASSIVE_LIMIT ist ein normales LMT am Bid, MARKETABLE_LIMIT ist ein LMT ueber dem Bid (aber unter dem Ask), AGGRESSIVE_LIMIT ist ein LMT am/nahe Ask.
- **Repricing-Policy (ODIN-020):** Die bestehende Repricing-Policy handhabt den Fall, dass ein Limit-Order nicht gefuellt wird (Repricing in Richtung Market). Die ExecutionPolicy bestimmt den *initialen* Order-Typ — Repricing uebernimmt danach. Diese beiden Policies muessen koharent sein.
- **odin-core Aenderung:** `PipelineFactory` muss `DefaultExecutionPolicy` erzeugen und an `OrderManagementService` uebergeben. Das ist eine kleine Aenderung im Core-Modul.
