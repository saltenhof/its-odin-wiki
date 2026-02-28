# ODIN-035: Backtest Governance Variants (A-E)

**Modul:** odin-backtest
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine
**Geschaetzter Umfang:** M

---

## Kontext

Das Konzept definiert 5 Governance-Varianten (A-E) fuer Backtests, die unterschiedliche Grade der LLM-Integration testen. Aktuell gibt es nur einen Backtest-Modus (mit oder ohne LLM-Cache). Die Varianten ermoeglichen es, den Beitrag jedes System-Bestandteils zu isolieren und zu messen — entscheidend fuer die Validierung des LLM-Nutzens.

## Scope

**In Scope:**
- 5 Governance-Varianten als Enum `GovernanceVariant` (A-E)
- BacktestRunner akzeptiert GovernanceVariant als Parameter
- Variante A: Pure Quant (kein LLM, nur KPI + Rules + QuantScore)
- Variante B: Quant + LLM Regime (LLM nur fuer Regime, keine taktischen Parameter)
- Variante C: Full Hybrid (Quant + LLM mit Dual-Key, Default-Variante)
- Variante D: LLM-Heavy (LLM-Empfehlung dominiert, Quant nur als Safety-Net)
- Variante E: LLM-Only (reines LLM, nur Risk-Limits als Hard-Stop) — fuer Vergleichszwecke
- `NoOpLlmAnalyst` als Port-Implementierung fuer Variante A

**Out of Scope:**
- UI fuer Varianten-Auswahl im Backtest-Frontend
- Automatischer Varianten-Vergleich (das ist Aufgabe von ODIN-033/034)
- Mehr als 5 Varianten

## Akzeptanzkriterien

- [ ] Enum `GovernanceVariant` mit 5 Werten (A-E) und Beschreibung/JavaDoc
- [ ] BacktestRunner akzeptiert `GovernanceVariant` als Parameter
- [ ] Variante A: `LlmAnalyst` wird durch `NoOpLlmAnalyst` ersetzt (null-Antworten)
- [ ] Variante B: LLM liefert nur Regime, keine taktischen Parameter
- [ ] Variante C: Voller Dual-Key-Modus (Default)
- [ ] Variante D: Arbiter gewichtet LLM-Vote staerker als QuantScore
- [ ] Variante E: Quant-Gates deaktiviert, nur Risk-Limits aktiv
- [ ] Ergebnisse pro Variante separat gespeichert und vergleichbar
- [ ] BacktestReport enthaelt `GovernanceVariant` als Feld
- [ ] Varianten-Logik ist in einer Factory gekapselt (nicht direkt im BacktestRunner)

## Technische Details

**Dateien:**
- `odin-backtest/src/main/java/de/its/odin/backtest/GovernanceVariant.java` (neues Enum)
- `odin-backtest/src/main/java/de/its/odin/backtest/GovernanceVariantFactory.java` (neue Factory)
- `odin-backtest/src/main/java/de/its/odin/backtest/BacktestRunner.java` (Varianten-Logik einbinden)
- `odin-brain/src/main/java/de/its/odin/brain/llm/NoOpLlmAnalyst.java` (fuer Variante A)

## Konzept-Referenzen

- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 1 "5 Governance-Varianten (A-E)"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 1, Tabelle "Varianten A-E mit Beschreibung"
- `docs/concept/09-backtesting-evaluation.md` — Abschnitt 2 "LLM-Determinismus im Backtest"

## Guardrail-Referenzen

- `docs/backend/guardrails/development-process.md` — "Risikoeinstufung MEDIUM"
- `docs/backend/guardrails/module-structure.md` — Abhaengigkeitsregeln (Fachmodul→Fachmodul verboten)
- `T:\codebase\its_odin\CLAUDE.md` — ENUM statt String fuer endliche Mengen, Port-Abstraktion

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-backtest,odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] `GovernanceVariant` als ENUM (endliche Menge)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen — insbesondere JavaDoc auf jedem Enum-Wert mit Beschreibung des Verhaltens
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.backtest.*` fuer Konfiguration
- [ ] Port-Abstraktion: `NoOpLlmAnalyst` implementiert `LlmAnalyst`-Port aus `de.its.odin.api.port`

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `GovernanceVariantFactory`: Jede der 5 Varianten erzeugt korrekte Konfiguration
- [ ] Unit-Tests fuer `NoOpLlmAnalyst`: Gibt immer null/leere Antwort zurueck, kein Exception
- [ ] Unit-Tests fuer Variante A im BacktestRunner: Quant-Only ohne LLM-Aufrufe
- [ ] Unit-Tests fuer Variante C (Default): Dual-Key-Modus korrekt aktiviert
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: BacktestRunner mit Variante A (NoOpLlmAnalyst) laeuft korrekt — keine LLM-Aufrufe, nur Quant-Entscheidungen
- [ ] Integrationstest: BacktestRunner mit Variante C (Default) — LLM und Quant arbeiten zusammen
- [ ] Integrationstest: BacktestReport enthaelt die korrekte GovernanceVariant nach dem Lauf
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass GovernanceVariantFactory, BacktestRunner und NoOpLlmAnalyst korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

Nicht zutreffend. `GovernanceVariant`-Logik ist reine Konfiguration und In-Memory-Orchestrierung ohne direkten Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `GovernanceVariant.java`, `GovernanceVariantFactory.java`, `NoOpLlmAnalyst.java`, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei Variante E wenn LLM einen Fehler liefert? Wie unterscheidet sich Variante D von C in der Gewichtung (numerisch)? Ist der Fallback bei LLM-Timeout in Variante D definiert?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, falsche Varianten-Zuordnung, Null-Safety bei NoOpLlmAnalyst, korrekte Factory-Kapselung, unbeabsichtigte Seiteneffekte zwischen Varianten"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/09-backtesting-evaluation.md` Abschnitte 1 und 2 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 5 Varianten korrekt aus dem Konzept abgeleitet sind. Stimmen die Verhaltensbeschreibungen fuer A-E mit dem Konzept ueberein? Ist NoOpLlmAnalyst die richtige Implementierung fuer Variante A?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche praktischen Probleme entstehen wenn Variante E (LLM-Only) in der Praxis laeuft und das LLM unerwartete Outputs liefert? Welche Safety-Nets fehlen moeglicherweise?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Factory-Pattern vs. Strategy-Pattern, Gewichtungs-Implementierung fuer Variante D)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- `NoOpLlmAnalyst` implementiert `LlmAnalyst`-Port und liefert immer null — RulesEngine muss mit null-LLM-Antworten korrekt umgehen
- Die Varianten-Logik sollte in einer Factory (`GovernanceVariantFactory`) gekapselt sein, nicht direkt im BacktestRunner
- Variante E ist kontrovers (reines LLM) und dient nur dem Vergleich — keine Produktions-Safety-Annahmen
- Abhaengigkeitsregel beachten: `NoOpLlmAnalyst` lebt in odin-brain, nicht in odin-backtest (Fachmodul-Grenze)
