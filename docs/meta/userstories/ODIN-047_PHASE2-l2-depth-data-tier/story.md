# ODIN-047 [PHASE2]: L2-Depth Data Tier

**Modul:** odin-data, odin-broker
**Phase:** 2 -- Nicht fuer V1 vorgesehen. Umsetzung erst nach explizitem Stakeholder-Beschluss.
**Groesse:** XL
**Abhaengigkeiten:** Keine

---

## Kontext

V1 arbeitet mit OHLCV_ONLY und BASIC_QUOTES. Das Konzept definiert L2_DEPTH als optionale dritte Tier-Stufe fuer Live-Execution-Enhancement. L2-Daten (Orderbuch) ermoeglichen bessere Spread-Schaetzung, Liquiditaetserkennung, und optimierte Order-Platzierung.

## Scope

**In Scope:**
- L2-Subscription ueber IB TWS API (`reqMktDepth`)
- Orderbuch-Aggregation (Top 5 Levels Bid/Ask)
- Spread-Berechnung aus realem Bid/Ask statt Proxy
- Fill-Wahrscheinlichkeits-Schaetzung basierend auf Orderbuch-Tiefe
- Execution-Enhancement: Limit-Preis-Optimierung basierend auf Orderbuch

**Out of Scope:**
- L2-Daten fuer Backtesting (historische L2-Daten werden nicht gespeichert)
- Full Orderbook > Top 5 Levels
- HFT-artige Execution-Optimierung (ODIN ist kein HFT-System)

## Akzeptanzkriterien

- [ ] `reqMktDepth` wird pro Instrument subscribed wenn L2_DEPTH-Tier aktiv
- [ ] Orderbuch-Aggregation: Top 5 Bid-Levels und Top 5 Ask-Levels verfuegbar
- [ ] Spread-Berechnung: Best Bid / Best Ask aus realem Orderbuch
- [ ] Liquiditaets-Score basierend auf Orderbuch-Tiefe (Volumen der Top-5-Levels)
- [ ] Execution-Enhancement: Limit-Order-Preis wird an Orderbuch ausgerichtet
- [ ] L2_DEPTH ist optional und konfigurierbar -- OHLCV_ONLY und BASIC_QUOTES weiterhin vollstaendig funktionsfaehig
- [ ] IB TWS API Rate-Limits fuer `reqMktDepth` werden respektiert

## Technische Details

**Betroffene Klassen (Schaetzung):**
- `odin-broker`: IB TWS API L2-Handler (`IEWrapper.updateMktDepth`)
- `odin-data`: L2-Aggregator, OrderBookSnapshot
- `odin-api`: Neue Port-Erweiterung fuer L2-Daten (falls noetig)
- Konfiguration: `odin.data.tier` Enum-Erweiterung um `L2_DEPTH`

## Konzept-Referenzen

- `docs/concept/01-data-pipeline.md` -- Abschnitt 1 "3 Daten-Tiers: L2_DEPTH"
- `docs/concept/07-oms-execution.md` -- Abschnitt 13 "L2-Execution-Enhancement"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Modulabhaengigkeiten (odin-broker → odin-api, nicht umgekehrt)
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints
- `T:/codebase/its_odin/CLAUDE.md` -- IB TWS API 10.4x, Coding-Regeln

## Definition of Done

> **Hinweis Phase 2:** Diese Story wird erst umgesetzt nach explizitem Stakeholder-Beschluss. Die nachfolgende DoD gilt dann vollstaendig und ohne Ausnahme.

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-data,odin-broker`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- Konstanten fuer Max-Levels (5), Rate-Limit-Schwellen
- [ ] Records fuer OrderBookSnapshot, L2Level DTOs
- [ ] ENUM fuer Seiten (BID, ASK)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] MarketClock verwenden: kein `Instant.now()` im Trading-Codepfad
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren
- [ ] L2_DEPTH optional: OHLCV_ONLY und BASIC_QUOTES weiterhin voll funktionsfaehig

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer Orderbuch-Aggregation (Top-5-Logik, korrekte Bid/Ask-Sortierung)
- [ ] Unit-Tests fuer Spread-Berechnung aus L2-Daten
- [ ] Unit-Tests fuer Liquiditaets-Score-Berechnung
- [ ] Unit-Tests fuer Execution-Enhancement-Logik (Limit-Preis-Optimierung)
- [ ] Regression-Tests: OHLCV_ONLY und BASIC_QUOTES unveraendert
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet
- [ ] L2-Handler + Aggregator-Integration: IB-Callback → OrderBookSnapshot korrekt aggregiert
- [ ] Execution-Enhancement-Integration: OrderBookSnapshot → optimierter Limit-Preis
- [ ] Mindestens 1 Integrationstest der L2-Strecke End-to-End (Callback → Aggregation → Snapshot → Enhancement)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- L2-Daten werden nicht persistiert (nur In-Memory-Snapshot).

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Implementierungscode, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. leeres Orderbuch, IB TWS Disconnect waehrend L2-Subscription, rapid Order-Updates (Flooding), Instrument ohne L2-Daten)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben: "Pruefe auf Bugs, Race Conditions im L2-Handler (IB-Callbacks auf Thread-Pool), Memory Leaks bei Orderbuch-Updates (unbegrenztes Wachstum?), Thread-Safety des OrderBookSnapshot"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/01-data-pipeline.md` + `docs/concept/07-oms-execution.md` an Gemini
- [ ] Auftrag: "Pruefe ob Implementierung dem Konzept entspricht -- insb. Tier-Logik, Enhancement-Mechanismus, Optionalitaet"
- [ ] Abweichungen bewertet

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es Themen die im Konzept nicht behandelt sind? Z.B. IB-Gebuehren fuer L2-Daten-Subscription, Latenz-Charakteristik von L2-Updates vs. OHLCV, Dark Pool Orders die im Orderbuch nicht sichtbar sind?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State live aktualisiert waehrend der Arbeit
- [ ] Design-Entscheidungen dokumentiert (z.B. Thread-Safety-Strategie fuer Orderbuch)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- XL-Story: Erwaegen ob Sub-Stories sinnvoll sind (z.B. ODIN-047a: L2-Subscription, ODIN-047b: Aggregation, ODIN-047c: Execution-Enhancement)
- IB TWS API: `reqMktDepth` hat Rate-Limits -- IB-Dokumentation konsultieren (TWS API 10.4x)
- L2-Callbacks kommen auf IB-Worker-Threads -- Thread-Safety des Orderbuchs sicherstellen
- Orderbuch muss bounded sein: Bei Flood von Updates nur Top-5 behalten, Rest verwerfen
- `reqMktDepth` kostet u.U. extra IB-Gebuehren -- Stakeholder vorab informieren
