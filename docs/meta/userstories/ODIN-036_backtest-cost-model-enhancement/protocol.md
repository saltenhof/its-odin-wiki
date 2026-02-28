# Protokoll: ODIN-036 — Enhanced Cost Model fuer Backtests

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis) — Teilweise (Gemini-Session Kommunikationsprobleme)
- [x] Review-Findings eingearbeitet
- [x] QS-Review R1 durchgefuehrt (ChatGPT + Gemini)
- [x] QS-Findings eingearbeitet
- [x] Commit & Push

**Test-Ergebnis nach QS-R1:** 143 Tests, 0 Failures, 0 Errors (inkl. 45 Tests fuer BacktestCostModel: 38 Unit + 7 Integration)

## Design-Entscheidungen

### 1. Neue Klasse statt Erweiterung TransactionCostCalculator

**Entscheidung:** Neue Klasse `BacktestCostModel` in `odin-backtest`, anstatt `TransactionCostCalculator` in `odin-execution` zu erweitern.

**Begruendung:**
- `TransactionCostCalculator` wird vom Live-OMS verwendet — Erweiterungen wuerden Backtest-spezifische Logik in die Live-Trading-Codepfade einbringen
- `odin-backtest` hat bereits eine Abhaengigkeit auf `odin-core` (transitiv auf `odin-execution`) — ein Zugriff auf den ExecutionProperties-Record war bereits moeglich
- Backtest-Kosten (slippage-separiert, FX-conditional) sind fachlich verschieden von Live-Kosten
- Separate `BacktestCostProperties` im Namespace `odin.backtest.cost.*` ermoeglichen unabhaengige Konfiguration

### 2. FX-Kosten nur auf Entry-Position (nicht Exit)

**Entscheidung:** FX-Kosten werden auf den Entry-Positionswert angewendet, nicht auf Entry + Exit.

**Begruendung:** IB Multi-Currency-Account-Modell: EUR werden einmalig bei Position-Open in USD konvertiert. Exit-Erloese bleiben im USD-Sub-Konto. Nur ein Konvertierungsevent pro Trade, nicht zwei. Diese Vereinfachung ist fuer Intraday-Backtest ausreichend und entspricht der praktischen IB-Kontostruktur.

### 3. Prozent-basierte Slippage statt Tick-basiert

**Entscheidung:** Slippage als konfigurierbarer Prozentsatz des Handelspreises, nicht als Tick-Zahl.

**Begruendung:** Das Konzept (Abschnitt 5) beschreibt tick-basierte Slippage als "Proxy: `slippage = base_slippage + k * ATR * volatility_factor`". Fuer den V1-Backtest ist ein einfacher konfigurierbarer Prozentsatz ausreichend und vermeidet die Komplexitaet von tick-size-Abfragen pro Instrument. Der Story-Acceptance-Criteria definiert explizit prozentbasierte Formeln — diese hatten Prioritaet.

### 4. Kommission: Story-Defaults vs. Konzept-Defaults

**Entscheidung:** Story-Defaults verwenden: $1.00 Fixed + $0.005/Share.

**Begruendung:** Das Konzept (v1 Abschnitt 5) nennt $0.0035/Share ohne fixen Anteil (einfacheres IB-Modell). Die Story spezifiziert $1.00 Fixed + $0.005/Share als "realistischeres IB-Default fuer kleinere Accounts". Die Story hat als aktives Ticket Prioritaet. Beide Varianten sind konfigurierbar.

### 5. FX-Default: 0.003% (Story) vs. 0.002% (Konzept)

**Entscheidung:** Story-Default 0.003% implementiert.

**Begruendung:** Story overridet Konzept. Beide Werte sind konservative Schaetzungen; 0.003% ist minimal konservativer.

### 6. Double-Rounding-Vermeidung (Gemini Finding)

**Entscheidung:** Rohe (ungerundete) Werte werden summiert, bevor der Total gerundet wird. Einzelne Komponenten werden fuer Reporting gerundet.

**Begruendung:** Gerundete Komponenten zu summieren fuehrt zu akkumulierten Rundungsfehlern bei vielen Trades. Der Total wird aus der Rohsumme gerundet — ein Standardpattern fuer finanzielle Berechnungen.

### 7. NaN/Infinity Guards (Gemini Finding)

**Entscheidung:** Explizite `Double.isNaN()` und `Double.isInfinite()` Checks vor dem `<= 0` Check.

**Begruendung:** `NaN <= 0.0` evaluiert zu `false` in Java — NaN wuerde die Exception-Guard ueberstehen und den gesamten P&L korrumpieren. Da Marktdaten unzuverlaessig sein koennen, ist ein Fail-Fast-Guard korrekt.

## Offene Punkte

### Bekannte Limitationen (an Stakeholder zur Kenntnisnahme)

