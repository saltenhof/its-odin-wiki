# ODIN-033: Walk-Forward Validation Framework

**Modul:** odin-backtest
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-035 (Governance Variants)
**Geschaetzter Umfang:** L

---

## Kontext

Das Konzept definiert Walk-Forward Validation: 60/20 Train/Validate Split, Rolling 20-Tage-Fenster, Out-of-Sample Performance-Tracking. Der aktuelle BacktestRunner fuehrt einfache Single-Pass-Backtests aus. Walk-Forward ist kritisch um Overfitting zu erkennen und sicherzustellen, dass Strategie-Parameter generalisieren.

## Scope

**In Scope:**
- Neue Klasse `WalkForwardRunner` die BacktestRunner mehrfach ausfuehrt
- Train-Window: 60 Tage, Validate-Window: 20 Tage, Rolling mit 20-Tage-Schritt (konfigurierbar)
- Aggregation der Out-of-Sample-Ergebnisse ueber alle Windows
- Overfitting-Detection: Vergleich In-Sample vs. Out-of-Sample Performance
- Ergebnis: `WalkForwardReport` Record mit aggregierten Metriken

**Out of Scope:**
- Automatische Parameteroptimierung waehrend Walk-Forward
- Parallelisierung der Windows (optional spaeter, V1 sequenziell)
- UI-Integration des WalkForwardReport

## Akzeptanzkriterien

- [ ] `WalkForwardRunner` teilt Gesamtzeitraum in Train/Validate Windows korrekt auf
- [ ] Jedes Window wird als separater Backtest-Run ausgefuehrt (BacktestRunner als Delegate)
- [ ] Train-Ergebnis wird gespeichert aber NICHT fuer Gesamt-Performance gewertet
- [ ] Validate-Ergebnis geht in die Gesamt-Performance ein
- [ ] Overfitting-Score: `trainSharpe / validateSharpe` (>2.0 = Overfitting-Warnung)
- [ ] `WalkForwardReport`: Aggregierte Metriken ueber alle Validate-Windows (Sharpe, WinRate, MaxDrawdown)
- [ ] Mindestens 3 Walk-Forward-Windows fuer statistisch sinnvolles Ergebnis (Fehler wenn zu wenig Daten)
- [ ] Train/Validate/Step-Window-Groessen sind konfigurierbar (Defaults: 60/20/20 Tage)

## Technische Details

**Dateien:**
- `odin-backtest/src/main/java/de/its/odin/backtest/WalkForwardRunner.java` (neue Klasse)
- `odin-backtest/src/main/java/de/its/odin/backtest/WalkForwardReport.java` (neues Record)

**Pattern:** `WalkForwardRunner` delegiert an `BacktestRunner` — keine Duplikation der Backtest-Logik.

## Konzept-Referenzen

- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 9 "Walk-Forward Validation"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 9, Diagramm "60/20 Train/Validate Split"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 10 "Stress Tests und Ablation Tests"

## Guardrail-Referenzen

- `docs/backend/guardrails/development-process.md` — "Risikoeinstufung MEDIUM"
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen, Namensregeln
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln, keine Magic Numbers

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-backtest`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `DEFAULT_TRAIN_DAYS`, `DEFAULT_VALIDATE_DAYS`, `DEFAULT_STEP_DAYS`, `OVERFITTING_THRESHOLD`)
- [ ] Records fuer DTOs (`WalkForwardReport`)
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.backtest.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer Window-Splitting-Logik: Korrekte Berechnung der Train/Validate-Grenzen
- [ ] Unit-Tests fuer Overfitting-Score-Berechnung: > 2.0 ergibt Warnung, < 2.0 kein Warning
- [ ] Unit-Tests fuer Aggregation: Metriken aus mehreren Validate-Windows korrekt aggregiert
- [ ] Unit-Tests fuer Fehlerfall: Zu wenig Daten fuer mindestens 3 Windows → aussagekraeftige Exception
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer BacktestRunner (Port-Interface oder Stub-Implementierung)
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: `WalkForwardRunner` mit echtem (leichtgewichtigem) `BacktestRunner` ueber synthetische Daten — prueft ob Windows korrekt aufgeteilt und Ergebnisse aggregiert werden
- [ ] Integrationstest: Overfitting-Detection funktioniert mit realen Metriken aus BacktestRunner
- [ ] Integrationstest: Konfigurierbare Window-Groessen werden korrekt angewendet
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass WalkForwardRunner und BacktestRunner korrekt zusammenarbeiten und der Report vollstaendig befuellt wird.

### 2.4 Tests — Datenbank

Nicht zutreffend. `WalkForwardRunner` ist eine reine Orchestrierungsklasse ohne direkten Datenbankzugriff. Persistenz liegt beim delegierten `BacktestRunner`.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `WalkForwardRunner.java`, `WalkForwardReport.java`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei genau 3 Windows (Mindest)? Bei Zeitreihen-Luecken (Ferien, Weekends)? Bei identischen Train/Validate-Ergebnissen (Overfitting-Score = 1.0)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Off-by-one-Fehler bei Window-Berechnung, Null-Safety bei leeren Validate-Ergebnissen, korrekte Aggregation von NaN-Werten"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/09-backtesting-evaluation.md` Abschnitt 9 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Walk-Forward-Implementierung dem Konzept entspricht. Stimmen 60/20/20-Defaults, Overfitting-Schwellenwert 2.0, Mindest-Windows 3, Aggregations-Metriken mit dem Konzept ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es bei Walk-Forward-Validation in der Praxis typische Fallstricke, die hier nicht behandelt sind? Z.B. Look-Ahead-Bias, kalenderbedingte Asymmetrien, Bedeutung der Window-Groessen-Wahl?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Sequenziell vs. Parallel, Aggregations-Methodik)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- `WalkForwardRunner` nutzt `BacktestRunner` als Delegate — keine Duplikation der Backtest-Logik
- Die 60/20/20 Konfiguration muss konfigurierbar sein (kein Hardcoding)
- Parallelisierung: Verschiedene Windows KOENNEN parallel laufen (sie sind unabhaengig) — V1 sequenziell, aber Design muss Parallelisierung nicht verhindern
- ODIN-035 (Governance Variants) muss vor dieser Story abgeschlossen sein — `WalkForwardRunner` muss `GovernanceVariant` als Parameter akzeptieren
