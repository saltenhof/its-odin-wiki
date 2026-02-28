# ODIN-023: FX-Conversion in Position Sizing

**Modul:** odin-execution
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** Keine

---

## Context

Das Konzept definiert FX-Conversion als Schritt 1b im Position-Sizing: `max_risk_usd = max_risk_eur * EUR/USD_spot`. Die aktuelle `PositionSizer` und `RiskGate` nutzen einen statischen FX-Rate aus der Konfiguration. Fuer korrekte Sizing muss der aktuelle Spot-Kurs verwendet werden.

## Scope

**In Scope:**
- Erweiterung von `PositionSizer` um dynamischen FX-Rate-Parameter
- FX-Rate wird von `GlobalRiskManager` via `AccountRiskState.fxRate()` bereitgestellt
- Formel: `maxRiskUsd = maxRiskEur * fxRate`
- Position-Value-Cap: `maxPositionValue = dailyCapital * 50% * fxRate`

**Out of Scope:**
- Live FX-Feed vom Broker (V1: statischer FX-Rate aus Config reicht)
- FX-Hedging oder Multi-Currency-Accounting
- FX-Rate fuer EUR-denominierte Aktien (FX=1.0 per Default)

## Acceptance Criteria

- [ ] `PositionSizer.calculate()` nutzt FX-Rate aus AccountRiskState
- [ ] FX-Rate-Umrechnung fuer Risk-Betrag UND Position-Value-Cap
- [ ] Bei FX-Rate <= 0: Fallback auf konfiguriertem Default (1.0 fuer EUR-Aktien, 1.08 fuer USD-Aktien)
- [ ] EventLog: FX-Rate im ENTRY_SIZED Event
- [ ] Unit-Tests mit verschiedenen FX-Rates

## Technical Details

**Dateien:**
- `odin-execution/src/main/java/de/its/odin/execution/risk/PositionSizer.java` (Erweiterung)
- `odin-execution/src/main/java/de/its/odin/execution/risk/RiskGate.java` (FX-Durchreichung)

**Konfiguration:**
- `odin.execution.sizing.default-fx-rate-eur-usd` (Default: 1.08)
- Namespace gemaess `odin.{modul}.{komponente}.{property}`

## Concept References

- `docs/concept/06-risk-management.md` -- Abschnitt 2.1 "Schritt 1b: FX-Conversion"
- `docs/concept/06-risk-management.md` -- Abschnitt 2.1, Formel-Tabelle

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung MEDIUM"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln, Konfigurationskonventionen (CSpec)

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.execution.sizing.*` fuer Konfiguration
- [ ] Port-Abstraktion: `EventLog`-Interface aus `de.its.odin.api.port` fuer Logging

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests: EUR-Aktien (FX=1.0) — Risk und Position-Cap korrekt
- [ ] Unit-Tests: USD-Aktien (FX=1.08) — Umrechnung korrekt
- [ ] Unit-Tests: Edge Case FX-Rate <= 0 -> Fallback auf Default
- [ ] Unit-Tests: Position-Value-Cap wird korrekt mit FX skaliert
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer `EventLog` Port-Interface
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `PositionSizer` + `RiskGate` zusammengeschaltet — FX-Rate wird korrekt durchgereicht
- [ ] Integrationstest: AccountRiskState mit unterschiedlichen FX-Rates — Sizing-Ergebnis korrekt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests — Datenbank

Nicht zutreffend — `PositionSizer` und `RiskGate` sind Pro-Pipeline-POJOs ohne Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: erweitertem `PositionSizer`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Extreme FX-Rates (z.B. 0.5 oder 2.0)? Rounding-Probleme bei Ganzzahl-Shares?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis in `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen

- [ ] **Dimension 1 (Code-Bugs):** Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Null-Safety bei FX-Rate-Zugriff, korrekte Reihenfolge der Berechnungen (Risk DANN Position-Cap)"
- [ ] Findings bewertet und berechtigte Findings behoben
- [ ] **Dimension 2 (Konzepttreue):** Code + `docs/concept/06-risk-management.md` (Abschnitt 2.1) an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob FX-Conversion-Formel, Fallback-Logik und Event-Logging dem Konzept entsprechen"
- [ ] Abweichungen begruendet oder korrigiert
- [ ] **Dimension 3 (Praxis-Gaps):** Auftrag: "Welche praktischen FX-Szenarien koennten auftreten, die im Konzept nicht behandelt sind? (z.B. stark schwankender EUR/USD waehrend der Sitzung)"
- [ ] Neue Erkenntnisse in `protocol.md` unter "Offene Punkte" dokumentiert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Fallback-Strategie bei ungueltigem FX-Rate)
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Findings dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notes for Implementer

- V1: Statischer FX-Rate per Config reicht (kein Live-FX-Feed noetig)
- AccountRiskState hat bereits `fxRate()` Feld -- wird aktuell aus Config gelesen
- Langfristig: FX-Rate vom Broker-Account abrufen (IB liefert Account-Waehrung + FX)