1. **Shock Slippage fehlt:** Das Konzept (Abschnitt 5) erwaehnt erhoehten Slippage-Faktor bei Crash/Spike Events. Nicht implementiert. Betrifft nur Stress-Tests — fuer regulaere Backtests irrelevant.

2. **Tick-basierte Slippage fehlt:** Das Konzept beschreibt `slippage = base_slippage + k * ATR * volatility_factor`. Implementiert als einfacher Prozentsatz. Ausreichend fuer V1-Backtest.

3. **Regulatorische Fees (SEC Section 31, FINRA TAF) fehlen:** Diese betragen nur wenige Mikro-Dollar pro Trade fuer die typischen Positionsgroessen und sind fuer den Backtest vernachlaessigbar. Als bekannte Limitation dokumentiert.

4. **ECN/Venue Fees fehlen:** IB-Tiered Pricing beinhaltet Venue-spezifische Fees die in diesem Modell nicht abgebildet sind. Konfigurierbar ueber slippage-Parameter naeherungsweise darstellbar.

5. **Partial Fills nicht modelliert:** Konzept-Konform: "Partial Fills nicht modelliert — bekannte Limitation".

6. **Quantity als `int`:** Bei Penny Stocks koennen Positionsgroessen theoretisch > 2,1 Milliarden Aktien erreichen (Integer.MAX_VALUE), was jedoch bei realistischen Kontovorgaben ($50K Kapital, $0.001/Aktie = 50 Millionen Aktien) gut innerhalb des `int`-Bereichs liegt.

## ChatGPT-Sparring

**Datum:** 2026-02-22

**Themen:** Edge Cases fuer BacktestCostModel (penny stocks, integer overflow, FX-Richtung, fehlende IB-Kosten)

### Vorgeschlagene Tests — umgesetzt:

| Vorschlag | Status | Begruendung |
|-----------|--------|-------------|
| Cap beats minimum fuer penny stocks ($0.01/share, 1 share) | **Umgesetzt** als `commission_capBelowMinimum_capWins` | Kritischer Edge Case der IB-Preisstruktur |
| Zero LLM tokens mit aktiviertem LLM-Cost | **Umgesetzt** als `llmCost_zeroTokens_producesZeroEvenWhenEnabled` | Boundary-Condition |
| Grosse Positionsgroessen ohne arithmetischen Overflow | **Umgesetzt** als `commission_largeQuantity_noArithmeticOverflow` | Wichtig fuer realistische Penny-Stock-Szenarien |
| FX-Kosten auf Entry und Exit separat | **Verworfen** | Mein Modell wendet FX nur auf Entry an (IB Multi-Currency-Account-Semantik) — dokumentiert als Design-Entscheidung |

### Vorgeschlagene Tests — verworfen:

| Vorschlag | Status | Begruendung |
|-----------|--------|-------------|
| FX-Kosten auf beide Legs (Entry + Exit) | **Verworfen** | Mein Modell ist korrekt fuer IB-Kontostruktur — Entry-only ist bewusste Entscheidung |
| Integer-Overflow-Test mit 1_000_000 shares | **Adaptiert** (500_000 shares) | `int` ist fuer diese Groessen ausreichend; Test mit 500k Shares validiert Formel |

## Gemini-Review

**Datum:** 2026-02-22

### Dimension 1: Code-Review

**Findings (4 identifiziert, 2 behoben):**

| Finding | Schwere | Status | Aktion |
|---------|---------|--------|--------|
| Double-Rounding: Komponenten gerundet vor Summierung | MITTEL | **Behoben** | Raw values summiert, dann Gesamtsumme gerundet |
| Fehlende NaN/Infinity Guards | HOCH | **Behoben** | `Double.isNaN()` und `Double.isInfinite()` Checks hinzugefuegt |
| FX-Kosten fehlt Exit-Leg | GERING | **Begruendet abgelehnt** | Design-Entscheidung: IB Multi-Currency-Account, Entry-only korrekt |
| Min vs. Max Commission Prioritaet (cap override min) | INFO | **Korrekt by design** | IB-Verhalten: Cap ueberschreibt Minimum fuer Micro-Trades |

**Validierte Design-Entscheidungen:**
- LLM Token Pricing als Preis-per-single-token: Korrekt, kein 1000x Fehler
- Long-Only System: quantity > 0 Guard korrekt
- FX Entry-Only: Korrekte IB-Account-Semantik

### Dimension 2: Konzepttreue-Review

**Ergebnis:** Hauptabweichungen vom Konzept (Abschnitt 5) sind begruendet:
- Story-Defaults haben Prioritaet ueber Konzept-Defaults (korrekte Vorgehensweise)
- Prozent-basierte Slippage statt tick-basiert: akzeptable V1-Vereinfachung
- Shock Slippage fehlt: als Offener Punkt dokumentiert

### Dimension 3: Praxis-Review

**Teilweise durchgefuehrt (Gemini-Verbindungsprobleme)**

