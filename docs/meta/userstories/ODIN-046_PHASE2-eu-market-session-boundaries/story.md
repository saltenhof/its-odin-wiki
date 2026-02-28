# ODIN-046 [PHASE2]: EU Market Session Boundaries

**Modul:** odin-data, odin-core
**Phase:** 2 -- Nicht fuer V1 vorgesehen. Umsetzung erst nach explizitem Stakeholder-Beschluss.
**Groesse:** L
**Abhaengigkeiten:** ODIN-008 (Session Boundaries)

---

## Kontext

V1 unterstuetzt ausschliesslich US-Maerkte (NYSE, NASDAQ). Phase 2 erweitert um EU-Maerkte (XETRA, LSE, Euronext) mit unterschiedlichen Session-Zeiten, Halbtages-Kalender, und Waehrungen.

## Scope

**In Scope:**
- Exchange-spezifische Session-Boundaries (XETRA 09:00-17:30 CET, LSE 08:00-16:30 GMT)
- Kalender-Integration: EU-Feiertage, Halbtage
- FX-Handling fuer EUR, GBP, CHF (P&L-Konvertierung in USD)
- Multi-Exchange Instrument-Mapping (ISIN-basiert)
- Pre-Market-Regeln fuer EU (meist kein Pre-Market)

**Out of Scope:**
- Handelslogik fuer EU-spezifische Regularien (MiFID II etc.)
- Automatisches FX-Hedging
- Realtime FX-Kurs-Subscription (nur EOD-FX-Rate fuer P&L)

## Akzeptanzkriterien

- [ ] `ExchangeSessionConfig` unterstuetzt XETRA, LSE, Euronext mit korrekten Zeitzonen und Session-Zeiten
- [ ] Kalender-Service kennt EU-Feiertage und Halbtage fuer Folgejahr
- [ ] `MarketClock` liefert korrekte Session-Boundaries je Exchange
- [ ] FX-Rate-Service: EOD-Konvertierung EUR/GBP/CHF nach USD fuer P&L
- [ ] Instrument-Mapping: ISIN → Exchange → Ticker funktioniert fuer XETRA/LSE
- [ ] Pre-Market-Logik deaktiviert fuer EU-Exchanges (konfigurierbar)
- [ ] Alle US-Maerkte funktionieren unveraendert (keine Regression)

## Technische Details

**Betroffene Klassen (Schaetzung -- konkrete Implementierung entscheidet):**
- `odin-data`: EU-spezifische Session-Boundary-Implementierung
- `odin-core`: MarketClock-Erweiterung fuer Multi-Exchange-Support
- `odin-api`: Moegliche Erweiterung der Exchange-Enum und SessionConfig-Ports
- Konfiguration unter `odin.data.session.{exchange}.{property}`

## Konzept-Referenzen

- `docs/concept/00-overview.md` -- Abschnitt 2 "US und Europa" (V1 = US only, Phase 2 = EU)
- `docs/concept/03-strategy-logic.md` -- Abschnitt 1 "Tagesablauf" (US-spezifisch, als Referenz fuer Adaption)

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Modulabhaengigkeiten, Package-Konventionen
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints
- `T:/codebase/its_odin/CLAUDE.md` -- Coding-Regeln, MarketClock-Pflicht statt `Instant.now()`

## Definition of Done

> **Hinweis Phase 2:** Diese Story wird erst umgesetzt nach explizitem Stakeholder-Beschluss. Die nachfolgende DoD gilt dann vollstaendig und ohne Ausnahme.

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data,odin-core`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten fuer Session-Zeiten, Waehrungskurse
- [ ] Records fuer DTOs (SessionConfig, ExchangeCalendar)
- [ ] ENUM fuer Exchange-Typen (XETRA, LSE, EURONEXT, NYSE, NASDAQ)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace: `odin.data.session.{exchange}.{property}`
- [ ] MarketClock verwenden: kein `Instant.now()` im Trading-Codepfad
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren
- [ ] US-Maerkte weiterhin vollstaendig funktionsfaehig (keine Regression)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer EU Session-Boundary-Berechnung (XETRA, LSE, Euronext)
- [ ] Unit-Tests fuer Feiertag-Kalender (bekannte EU-Feiertage geprueft)
- [ ] Unit-Tests fuer FX-Rate-Konvertierung (EUR/GBP/CHF → USD)
- [ ] Unit-Tests fuer ISIN-Mapping
- [ ] Regression-Tests: US-Session-Boundaries unveraendert
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet
- [ ] MarketClock-Integration: Gibt korrekte Session-Zeiten fuer EU und US Exchange
- [ ] DataPipeline-Integration: EU-Instrument wird korrekt verarbeitet (Session Start/End)
- [ ] Mindestens 1 Integrationstest der EU-Pipeline End-to-End mit echten Session-Boundaries testet

### 2.4 Tests -- Datenbank

- [ ] Falls neue DB-Tabellen oder Spalten hinzukommen: Embedded-Postgres-Tests mit Zonky
- [ ] Flyway-Migrationen werden im Test ausgefuehrt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Falls keine DB-Aenderungen: "Nicht zutreffend -- keine Schemaerweiterung"

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Implementierungscode, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Sommerzeit-Wechsel CET/BST, britische Bankfeiertage, Halbtag-Schnittstellen, gleichzeitige US+EU Pipelines, FX-Rate fehlt)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben: "Pruefe auf Bugs, Zeitzonen-Fehler (insb. DST-Uebergaenge), Null-Safety bei fehlendem FX-Kurs, Race Conditions im Kalender-Service"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + Konzept-Referenzen an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Implementierung dem Konzept entspricht -- insb. korrekter Scope (nur EU Phase 2), keine US-Regression"
- [ ] Abweichungen bewertet

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es Themen die im Konzept nicht behandelt sind? Z.B. unterschiedliche Pre-Market-Regeln je EU-Exchange, MiFID-II-Reporting-Anforderungen, Liquidity-Unterschiede EU vs US?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State live aktualisiert waehrend der Arbeit
- [ ] Design-Entscheidungen dokumentiert
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Zeitzonen-Handling ist fehleranfaellig: Immer `ZonedDateTime` statt `LocalDateTime`, DST-Uebergaenge testen
- XETRA: CET/CEST (UTC+1/UTC+2), LSE: GMT/BST (UTC+0/UTC+1) -- unterschiedliche DST-Regeln
- EU hat meist kein Pre-Market -- Pre-Market-Logik per Exchange-Config deaktivierbar machen
- FX-Rate: EOD-Kurs reicht fuer P&L-Berechnung -- kein Realtime-FX-Feed noetig
