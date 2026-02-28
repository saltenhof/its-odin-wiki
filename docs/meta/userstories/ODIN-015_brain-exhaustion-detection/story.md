# ODIN-015: 3-Pillar Exhaustion Detection

**Modul:** odin-brain
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine

---

## Kontext

Das Konzept definiert eine 3-Pillar Exhaustion Detection (Extension + Climax + Rejection) als Exit-Trigger. Wenn mindestens 2 der 3 Pillars gleichzeitig aktiv sind, wird ein TACTICAL_EXIT ausgeloest. Die aktuelle ExitRules hat keine Exhaustion-Logik.

## Scope

**In Scope:**
- Neue Klasse `ExhaustionDetector` in `de.its.odin.brain.rules`
- 3 Pillars: Extension (Preis weit ueber VWAP/EMA), Climax (Volume-Spike + Range-Expansion), Rejection (Wick/Doji nach Exhaustion-Move)
- Scoring: Jeder Pillar liefert boolean, Exhaustion = mindestens 2 von 3
- Integration in `ExitRules` als zusaetzlicher Exit-Check zwischen TRAILING_STOP und REGIME_CHANGE
- Neuer ExitReason: `TACTICAL_EXIT` im ExitReason-Enum (falls noch nicht vorhanden)

**Out of Scope:**
- TACTICAL_EXIT-Mechanismus mit LLM-Bestaetigung (folgt in ODIN-016)
- Aenderungen am Trailing-Stop oder anderen Exit-Pfaden
- Neue Pillar-Typen ausserhalb der 3 definierten

## Akzeptanzkriterien

- [ ] Extension Pillar: Preis > VWAP + extensionAtrFactor * ATR UND Preis > EMA50 + extensionEmaFactor * ATR
- [ ] Climax Pillar: Volume > climaxVolumeMultiplier * avgVolume UND Bar-Range > climaxRangeMultiplier * avgRange
- [ ] Rejection Pillar: Wick-Anteil > rejectionWickPercent (z.B. > 60% Wick = Ablehnungskerze)
- [ ] Exhaustion = mindestens 2 von 3 Pillars aktiv
- [ ] Integration in ExitRules: Exhaustion-Check VOR Target-Check
- [ ] Neues ExitReason `TACTICAL_EXIT` im ExitReason-Enum
- [ ] Alle Schwellen konfigurierbar in `BrainProperties`

## Technische Details

**Dateien:**
- `odin-brain/src/main/java/de/its/odin/brain/rules/ExhaustionDetector.java` (neue Klasse)
- `odin-brain/src/main/java/de/its/odin/brain/rules/ExitRules.java` (Integration: Exhaustion-Check einfuegen)
- `odin-api/src/main/java/de/its/odin/api/model/ExitReason.java` (neuer Wert TACTICAL_EXIT, falls noch nicht vorhanden)
- `odin-brain/src/main/java/de/its/odin/brain/config/BrainProperties.java` (Schwellen fuer alle 3 Pillars)

**Wick-Formel:** `wick = max(high - close, close - low) / (high - low)`. Bei wick > 0.60 gilt als Ablehnungskerze.

## Konzept-Referenzen

- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 6 "3-Pillar Exhaustion Detection"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 6, Tabelle "3 Pillars"
- `docs/concept/05-stops-profit-protection.md` -- Abschnitt 7 "TACTICAL_EXIT-Mechanismus"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Package-Konventionen, Namensregeln
- `docs/backend/guardrails/development-process.md` -- "Risikoeinstufung HIGH"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13)

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain,odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten (Wick-Schwellen, Default-Multiplikatoren)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen (ExitReason als Enum mit TACTICAL_EXIT)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.brain.rules.exhaustion.*`
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Test: Extension Pillar einzeln -- true wenn beide Bedingungen erfuellt, false wenn eine fehlt
- [ ] Unit-Test: Climax Pillar einzeln -- true bei Volume-Spike + Range-Expansion, false bei nur einer Bedingung
- [ ] Unit-Test: Rejection Pillar einzeln -- Wick > 60% -> true, Wick < 60% -> false
- [ ] Unit-Test: 2-von-3-Logik -- alle 8 Kombinationen (0, 1, 2, 3 Pillars aktiv)
- [ ] Unit-Test: Integration in ExitRules -- Exhaustion triggert, bevor Target-Check erreicht wird
- [ ] Unit-Test: Wick-Berechnung bei Doji (high == low) -- Division durch Null abgefangen
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks fuer MarketSnapshot (keine Spring-Kontexte noetig)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `ExhaustionDetector` zusammen mit `ExitRules` (reale Klassen zusammengeschaltet)
- [ ] Integrationstest: Vollstaendiger Exit-Entscheidungspfad -- Snapshot mit Exhaustion-Merkmalen -> TACTICAL_EXIT wird produziert
- [ ] Mindestens 1 Integrationstest der die Reihenfolge TRAILING_STOP -> Exhaustion -> Target in ExitRules bestaetigt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- diese Story hat keinen Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: ExhaustionDetector-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei Doji (high == low)? Was wenn Volume-Daten fehlen (kein avgVolume berechnet)? Kann Exhaustion in der ersten Minute nach Open zu fruehzeitig triggern?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Division-durch-Null bei Wick-Berechnung, Null-Safety bei Snapshot-Feldern, Reihenfolge der Exit-Checks in ExitRules"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/05-stops-profit-protection.md` Abschnitte 6 und 7 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Pillar-Definitionen, 2-von-3-Semantik, Prioritaet in ExitRules, ExitReason-Wert"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Kann Exhaustion bei duennem Volumen (Pre-Market) faelschlicherweise triggern? Sollte Exhaustion nach einem erfolgreichen TACTICAL_EXIT deaktiviert werden bis zur naechsten Position?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Wick-Formel, avgVolume-Baseline, Check-Reihenfolge)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Extension-Pillar muss relativ zu ATR sein, nicht absolute Distanz
- Climax-Volume: Vergleich mit SMA(20) des Bar-Volumes (avgVolume = rollendes Fenster)
- Rejection: `wick = max(high - close, close - low) / (high - low)`. Bei `high == low` (Doji-Edge-Case): wick = 0 oder speziell behandeln
- Dieses Feature ist ein starker Indikator fuer "Trendende" und verhindert, dass man am Top kauft oder zu lange haelt
- ODIN-016 baut auf diesem Detector auf: TACTICAL_EXIT mit LLM-Bestaetigung. Diese Story liefert nur den Detector, die LLM-Bestaetigung kommt in ODIN-016
