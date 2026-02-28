# ODIN-049 [PHASE2]: Universe Screening

**Modul:** odin-core, odin-data
**Phase:** 2 -- Nicht fuer V1 vorgesehen. Umsetzung erst nach explizitem Stakeholder-Beschluss.
**Groesse:** L
**Abhaengigkeiten:** Keine

---

## Kontext

V1 verwendet manuell konfigurierte Instrument-Listen. Phase 2 soll ein automatisches Screening der handelsbaren Aktien implementieren: Pre-Market-Scan nach Liquiditaet, Volatilitaet, und Nachrichtenlage. Die besten 2-3 Kandidaten werden automatisch als Pipelines gestartet.

## Scope

**In Scope:**
- Pre-Market Scanner: Volumen-Screening, Gap-Detection, News-Check
- Scoring-Modell fuer Instrument-Selektion (Punkte-basiert)
- Dynamisches Pipeline-Management: Instrumente waehrend des Tages wechseln
- Instrument-Watchlist mit konfigurierbaren Filtern (Min-Volume, Min-ATR, etc.)

**Out of Scope:**
- Fundamentaldaten-Analyse (nur technische/quantitative Kriterien)
- EU-Maerkte (Scope ist US-Maerkte, analog V1)
- Realtime-Nachrichten-Parsing (nur Nachrichten-Flag aus IB-API oder konfigurierbarer Drittdienst)

## Akzeptanzkriterien

- [ ] Pre-Market Scanner laeuft taeglich im Zeitfenster 08:00-09:00 ET (konfigurierbar)
- [ ] Volumen-Screening: Relativ zum 20-Tage-Durchschnitt (Mindest-Faktor konfigurierbar)
- [ ] Gap-Detection: Overnight-Gap berechnet (abs und relativ zum ATR)
- [ ] Scoring-Modell: Gewichtete Summe aus Volumen, Gap, Volatilitaet (Gewichte konfigurierbar)
- [ ] Top-2-3 Kandidaten werden automatisch als Pipelines gestartet
- [ ] Dynamisches Wechseln: Schwache Performer koennen intraday ersetzt werden (konfigurierbar ein/aus)
- [ ] Watchlist konfigurierbar: Symbol-Liste + Filterkriterien in `application.properties`
- [ ] Manuelles Override bleibt moeglich: Konfigurierte Instrumente haben Vorrang

## Technische Details

**Betroffene Klassen (Schaetzung):**
- `odin-data`: PreMarketScanner, InstrumentScorer, ScanResult
- `odin-core`: PipelineFactory-Erweiterung fuer dynamisches Starten, ScreeningOrchestrator
- `odin-api`: ScanResult-DTO, ScoringConfig
- Konfiguration: `odin.screening.{property}`

## Konzept-Referenzen

- `docs/concept/00-overview.md` -- Abschnitt 2 "1-3 Instrumente parallel"
- `docs/concept/03-strategy-logic.md` -- Abschnitt 1 "Pre-Market Phase: Instrument-Selektion"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Modulabhaengigkeiten (odin-core orchestriert, odin-data liefert Daten)
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints
- `T:/codebase/its_odin/CLAUDE.md` -- MarketClock-Pflicht, Coding-Regeln, Lean-Prinzip

## Definition of Done

> **Hinweis Phase 2:** Diese Story wird erst umgesetzt nach explizitem Stakeholder-Beschluss. Die nachfolgende DoD gilt dann vollstaendig und ohne Ausnahme.

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core,odin-data`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- Konstanten/Konfiguration fuer alle Schwellenwerte und Gewichte
- [ ] Records fuer DTOs (ScanResult, ScoringConfig)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace: `odin.screening.{property}`
- [ ] MarketClock verwenden: kein `Instant.now()` im Trading-Codepfad
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren
- [ ] `@ConfigurationProperties` als Record + `@Validated` fuer ScoringConfig

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer InstrumentScorer: Scoring-Berechnung mit bekannten Inputs und erwarteten Outputs
- [ ] Unit-Tests fuer Gap-Detection: Overnight-Gap-Berechnung (absolute und relative Werte)
- [ ] Unit-Tests fuer Volumen-Screening: Faktor-Berechnung gegen 20-Tage-Durchschnitt
- [ ] Unit-Tests fuer Top-N-Selektion: Korrekte Sortierung und Auswahl der besten Kandidaten
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet
- [ ] PreMarketScanner + InstrumentScorer Integration: Vollstaendiger Scan mit synthetischen Marktdaten
- [ ] ScreeningOrchestrator + PipelineFactory Integration: Top-Kandidaten starten korrekte Pipelines
- [ ] Manuelles Override Integration: Konfigurierte Instrumente haben Vorrang vor Scanner-Ergebnis
- [ ] Mindestens 1 Integrationstest der vollstaendige Screening-zu-Pipeline-Kette abdeckt

### 2.4 Tests -- Datenbank

- [ ] Falls Scan-Ergebnisse persistiert werden: Embedded-Postgres-Tests mit Zonky
- [ ] Falls keine Persistenz: "Nicht zutreffend -- Scan-Ergebnisse sind transient"
- [ ] Entscheidung dokumentieren im `protocol.md`

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Implementierungscode, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. keine Kandidaten uebrig nach Filterung, alle Kandidaten gleich gescort, Scanner-Fehler kurz vor Market-Open, dynamischer Wechsel waehrend offener Position)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben: "Pruefe auf Bugs, Division-by-Zero bei Volumen-Berechnung, Null-Safety bei fehlenden Marktdaten, Race Conditions bei dynamischem Pipeline-Wechsel"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + Konzept-Referenzen an Gemini
- [ ] Auftrag: "Pruefe ob Implementierung dem Konzept entspricht -- insb. Pre-Market-Timing, korrekte Scoring-Kriterien, Anzahl-Limit fuer Pipelines"
- [ ] Abweichungen bewertet

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es Themen die im Konzept nicht behandelt sind? Z.B. Was passiert wenn ein automatisch gestartetes Instrument spaeter keine Daten mehr liefert? Wie verhindert man, dass das Screening dieselben schlechten Instrumente immer wieder auswaehlt (Feedback-Loop)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State live aktualisiert waehrend der Arbeit
- [ ] Design-Entscheidungen dokumentiert (z.B. Persistenz von Scan-Ergebnissen ja/nein)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Pre-Market-Daten kommen ueber dieselbe IB TWS API -- sicherstellen dass Pre-Market-Daten-Subscription aktiv ist
- Scoring-Gewichte MUESSEN konfigurierbar sein (kein Hardcoding) -- sonst entsteht implizites Overfitting auf Testdaten
- Dynamischer Wechsel waehrend offener Position ist heikel: Was passiert mit der alten Pipeline? Soft-Exit erzwingen? Vorab mit Stakeholder klaeren
- Fallback: Wenn Scanner keine Kandidaten liefert (z.B. technischer Fehler) → manuell konfigurierte Liste verwenden
