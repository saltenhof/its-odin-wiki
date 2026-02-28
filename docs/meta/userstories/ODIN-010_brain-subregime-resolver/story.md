# ODIN-010: Subregime-Resolver im Brain-Modul

**Modul:** odin-brain
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-001 (Subregime-Enum)

---

## Context

Der aktuelle `RegimeResolver` bestimmt eines von 5 Regimes basierend auf KPI-Schwellen. Das Konzept fordert die Bestimmung von 20 Subregimes (4 pro Regime) mit KPI-Kriterien. Die Subregime-Information fliesst in Entry/Exit-Regeln und LLM-Kontext ein.

## Scope

**In Scope:**
- Erweiterung von `RegimeResolver` um Subregime-Bestimmung
- KPI-Kriterien fuer jedes der 20 Subregimes gemaess Konzept
- `ResolvedRegime` Record erhaelt `Subregime subregime` Feld
- Konfigurierbare Schwellen in `BrainProperties`

**Out of Scope:**
- Nutzung der Subregime-Information in Entry/Exit-Regeln (folgt in spaeteren Brain-Stories)
- LLM-Kontext-Anreicherung mit Subregime (folgt in LLM-Integration-Stories)
- Persistierung der Subregime-Bestimmung

## Acceptance Criteria

- [ ] `RegimeResolver.resolve()` liefert `ResolvedRegime` mit Regime + Subregime + Confidence
- [ ] Subregime-Logik fuer TREND_UP: STRONG (ADX>40, +DI>>-DI), MODERATE (ADX 25-40), EARLY (ADX steigend von <25), LATE (ADX fallend von >40)
- [ ] Subregime-Logik fuer TREND_DOWN: STRONG, MODERATE, EARLY, LATE -- analog TREND_UP aber mit -DI dominant
- [ ] Subregime-Logik fuer RANGE_BOUND: NARROW (BB-Width < Median), WIDE (BB-Width > Median), UPPER (nahe oberer BB), LOWER (nahe unterer BB)
- [ ] Subregime-Logik fuer HIGH_VOLATILITY: BREAKOUT (BB-Break + Volume), EXPANSION (ATR steigend), SQUEEZE (BB kontrahiert), REVERSAL (V-Umkehr)
- [ ] Subregime-Logik fuer UNCERTAIN: NO_SIGNAL (alle Indikatoren neutral), CONFLICTING (widerspruechliche Signale), TRANSITIONAL (Regime gerade gewechselt), LOW_VOLUME (Volumen < 50% Durchschnitt)
- [ ] Hysterese bleibt erhalten: Regime-Wechsel erfordern 2-Bar-Bestaetigung
- [ ] Subregime-Wechsel INNERHALB eines Regimes hat keine Hysterese (sofort)

## Technical Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/rules/RegimeResolver.java` (Erweiterung)
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (neue Subregime-Schwellen)

## Concept References

- `docs/concept/02-regime-detection.md` -- Abschnitt 2 "Subregimes", KPI-Kriterien-Tabelle
- `docs/concept/02-regime-detection.md` -- Abschnitt 3 "Regime-Hysterese"
- `docs/concept/02-regime-detection.md` -- Abschnitt 4 "Drei Ansaetze zur Regime-Bestimmung"

## Guardrail References

- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `docs/backend/guardrails/module-structure.md` -- "odin-brain: Fachmodul, abhaengig nur von odin-api"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13), insbesondere "Keine Magic Numbers"
- `T:\codebase\its_odin\CLAUDE.md` -- "Port-Abstraktion: Gegen Interfaces aus de.its.odin.api.port programmieren"

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten fuer alle Schwellenwerte (ADX, BB-Width, Volume-Ratio etc.) mit sinnvollen Defaults in `BrainProperties`
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen -- Methoden-JavaDoc beschreibt KPI-Kriterien und Hysterese-Verhalten
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.regime.{property}` fuer alle neuen Konfigurationsfelder
- [ ] Keine Abhaengigkeit von anderen Fachmodulen -- nur odin-api erlaubt
- [ ] `RegimeResolver` modular erweitert -- nicht alles in eine Methode

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests: Jedes der 20 Subregimes mit synthetischen Indikator-Werten
- [ ] Unit-Test TREND_UP_STRONG: ADX=45, +DI=30, -DI=10 -> TREND_UP_STRONG
- [ ] Unit-Test TREND_UP_MODERATE: ADX=30, +DI=20, -DI=12 -> TREND_UP_MODERATE
- [ ] Unit-Test RANGE_BOUND_NARROW: BB-Width < Median -> RANGE_BOUND_NARROW
- [ ] Unit-Test UNCERTAIN_LOW_VOLUME: Volume < 50% Durchschnitt -> UNCERTAIN_LOW_VOLUME
- [ ] Unit-Test Hysterese Regime-Wechsel: Einmalige Signal-Umkehr reicht NICHT fuer Regime-Wechsel
- [ ] Unit-Test Hysterese Regime-Wechsel: Nach 2 Bars Bestaetigung -> Regime-Wechsel erfolgt
- [ ] Unit-Test Subregime-Wechsel innerhalb Regime: Sofortiger Wechsel ohne Hysterese
- [ ] Testklassen-Namenskonvention: `RegimeResolverTest` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `RegimeResolver` liefert `ResolvedRegime` mit konsistentem Regime und Subregime (Subregime.parentRegime() == Regime)
- [ ] Integrationstest: Vollstaendiger Datenstrom -- Regime-Wechsel mit Hysterese ueber mehrere Bars korrekt verarbeitet
- [ ] Integrationstest: Alle 20 Subregimes koennen in einem synthetischen Session-Durchlauf erreicht werden
- [ ] Testklassen-Namenskonvention: `RegimeResolverIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- `RegimeResolver` greift nicht auf die Datenbank zu. Bestimmung ist rein In-Memory basierend auf Indikator-Werten.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `RegimeResolver.java`, `BrainProperties`-Erweiterung, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. ADX genau = 40 (Grenzwert STRONG/MODERATE), Regime-Wechsel genau im Moment eines Subregime-Wechsels, UNCERTAIN_TRANSITIONAL direkt nach Regime-Wechsel, alle Indikatoren genau neutral fuer NO_SIGNAL)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs bei der Hysterese-Implementierung (Off-by-one, falsche Bestaedigungszaehlung), Null-Safety bei fehlenden Indikator-Werten, Konsistenz zwischen Regime und Subregime (parentRegime() muss stimmen)"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/02-regime-detection.md` Abschnitte 2, 3, 4 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 20 Subregime-Kriterien dem Konzept entsprechen, ob die Hysterese-Regel (2-Bar fuer Regime, sofort fuer Subregime) korrekt implementiert ist, und ob die drei Ansaetze zur Regime-Bestimmung beachtet wurden"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. was passiert wenn Subregime-Kriterien innerhalb desselben Regimes ueberlappend sind (mehrere Subregimes koennten zutreffen), Prioritaetsreihenfolge bei Konflikten, Verhalten bei Simulation mit sehr wenigen Bars"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. wie TREND_UP_EARLY vs. TREND_UP_MODERATE abgegrenzt wird wenn ADX steigend aber noch < 25, Prioritaet bei sich ueberschneidenden Subregime-Kriterien)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Die KPI-Kriterien fuer Subregimes stehen detailliert in Konzept 02, Abschnitt 2
- Subregime-Bestimmung ist rein deterministisch (KPI-basiert) -- kein LLM-Input noetig
- TREND_DOWN Subregimes: STRONG, MODERATE, EARLY, LATE -- analog zu TREND_UP aber mit -DI dominant
- Der bestehende `RegimeResolver`-Code muss modular erweitert werden, nicht alles in eine Methode
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