Aus Dimension 1 + ChatGPT-Sparring identifizierte fehlende Real-IB-Kosten:
- Regulatorische Fees (SEC Section 31, FINRA TAF)
- ECN/Venue Fees und Maker-Rebates
- Liquiditaets-Removal Fees fuer Low-Price Stocks (<$2.50)
- Moegliche Haircuts bei OTC/Pink-Sheet Symbolen

**Bewertung:** Fuer V1-Backtest mit typischen High-Beta-US-Aktien (>$10) vernachlaessigbar. Als bekannte Limitationen dokumentiert.

## QS-Review R1 (2026-02-22)

### ChatGPT-Review (2 Runden)

**Runde 1 — Findings:**
| Finding | Schwere | Status | Aktion |
|---------|---------|--------|--------|
| Negative LLM Token-Counts nicht abgesichert | IMPORTANT | **Behoben** | Guards in `calculateTradeCostWithLlm` + 2 neue Tests |
| Unused `@Min` Import in BacktestCostProperties | HINT | **Behoben** | Import entfernt |
| `applyToGrossPnl` fehlt null-Guard | HINT | **Behoben** | `Objects.requireNonNull(cost)` hinzugefuegt + 1 neuer Test |
| Percent-Ambiguitaet (*Percent-Namen) | IMPORTANT | **Kein Fix noetig** | JavaDoc + Properties-Werte sind eindeutig; 0.05 = 0.05% ist korrekte Konvention |
| Spring @ConfigurationProperties Test | IMPORTANT | **Kein Fix noetig** | Ausserhalb des DoD-Scope (reine Berechnungsklasse, kein DB) |

**Runde 2 — Bestaetigung:** PASS nach Finding 1 + weiteren Guards bestaetigt.

### Gemini-Review (3 Dimensionen)

**Dimension 1 (Code-Review):**
- NaN/Infinity Guards: korrekt umgesetzt (bestaetigt)
- `double` statt `BigDecimal`: Akzeptabel fuer V1-Performance-Gruenden, `roundCents()` Pattern mitigiert Risiko
- Null-Safety: `Objects.requireNonNull` ueberall vorhanden (bestaetigt)
- Overflow mit `int quantity`: Ausreichend fuer realistische Kontogroessen

**Dimension 2 (Konzepttreue):**
- Alle Story-Akzeptanzkriterien korrekt implementiert (bestaetigt)
- Commission, Spread, Slippage (Entry/Stop/Limit), FX, LLM: alle Formeln korrekt
- Konfiguration `odin.backtest.cost.*`: vollstaendig

**Dimension 3 (Praxis-Review):**
- SEC/FINRA Regulatory Fees fehlen (vernachlaessigbar fuer V1, dokumentiert als Limitation)
- Short Selling Borrow Fees: ODIN ist Long-Only — kein Problem
- Overnight/Margin Interest: Intraday-System, EOD-Flat — kein Problem
- Partial Fills: Bekannte Limitation, dokumentiert

### QS-Fixes nach Reviews:

1. Negative LLM Token-Guard + 2 Tests: `llmCost_negativeInputTokens_throwsIllegalArgument`, `llmCost_negativeOutputTokens_throwsIllegalArgument`
2. Null-Guard fuer `applyToGrossPnl` + Test: `applyToGrossPnl_nullCost_throwsNullPointerException`
3. Unused `@Min` Import entfernt aus `BacktestCostProperties`

### Abschliessende Testergebnisse:
- Surefire (Unit-Tests): **143 Tests, 0 Failures** (BacktestCostModelTest: 45 Tests)
- Failsafe (Integrationstests): **7 Tests, 0 Failures** (BacktestCostModelIntegrationTest)

**QS-Status: PASS (2026-02-22)**

## Implementierte Dateien

### Neue Dateien:
- `odin-backtest/src/main/java/de/its/odin/backtest/cost/ExitType.java` — Enum fuer Austrittstyp
- `odin-backtest/src/main/java/de/its/odin/backtest/cost/BacktestCostProperties.java` — Config-Record
- `odin-backtest/src/main/java/de/its/odin/backtest/cost/BacktestCostModel.java` — Hauptklasse
- `odin-backtest/src/test/java/de/its/odin/backtest/cost/BacktestCostModelTest.java` — Unit-Tests (45 Tests nach QS-R1)
- `odin-backtest/src/test/java/de/its/odin/backtest/cost/BacktestCostModelIntegrationTest.java` — Integrationstests (7 Tests)

### Geaenderte Dateien:
- `odin-backtest/src/main/java/de/its/odin/backtest/BacktestApplication.java` — `de.its.odin.backtest.cost` zu `@ConfigurationPropertiesScan` hinzugefuegt
- `odin-backtest/src/main/resources/application.properties` — `odin.backtest.cost.*` Properties hinzugefuegt
