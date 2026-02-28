# ODIN-045: Audit Hash-Chain Schema Migration

**Modul:** odin-persistence, odin-audit
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** Keine (kann vor ODIN-031 laufen)

---

## Kontext

Die Hash-Chain-Integritaet (ODIN-031) benoetigt zwei neue Spalten in der `event_record` Tabelle. Diese Migration kann vorab ausgefuehrt werden, damit die Schema-Aenderung unabhaengig von der Logik-Implementierung ist.

## Scope

**In Scope:**
- Flyway-Migration: `previous_hash` und `current_hash` Spalten in `event_record`
- Beide Spalten nullable (Backward-Compatibility mit bestehenden Records ohne Hash)
- Index auf `current_hash` fuer Validierungsabfragen
- `EventRecordEntity` erhaelt die neuen Felder als optionale JPA-Attribute

**Out of Scope:**
- Hash-Berechnung und -Verkettungslogik (ist ODIN-031)
- Validierung der Hash-Kette (ist ODIN-031)
- Retroaktives Berechnen von Hashes fuer bestehende Records

## Akzeptanzkriterien

- [ ] Migration V026 (oder naechste freie Nummer): `ALTER TABLE event_record ADD COLUMN previous_hash VARCHAR(64)`, `ADD COLUMN current_hash VARCHAR(64)`
- [ ] Index: `CREATE INDEX idx_event_record_current_hash ON event_record(current_hash)`
- [ ] Bestehende Records behalten NULL fuer beide Felder (kein Default, kein Backfill)
- [ ] `EventRecordEntity` erhaelt Felder `previousHash` und `currentHash` (nullable String)
- [ ] Migration laeuft fehlerfrei auf leerer und auf bestehender Datenbank mit vorhandenen event_record-Zeilen

## Technische Details

**Dateien:**
- `odin-persistence/src/main/resources/db/migration/V026__add_hash_chain_columns.sql` (oder naechste freie)
- `odin-audit/src/main/java/de/its/odin/audit/entity/EventRecordEntity.java` (neue Felder)

**Patterns:**
- Spaltenlaenge VARCHAR(64): SHA-256 als Hex = 64 Zeichen
- `@Column(nullable = true)` fuer beide neuen Felder
- JavaDoc auf neuen Feldern erklaert: Zweck (Hash-Chain fuer Tamper-Proof-Audit), Format (SHA-256 hex)

## Konzept-Referenzen

- `docs/concept/06-risk-management.md` -- Abschnitt 7.3 "Audit-Logs (Tamper-Proof)"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- "Flyway-Migrationen in odin-persistence"
- `docs/backend/guardrails/module-structure.md` -- "DDD-Modulschnitt: Persistenz (Entities in Domain-Modulen)"
- `T:/codebase/its_odin/CLAUDE.md` -- Coding-Regeln

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-audit,odin-persistence`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final int HASH_LENGTH = 64` (oder analog)
- [ ] JavaDoc auf neuen Feldern in `EventRecordEntity` (kompakt, erklaert Zweck und Format)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc + SQL-Kommentare)
- [ ] Entity `EventRecordEntity` liegt in odin-audit (DDD-Konvention)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `EventRecordEntity`: Neue Felder korrekt initialisiert (null by default), Getter/Setter
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet (nicht alles weggemockt)
- [ ] `EventRecordEntity` Persistenz-Integration: Speichern mit null-Hashes, Lesen zurueck
- [ ] Bestehende `EventRecordEntity`-Daten bleiben lesbar nach Migration (nullable Felder)
- [ ] Mindestens 1 Integrationstest der zeigt dass bestehende Records unveraendert lesbar bleiben

### 2.4 Tests -- Datenbank

- [ ] Embedded-Postgres-Tests mit Zonky (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migration laeuft im Test fehlerfrei durch (inkl. Vorgaenger-Migrationen)
- [ ] Test: Migration auf Datenbank mit vorhandenen event_record-Zeilen -- Zeilen bleiben intakt, neue Spalten sind NULL
- [ ] Test: Index `idx_event_record_current_hash` wird korrekt erstellt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Migration-SQL, Entity-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Migration auf DB mit Millionen Records, Index-Performance bei NULL-Werten, Partial Index statt Full Index fuer performance, parallele Migrationen)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Migration-SQL + Entity-Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, korrekte NULL-Semantik, Sinnhaftigkeit des Index (NULL-Werte in Index?), fehlende Constraints, JPA-Mapping korrektheit"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/06-risk-management.md` Abschnitt 7.3 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob Schema-Design dem Tamper-Proof-Audit-Konzept entspricht -- insb. VARCHAR(64) fuer SHA-256, Index-Strategie fuer Validierungsabfragen, Backward-Compatibility"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. Partial Index nur fuer non-NULL Werte (effizienter), Performance-Auswirkung des Index, Migration-Zeitschnitzel bei grossen Tabellen?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State nach initialer Implementierung dokumentiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Partial Index vs. Full Index, VARCHAR(64) Begruendung)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt nach Test-Sparring
- [ ] Gemini-Review-Abschnitt ausgefuellt nach Review
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Migrations-Nummer vorab pruefen: Aktuelle hoechste Nummer in `odin-persistence/src/main/resources/db/migration/` feststellen (laut MEMORY.md sind V019-V021 bereits vorhanden, V025 wird von ODIN-044 belegt)
- VARCHAR(64) ist die korrekte Laenge fuer SHA-256 als Hexadezimal-String (32 Bytes = 64 Hex-Zeichen)
- Beide neuen Spalten sind nullable -- es gibt bestehende Records ohne Hash und das ist OK
- Index auf `current_hash`: Ueberlegen ob Partial Index (`WHERE current_hash IS NOT NULL`) sinnvoller ist als Full Index auf nullable Spalte -- ChatGPT/Gemini dazu befragen
