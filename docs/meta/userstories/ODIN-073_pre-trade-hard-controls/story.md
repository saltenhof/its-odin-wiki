# ODIN-073: Pre-Trade Hard Controls

## Modul

odin-execution, odin-api

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

M

## Kontext

Systemische Pre-Trade-Kontrollen schuetzen vor Fat-Finger-Fehlern, Runaway-Algos und Schleifen — unabhaengig vom Strategy-Code. ODINs `RiskGate` prueft Position Sizing, R/R und globale Account-Limits, aber es fehlen explizite Fat-Finger-Limits, Preis-Collars und Order-Rate-Limits. Laut FIA Best Practices ist der Strategy-Code der haeufigste Fehlerort — die Pre-Trade-Kontrollen muessen daher unabhaengig vom Strategy-Code wirken. Diese Story fuegt drei Hard-Controls hinzu: Max Order Size, dynamische Price Collars und Order Rate Limits.

## Scope

### In Scope

- **Max Order Size (Fat-Finger-Schutz):** Harter Reject wenn Order-Groesse > konfigurierbares Maximum (absolut in Shares und relativ in % des ADV)
- **Dynamische Price Collars:** Reject wenn Order-Preis ausserhalb VWAP +/- N*ATR (konfigurierbar, Default: 5*ATR)
- **Order Rate Limit pro Pipeline:** Max N Orders pro Minute pro Pipeline (Default: 5), schuetzt vor Schleifen/duplizierten Orders
- **Max Intraday Position:** Maximale Gesamtposition pro Instrument (absolut in Shares)
- Integration in bestehenden `RiskGate` als zusaetzliche Checks VOR der bestehenden Logik

### Out of Scope

- Market-wide Controls (z.B. globaler Order-Rate-Limit ueber alle Pipelines — bereits durch GlobalRiskManager abgedeckt)
- Post-Trade Controls (z.B. Kill-Switch bei P&L-Schwelle — bereits vorhanden)
- Adaptive Limits basierend auf Marktbedingungen (feste Konfiguration, keine Dynamik)
- UI fuer Control-Konfiguration

## Akzeptanzkriterien

- [ ] `PreTradeControlGate` prueft Max Order Size: Reject wenn `orderShares > maxOrderShares` ODER `orderShares > maxOrderAdvPercent * avgDailyVolume`
- [ ] `PreTradeControlGate` prueft Price Collar: Reject wenn `abs(orderPrice - vwap) > priceCollarAtrMultiplier * atr`
- [ ] `PreTradeControlGate` prueft Order Rate: Reject wenn `ordersInLastMinute >= maxOrdersPerMinute` pro Pipeline
- [ ] `PreTradeControlGate` prueft Max Position: Reject wenn `currentPosition + orderShares > maxIntradayPosition`
- [ ] Jeder Reject wird mit spezifischem `RejectReason` im EventLog geloggt
- [ ] Checks laufen VOR der bestehenden RiskGate-Logik (PositionSizer, R/R-Check) — wenn Pre-Trade-Control rejectet, wird nichts weiter evaluiert
- [ ] Alle Schwellen sind konfigurierbar via `ExecutionProperties`
- [ ] Unit-Test verifiziert: Order mit 10.000 Shares bei maxOrderShares=5.000 wird rejected mit Reason `MAX_ORDER_SIZE_EXCEEDED`
- [ ] Unit-Test verifiziert: Order mit Preis bei VWAP + 6*ATR bei priceCollarAtrMultiplier=5 wird rejected mit Reason `PRICE_COLLAR_BREACH`
- [ ] Unit-Test verifiziert: 6. Order in 60 Sekunden bei maxOrdersPerMinute=5 wird rejected mit Reason `ORDER_RATE_EXCEEDED`
- [ ] Unit-Test verifiziert: Alle Controls PASS fuer normale Order -> weiter zu RiskGate

## Technische Details

### Neue Klassen

| Klasse | Modul | Package | Beschreibung |
|--------|-------|---------|-------------|
| `PreTradeControlGate` | odin-execution | `de.its.odin.execution.risk` | Orchestriert alle Pre-Trade-Checks. Pro-Pipeline-POJO. Haelt State fuer Order-Rate-Tracking |
| `OrderRateTracker` | odin-execution | `de.its.odin.execution.risk` | Sliding-Window-Counter: zaehlt Orders pro Pipeline in den letzten N Sekunden. Thread-safe (ConcurrentLinkedDeque) |
| `PreTradeRejectReason` | odin-api | `de.its.odin.api.model` | Enum oder Erweiterung von `RejectReason`: MAX_ORDER_SIZE_EXCEEDED, PRICE_COLLAR_BREACH, ORDER_RATE_EXCEEDED, MAX_POSITION_EXCEEDED |

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `RiskGate` (`de.its.odin.execution.risk.RiskGate`) | Neuer erster Check-Schritt: `PreTradeControlGate.evaluate()` aufrufen. Bei Reject: sofort zurueckkehren, keine weiteren Checks |
| `RejectReason` (`de.its.odin.api.model.RejectReason`) | Neue Enum-Werte: `MAX_ORDER_SIZE_EXCEEDED`, `PRICE_COLLAR_BREACH`, `ORDER_RATE_EXCEEDED`, `MAX_POSITION_EXCEEDED` |
| `ExecutionProperties` (`de.its.odin.execution.config.ExecutionProperties`) | Neue Nested-Record `PreTradeControlProperties`: maxOrderShares (int), maxOrderAdvPercent (double), priceCollarAtrMultiplier (double), maxOrdersPerMinute (int), maxIntradayPosition (int) |
| `PipelineFactory` (`de.its.odin.core.pipeline.PipelineFactory`) | PreTradeControlGate erstellen und an RiskGate uebergeben |

