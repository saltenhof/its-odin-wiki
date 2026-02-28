# ODIN-044: Cycle-Tabelle in Persistenz

**Modul:** odin-persistence, odin-execution
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** ODIN-006 (Cycle Tracking Model)

---

## Kontext

Das Datenmodell hat `trading_run` und `trade` Tabellen. Fuer Multi-Cycle-Day fehlt eine Cycle-Tabelle die das Bindeglied zwischen Run und Trade darstellt. Ein Cycle gruppiert alle Trades einer Re-Entry-Sequenz.

## Scope

**In Scope:**
- Flyway-Migration: Neue Tabelle `cycle` mit FK zu `trading_run` und `trade`
- Entity `CycleEntity` in odin-execution
- Repository `CycleRepository` mit grundlegenden Abfragemethoden
- Felder: cycleNumber, startTime, endTime, pnl, entryReason, trades (1:N)

**Out of Scope:**
- Service-Logik fuer Cycle-Management (ist andere Story)
- UI-Darstellung von Cycles
- Retroaktives Backfill bestehender Trade-Daten

## Akzeptanzkriterien

- [ ] Tabelle `cycle`: id (BIGSERIAL PK), trading_run_id (FK), cycle_number (INT), start_time (TIMESTAMPTZ), end_time (TIMESTAMPTZ nullable), pnl (NUMERIC), entry_reason (VARCHAR)
- [ ] Trade-Tabelle erhaelt `cycle_id` FK (nullable fuer Backward-Compatibility)
- [ ] `CycleEntity` mit vollstaendigem JPA-Mapping (Annotationen, Relationen)
- [ ] `CycleRepository` mit `findByTradingRunId(Long tradingRunId)` und `findByTradingRunIdOrderByCycleNumberAsc`
- [ ] Flyway-Migration V025 (oder naechste freie Nummer -- vorab pruefen)
- [ ] Migration laeuft fehlerfrei auf leerer und auf bestehender Datenbank

## Technische Details

**Dateien:**
- `odin-persistence/src/main/resources/db/migration/V025__create_cycle.sql` (oder naechste freie)
- `odin-execution/src/main/java/de/its/odin/execution/persistence/CycleEntity.java`
- `odin-execution/src/main/java/de/its/odin/execution/persistence/CycleRepository.java`

**Patterns:**
- Entity im Domainmodul (odin-execution), nicht in odin-persistence (DDD-Prinzip)
- `@Column(nullable = true)` fuer end_time und cycle_id FK (Backward-Compatibility)
- JavaDoc auf allen public Klassen, Methoden, Attributen
- Keine Magic Numbers -- Konstanten fuer Spaltenlaengen

## Konzept-Referenzen

- `docs/concept/00-overview.md` -- Abschnitt 5 "Normative Terminologie: Trade, Cycle, Tranche"
- `docs/concept/09-backtesting-evaluation.md` -- Abschnitt 13 "Datenbankschema"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- "DDD-Modulschnitt: Persistenz (Entities in Domain-Modulen)"
- `docs/backend/guardrails/module-structure.md` -- "Flyway-Migrationen in odin-persistence"
- `T:/codebase/its_odin/CLAUDE.md` -- Coding-Regeln, Persistenz-Architektur

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-execution,odin-persistence`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten fuer Spaltenlaengen etc.
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc + SQL)
- [ ] Entity liegt in odin-execution (nicht in odin-persistence) -- DDD-Konvention

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `CycleEntity`-Konstruktion und Feld-Zugriffe
- [ ] Unit-Tests pruefen korrekte JPA-Annotationen (soweit ohne DB testbar)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet (nicht alles weggemockt)
- [ ] `CycleRepository` Integration: Speichern und Lesen von `CycleEntity` gegen echte DB
- [ ] `findByTradingRunId` Integration: Korrekte Ergebnisse bei mehreren Cycles pro Run
- [ ] Backward-Compatibility: Bestehende Trades ohne cycle_id bleiben lesbar
- [ ] Mindestens 1 Integrationstest der die vollstaendige Persistenz-Strecke abdeckt

### 2.4 Tests -- Datenbank

- [ ] Embedded-Postgres-Tests mit Zonky (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migration V025 wird im Test ausgefuehrt und laeuft fehlerfrei
- [ ] `CycleRepository`-Methoden gegen echte Embedded-DB getestet
- [ ] Test: Migration laeuft auch wenn bestehende Trade-Daten vorhanden (nullable FK)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Entity-Code, Repository-Code, Migration-SQL, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Cycle ohne Trades, mehrere gleichzeitige Cycles fuer denselben Run, negative PnL, fehlende end_time bei laufendem Cycle)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code + Migration-SQL an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, fehlende DB-Constraints, falsche JPA-Annotationen, N+1-Query-Risiken in Repository-Methoden, fehlende Indexes"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/00-overview.md` Abschnitt 5 + `docs/concept/09-backtesting-evaluation.md` Abschnitt 13 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Schema und Entity dem Konzept entsprechen -- insb. Felder, Relationen, Normative Terminologie fuer Cycle"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Archivierung alter Cycles, Index-Strategie fuer grosse Datenmengen, Cascade-Delete-Verhalten wenn trading_run geloescht wird?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State nach initialer Implementierung dokumentiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Wahl der Migrations-Nummer, Cascade-Verhalten)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt nach Test-Sparring
- [ ] Gemini-Review-Abschnitt ausgefuellt nach Review
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Migrations-Nummer vorab pruefen: Aktuelle hoechste Nummer in `odin-persistence/src/main/resources/db/migration/` feststellen, naechste freie nehmen (V025 ist Schaetzung)
- Entity-Platzierung: `CycleEntity` gehoert in `odin-execution`, NICHT in `odin-persistence` (DDD-Konvention -- Entities in Domain-Modulen)
- `cycle_id` FK auf `trade` ist nullable -- Backward-Compatibility mit Trades die vor diesem Schema existieren
- Vorab pruefen ob V019-V024 bereits existieren (laut MEMORY.md sind V019-V021 bereits vorhanden)
