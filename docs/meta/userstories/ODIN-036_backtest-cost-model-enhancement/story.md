# ODIN-036: Enhanced Cost Model fuer Backtests

**Modul:** odin-backtest
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine
**Geschaetzter Umfang:** M

---

## Kontext

Das Konzept definiert ein detailliertes Cost Model fuer Backtests: Commissions, Spread, Slippage, FX-Kosten. Der aktuelle Backtest-Runner hat einen einfachen `TransactionCostCalculator`. Das erweiterte Modell soll realistischere Ergebnisse liefern und Overfitting durch zu optimistische Kosten-Annahmen verhindern.

## Scope

**In Scope:**
- Erweiterung von `TransactionCostCalculator` oder neue Klasse `BacktestCostModel`
- Commission: Fixed per Trade + Variable per Share (konfigurierbar, IB-Staffeln)
- Spread-Cost: Estimated Spread als Half-Spread bei Entry/Exit
- Slippage: Konfigurierbar (Default 0.05% Entry, 0.1% Stop-Exit)
- FX-Cost: EUR/USD Conversion-Spread (Default 0.003%)
- LLM-Call-Kosten (optional): Kosten pro LLM-Call fuer ROI-Berechnung

**Out of Scope:**
- Realtime-Kostenberechnung fuer Live-Trading (nur Backtest)
- IB-spezifische Tiered-Pricing-Staffeln (konfigurierbar, aber nicht automatisch aus IB-API abgerufen)
- Steuerliche Kosten

## Akzeptanzkriterien

- [ ] Commission: $1.00 pro Trade + $0.005 per Share (IB Default, konfigurierbar)
- [ ] Spread-Cost: `0.5 * estimatedSpread * quantity * price`
- [ ] Slippage Entry: `entryPrice * slippageEntryPercent * quantity`
- [ ] Slippage Exit: `exitPrice * slippageExitPercent * quantity` (hoeher bei Stop-Exit als bei Limit-Exit)
- [ ] FX-Cost: `positionValue * fxSpreadPercent` (nur bei USD-Aktien mit EUR-Account)
- [ ] Gesamt-Kosten pro Trade: `Commission + Spread + Slippage + FX`
- [ ] Kosten werden vom Brutto-P&L abgezogen: `Netto-P&L = Brutto-P&L - Kosten`
- [ ] LLM-Kosten (optional): `inputTokens * pricePerInputToken + outputTokens * pricePerOutputToken`
- [ ] Alle Kosten-Parameter konfigurierbar ueber `@ConfigurationProperties`

## Technische Details

**Dateien:**
- `odin-execution/src/main/java/de/its/odin/execution/risk/TransactionCostCalculator.java` (Erweiterung)
- `odin-backtest/src/main/java/de/its/odin/backtest/BacktestCostModel.java` (neue Klasse falls benoetigt)

**Konfiguration:** Namespace `odin.backtest.cost.*` mit Record-based `@ConfigurationProperties @Validated`

## Konzept-Referenzen

- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 4 "Cost Model"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 4, Tabelle "Kostenkomponenten"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 16 "LLM-Kostenoptimierung"

## Guardrail-Referenzen

- `docs/backend/guardrails/development-process.md` — "Risikoeinstufung MEDIUM"
- `T:\codebase\its_odin\CLAUDE.md` — Konfiguration (CSpec), keine Magic Numbers, explizite Typen

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution,odin-backtest`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `DEFAULT_COMMISSION_FIXED`, `DEFAULT_COMMISSION_PER_SHARE`, `DEFAULT_SLIPPAGE_ENTRY`, `DEFAULT_SLIPPAGE_EXIT`, `DEFAULT_FX_SPREAD`)
- [ ] Records fuer DTOs und `@ConfigurationProperties`
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.backtest.cost.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer Commission-Berechnung: Fixed + Variable korrekt, Min/Max-Checks
- [ ] Unit-Tests fuer Spread-Cost: Formel korrekt fuer verschiedene Preise und Mengen
- [ ] Unit-Tests fuer Slippage Entry: Korrekte Prozent-Berechnung
- [ ] Unit-Tests fuer Slippage Exit: Hoehere Rate bei Stop-Exit vs. Limit-Exit
- [ ] Unit-Tests fuer FX-Cost: Nur wenn USD-Aktie mit EUR-Account, sonst 0
- [ ] Unit-Tests fuer Gesamt-Kosten: Alle Komponenten korrekt addiert
- [ ] Unit-Tests fuer LLM-Kosten (optional): Token-basierte Berechnung
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Backtest-Lauf mit erweitertem Cost Model — Netto-P&L korrekt berechnet (kleiner als Brutto-P&L)
- [ ] Integrationstest: Konfigurierbare Parameter werden aus `@ConfigurationProperties` korrekt geladen
- [ ] Integrationstest: Verschiedene Slippage-Konfigurationen ergeben unterschiedliche Netto-P&L
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass das Cost Model korrekt in den Backtest-Lauf integriert ist und realistische P&L-Werte liefert.

### 2.4 Tests — Datenbank

Nicht zutreffend. Das Cost Model ist eine reine Berechnungsklasse ohne Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `TransactionCostCalculator.java` (Erweiterung), `BacktestCostModel.java`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei Penny-Stocks (sehr niedrige Preise)? Was bei extremen Slippage-Szenarien? Ist das FX-Modell fuer gemischte USD/EUR-Portfolios korrekt?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Rundungsfehler (Geldbetraege!), Null-Safety bei optionalen Kostenkomponenten, korrekte Formel-Implementierung, Ueberlauf bei grossen Positionen"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/09-backtesting-evaluation.md` Abschnitt 4 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob das Cost Model dem Konzept entspricht. Stimmen alle Kostenformeln, Default-Werte und Konfigurationspunkte mit der Konzept-Tabelle ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche Kosten entstehen im echten IB-Trading, die in diesem Modell fehlen? Z.B. Borrowing-Costs bei Short, Options-Praemien, Regulatorische Fees, SEC-Tax?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Erweiterung vs. neue Klasse, Rundungs-Strategie fuer Geldbetraege)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- IB Tiered Pricing: $0.0035/Share mit Min $0.35, Max 1% des Trade-Werts — als konfigurierbare Defaults implementieren
- `TransactionCostCalculator` existiert bereits — erweitern, nicht ersetzen
- Slippage bei Stop-Exit sollte hoeher sein als bei Limit-Entry (Marktdruck bei Stops ist real)
- Rundung: Geldbetraege auf Cent genau (`BigDecimal` oder konsistente `double`-Rundung), keine Floating-Point-Fehler akkumulieren lassen
