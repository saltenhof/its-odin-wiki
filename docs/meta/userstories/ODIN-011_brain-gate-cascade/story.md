# ODIN-011: Gate-Kaskade Implementation im Brain-Modul

**Modul:** odin-brain
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-003 (GateType, GateCascadeResult)

---

## Kontext

Das Konzept ersetzt den gewichteten Quant-Score durch eine 7-Gate-Kaskade mit harten Pass/Fail-Checks. Die aktuelle `QuantValidation`-Klasse berechnet einen gewichteten Score -- dieses Modell wird durch die Gate-Kaskade ersetzt. Short-Circuit: Beim ersten Fail wird abgebrochen.

## Scope

**In Scope:**
- Neue Klasse `GateCascadeEvaluator` in `de.its.odin.brain.quant`
- 7 Gate-Implementierungen mit konfigurierbaren Schwellen
- Short-Circuit-Logik (erstes Fail stoppt Evaluation)
- Integration mit `DecisionArbiter` (Arbiter nutzt GateCascade statt QuantScore fuer Entries)
- `QuantValidation` bleibt als Fallback fuer Backtests mit altem Verhalten

**Out of Scope:**
- Aenderungen an anderen Exit-Pfaden oder Trailing-Stop-Logik
- Neue Gate-Typen ausserhalb der 7 definierten Gates
- Aenderungen am KPI-Berechnungsstack

## Akzeptanzkriterien

- [ ] Gate 1 SPREAD: Spread < maxSpreadPercent (Default 0.3%)
- [ ] Gate 2 VOLUME: Letzter Bar-Volume > minVolumeThreshold
- [ ] Gate 3 RSI: RSI nicht in Extremzone (nicht > 80 fuer Long-Entry, nicht < 20 fuer Long-Hold)
- [ ] Gate 4 EMA_TREND: Preis > EMA50, EMA50 > EMA100 (Long-Only)
- [ ] Gate 5 VWAP: Preis > VWAP (Long-Only Bias)
- [ ] Gate 6 ATR: ATR im gueltigen Range (nicht zu eng, nicht zu weit)
- [ ] Gate 7 ADX: ADX > minAdxThreshold (Default 20, Trend existiert)
- [ ] Short-Circuit: Evaluation stoppt bei erstem Fail
- [ ] `GateCascadeResult` wird im EventLog geloggt (welche Gates passed/failed)
- [ ] Alle Schwellen konfigurierbar in `BrainProperties`

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/quant/GateCascadeEvaluator.java` (neue Klasse)
- `odin-brain/src/main/java/de/its/odin/brain/quant/gates/SpreadGate.java` (und analog fuer alle 7)
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/DecisionArbiter.java` (Erweiterung)
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (neue Gate-Schwellen)

**Muster:** Jedes Gate implementiert ein Interface `EntryGate`. `GateCascadeEvaluator` iteriert die Gates in Reihenfolge und stoppt bei erstem Fail (Short-Circuit).

## Konzept-Referenzen

- `docs/concept/03-strategy-logic.md` -- Abschnitt 4.2 "Gate-Kaskade (7 Hard Gates)"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 4.2, Tabelle "7 Gates mit Schwellen"
- `docs/concept/09-backtesting-evaluation.md` -- Abschnitt 11 "Gate-Kaskade fuer Decision Logic"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH fuer Trading-Logic"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13), Port-Abstraktion

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (GateType als Enum)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.quant.*` fuer Gate-Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer alle 7 Gates einzeln (Pass-Fall + Fail-Fall)
- [ ] Unit-Test: Gesamtkaskade Short-Circuit-Verhalten (Abbruch bei erstem Fail)
- [ ] Unit-Test: Kaskade mit allen Gates passing -> ENTER-Ergebnis
- [ ] Unit-Test: EventLog-Eintrag bei Gate-Fail (welches Gate, warum)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer KPI-Snapshot-Daten (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `DecisionArbiter` mit realem `GateCascadeEvaluator` (nicht weggemockt)
- [ ] Integrationstest: Vollstaendige Kaskade mit realen Gate-Implementierungen zusammengeschaltet
- [ ] Mindestens 1 Integrationstest der den Pfad Entry-Kandidat -> GateCascade -> Arbiter-Entscheidung durchlaeuft
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: GateCascadeEvaluator-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei identischen Werten an Gate-Schwellen? Race Conditions bei Snapshot-Zugriff? Reihenfolgeabhaengigkeiten?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, fehlerhafte Short-Circuit-Logik, Performance-Probleme bei Gate-Evaluation"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/03-strategy-logic.md` Abschnitt 4.2 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Gate-Reihenfolge, Schwellenwerte, Short-Circuit-Semantik, EventLog-Format"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Was passiert wenn Indikatoren noch nicht berechnet sind (Warmup-Phase)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Interface-Design fuer EntryGate, Konfigurationsstruktur)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Die Gate-Schwellen stehen in Konzept 03, Abschnitt 4.2
- Die bestehende `QuantValidation` NICHT loeschen -- sie wird fuer Backtests mit altem Verhalten (Governance-Variante A) noch gebraucht
- Jedes Gate sollte eine eigene Klasse sein (Single Responsibility) mit Interface `EntryGate`
- ADX-Gate: ADX > 20 bedeutet "ein Trend existiert". Bei ADX < 20 ist der Markt richtungslos
- Warmup-Phase: Wenn Indikatoren noch nicht initialisiert sind (z.B. EMA50 braucht 50 Bars), muss das Gate defensiv reagieren (Fail, nicht Exception)