### Check-Reihenfolge im RiskGate (erweitert)

```
1. HMAC-Verifikation (bestehend)
2. PRE-TRADE CONTROLS (NEU):
   a. Max Order Size
   b. Price Collar
   c. Order Rate Limit
   d. Max Intraday Position
3. Pipeline-Context-Invariant (bestehend)
4. Position Sizing (bestehend)
5. R/R-Check (bestehend)
6. Account-Level-Limits (bestehend)
```

### Order-Rate-Tracking

```java
// Sliding Window: Deque von Timestamps
// Bei jedem Check: abgelaufene Eintraege entfernen, aktuellen zaehlen
Deque<Instant> orderTimestamps = new ConcurrentLinkedDeque<>();

boolean isWithinLimit(MarketClock clock) {
    Instant cutoff = clock.now().minus(Duration.ofMinutes(1));
    orderTimestamps.removeIf(ts -> ts.isBefore(cutoff));
    return orderTimestamps.size() < maxOrdersPerMinute;
}
```

### Konfiguration

```properties
odin.execution.pre-trade.max-order-shares=5000
odin.execution.pre-trade.max-order-adv-percent=1.0
odin.execution.pre-trade.price-collar-atr-multiplier=5.0
odin.execution.pre-trade.max-orders-per-minute=5
odin.execution.pre-trade.max-intraday-position=10000
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 11: Pre-Trade Hard Controls, Abschnitte IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/07-oms.md` — OMS-Architektur, Order-Flow, RiskGate-Integration
- `docs/backend/architecture/00-system-overview.md` — Kill-Switch, Safety-Mechanismen

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen (odin-execution Packages)
- `docs/backend/guardrails/development-process.md` — Phasen, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, ENUM statt String, MarketClock verwenden (kein Instant.now())

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution,odin-api`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer RejectReasons
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.execution.pre-trade.*`
- [ ] MarketClock fuer Zeitvergleiche (kein Instant.now())

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `PreTradeControlGate` (jeder Check einzeln und in Kombination)
- [ ] Unit-Tests fuer `OrderRateTracker` (Window-Sliding, Ablauf, Thread-Safety)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer MarketClock, EventLog

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: RiskGate mit PreTradeControlGate End-to-End
- [ ] Integrationstest: Mehrere Orders in schneller Folge -> Rate Limit greift
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen und Akzeptanzkriterien
- [ ] Edge Cases diskutiert (Partial Fills zaehlen als 1 Order?, Pre-Market vs. RTH Limits)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Thread-Safety, Time-Window-Logik, Null-Safety)
- [ ] Dimension 2: Konzepttreue-Review (FIA Best Practices, Control-Vollstaendigkeit)
- [ ] Dimension 3: Praxis-Review (realistishe Schwellenwerte, Markt-Szenarien)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **MarketClock verwenden, NICHT Instant.now():** Fuer Order-Rate-Tracking und alle Zeitvergleiche MarketClock verwenden. Im Backtest tickt die Clock schneller als Realzeit — `Instant.now()` wuerde falsche Ergebnisse liefern.
- **VWAP fuer Price Collar:** Der aktuelle VWAP muss dem PreTradeControlGate zur Verfuegung stehen. Er kommt aus dem `MarketSnapshot`. Ebenso der aktuelle ATR aus `IndicatorResult`.
- **ADV (Average Daily Volume):** Wird aus historischen Daten berechnet oder muss als Konfiguration/Profiling-Input vorliegen. Fuer V1: aus `InstrumentProfile` nehmen (wenn verfuegbar) oder als Konfigurationswert. KEIN Blocker wenn ADV nicht verfuegbar — dann nur absoluten Max-Order-Size-Check verwenden.
- **Partial Fills:** Ein Order, der in mehreren Fills ausgefuehrt wird, zaehlt als EINE Order im Rate-Limit (nicht pro Fill).
- **Check-Reihenfolge:** Pre-Trade-Controls VOR Position-Sizing, weil Position-Sizing aufwaendiger ist (Division, Account-State-Abfrage). Billige Checks zuerst.
- **Alle Rejects loggen:** Jeder Pre-Trade-Reject muss als Event im EventLog erscheinen, damit Forensik moeglich ist. Das ist bereits Muster im bestehenden RiskGate.
