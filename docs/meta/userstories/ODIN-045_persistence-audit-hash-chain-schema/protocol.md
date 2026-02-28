# ODIN-045: Audit Hash-Chain Schema — Implementierungsprotokoll

## Working State (Round 2 — ABGESCHLOSSEN)

Alle Akzeptanzkriterien erfuellt. ChatGPT-Sparring und Gemini-Review (alle 3 Dimensionen, zweite Runde) abgeschlossen. Unit-Tests (12) und Integration-Tests (13) grueen.

**QS Round 2 (2026-02-21): PASS** — Alle R1-Findings behoben. qa-report-r2.md erstellt. Story abgeschlossen.

## Erstellte/Geaenderte Dateien

| Datei | Aktion | Beschreibung |
|-------|--------|-------------|
| `odin-persistence/.../V026__add_hash_chain_columns.sql` | Erstellt | Flyway-Migration: previous_hash + current_hash Spalten, Partial Index |
| `odin-audit/.../EventRecordEntity.java` | Geaendert | Neue Felder previousHash/currentHash + HASH_HEX_LENGTH-Konstante + Getter/Setter + updatable=false |
| `odin-audit/.../EventRecordEntityTest.java` | Geaendert | 5 neue Unit-Tests fuer Hash-Felder, assertTrue-Style korrigiert |
| `odin-audit/.../EventRecordHashChainIntegrationTest.java` | Erstellt | 13 Integrationstests (Zonky Embedded Postgres, 5 neue Edge-Case-Tests) |
| `odin-audit/src/test/resources/application.properties` | Erstellt | Test-Konfiguration (Flyway, JPA) |
| `odin-audit/src/test/resources/db/migration/V000__stub_timescaledb.sql` | Erstellt | TimescaleDB-Stub fuer Embedded Postgres |
| `odin-audit/pom.xml` | Geaendert | Zonky-Dependencies, Failsafe-Plugin |

## Design-Entscheidungen

### Migrations-Nummer V026
V025 wurde von ODIN-044 (Cycle) belegt. V026 ist die naechste freie Nummer.

### Partial Index (WHERE current_hash IS NOT NULL)
Bewusste Entscheidung fuer einen Partial Index statt Full Index:
- Legacy-Records ohne Hash-Werte werden nicht indexiert (Speicherersparnis)
- Nur Records mit aktivierter Hash-Chain nutzen den Index fuer Validierungsabfragen
- Folgt der Empfehlung aus der Story-Definition

### VARCHAR(64) fuer SHA-256 Hex
SHA-256 erzeugt 32 Bytes = 64 Hex-Zeichen. Exakte Laenge als `private static final int HASH_HEX_LENGTH = 64` Konstante. Konstante steht am Klassenanfang (vor Instanzfeldern) — Java-Konvention.

### Nullable Spalten (keine NOT NULL Constraint)
Beide Hash-Spalten sind nullable:
- Bestehende Records (Pre-V026) behalten NULL — kein Backfill noetig
- Erste Record einer Hash-Chain hat `previous_hash = NULL` (kein Vorgaenger)
- Legacy-Records ohne Hash-Berechnung haben beide Felder NULL

### updatable = false auf Hash-Spalten
Hash-Felder sind `@Column(updatable = false)` — konsistent mit dem Append-Only-Charakter des Systems. Hashes werden einmalig beim Insert gesetzt (durch ODIN-031). JPA darf keine UPDATE-Statements fuer diese Spalten ausgeben. Gefunden durch Gemini-Review Round 2.

### Cross-Module FK via JDBC statt @ManyToOne
Integration-Test benoetigt einen `trading_run`-Eintrag fuer die FK-Constraint auf `event_record.run_id`. Da odin-audit nicht von odin-execution abhaengt (DDD-Modulgrenze), wird der trading_run per `JdbcTemplate.update()` direkt eingefuegt.

### @EntityScan/@EnableJpaRepositories in TestConfig
Die TestConfig-Klasse liegt in `de.its.odin.audit.entity`, aber das Repository in `de.its.odin.audit.repository`. @DataJpaTest scannt standardmaessig nur das Package der Config-Klasse. Explizite @EntityScan/@EnableJpaRepositories auf `de.its.odin.audit` notwendig.

## ChatGPT-Sparring (DoD 2.5)

Durchgefuehrt in Round 2 (Remediation). ChatGPT-Instanz: `impl-odin-045-chatgpt`.

**Gesendeter Kontext:** EventRecordEntity.java, EventRecordHashChainIntegrationTest.java, EventRecordEntityTest.java, V026__add_hash_chain_columns.sql inkl. Beschreibung der 8 bestehenden Integrationstests und 5 Unit-Tests.

**Fragen:** (1) Fehlende Edge Cases und Grenzwerte, (2) Inkonsistenter Hash-Zustand, Partial-Index-Semantik, Format-Validierung, (3) Migrations-Edge-Cases, (4) JPA-Mapping-Pitfalls.

### ChatGPT-Findings und Bewertung

| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| DB-Laengenrestriktivitaet: VARCHAR(64) erzwingt Ablehnung bei 65 Zeichen | Valide — wichtiger Grenzwert | Test `save_currentHash_length65_throwsDataIntegrityViolation` implementiert |
| Gleiches fuer previousHash | Valide | Test `save_previousHash_length65_throwsDataIntegrityViolation` implementiert |
| Partial-Index-Pradikatstest: existdef muss IS NOT NULL enthalten | Valide — Index-Existenz allein nicht ausreichend | Test `indexDefinition_currentHash_containsPartialPredicate` implementiert |
| Inkonsistenter Hash-Zustand (previous NOT NULL + current NULL): Dokumentation des permissiven Schemas | Valide — wichtige Boundary-Dokumentation | Test `save_inconsistentHashState_previousNotNull_currentNull_persists` implementiert (dokumentiert aktuelles Verhalten, CHECK constraint ist ODIN-031 Scope) |
| Schema-Metadaten-Test (VARCHAR 64, nullable) | Valide — verhindert stille Schema-Drifts | Test `schema_hashColumns_areVarchar64_nullable` implementiert |
| Empty-String-Semantik (leerer String = nicht NULL, wird indexiert) | Valide, aber Policy-Entscheidung fuer ODIN-031 | Offen gelassen — hash-chain-Logik ist ODIN-031 Scope |
| Format-Validierung (Hex-Chars, exakte 64-Laenge per Bean Validation) | Valide fuer spaeter, aber Scope von ODIN-031 | Verworfen fuer V026 — Schema-Migration haelt sich minimal |
| CREATE INDEX CONCURRENTLY fuer Produktions-Safety | Valide operativer Hinweis | Als offener Punkt dokumentiert (Flyway-Einschraenkung bei Concurrent Indexes) |
| Immutabilitaets-Enforcement via updatable=false | Valide Architekturempfehlung | Vorab durch Gemini-Review Round 2 bereits implementiert |

**Resultat:** 5 neue Integrationstests implementiert (jetzt 13 gesamt). Style-Fix (assertTrue statt assertEquals(true)).

## Gemini-Review (3 Dimensionen)

### Round 1 (initiale Implementierung)

#### Dimension 1: Code-Qualitaet
- Ergebnis: Positiv. Korrekte Typisierung, JavaDoc, HASH_HEX_LENGTH-Konstante bestaetigt.
- Keine Findings.

#### Dimension 2: Architektur-Konformitaet
- Ergebnis: Positiv. DDD-Modulschnitt korrekt (Entity in odin-audit), nullable Spalten fuer Backward-Compatibility, Partial Index bestaetigt.
- Keine Findings.

#### Dimension 3: Praxis-Gaps (Round 1)

| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| Hash-Chain Race Condition (fehlende UQ auf run_id + instrument_id + sequence_number) | Verworfen | Der UQ-Constraint wurde bewusst in V024 entfernt (zu restriktiv fuer OMS-Events). Pipelines sind single-threaded — kein Concurrency-Problem. |
| Index auf previous_hash fuer Rueckwaerts-Traversierung fehlt | Notiert als offener Punkt | Chain-Validierung ist eine seltene administrative Operation. Index wird bei Bedarf spaeter ergaenzt. |

### Round 2 (Remediation — nach Implementierung der neuen Tests)

#### Dimension 1: Code-Qualitaet (Round 2)
- **FINDING: Fehlendes `updatable = false` auf Hash-Spalten** — VALIDE, BEHOBEN. Hash-Felder sind jetzt `@Column(updatable = false)` — konsistent mit dem Append-Only-Design des Systems.
- Style-Finding: `assertEquals(true, ...)` statt `assertTrue(...)` — BEHOBEN.

#### Dimension 2: Konzepttreue (Round 2)
- Ergebnis: Positiv nach Behebung. Partial Index Strategy bestaetigt. Backward-Compatibility korrekt.
- Index auf `previous_hash` fuer Vorwaerts-Traversierung fehlt — als offener Punkt dokumentiert (seltene administrative Operation).

#### Dimension 3: Praxis-Gaps (Round 2)

| Finding | Bewertung | Aktion |
|---------|-----------|--------|
| `CREATE INDEX CONCURRENTLY` fuer Produktions-Safety (ShareLock blockiert INSERT/UPDATE) | Valide operativer Hinweis | Als offener Punkt dokumentiert. Flyway unterstuetzt CONCURRENTLY nicht nativ in Transaktionen — separates Deployment-Dokument. |
| CHECK-Constraint fuer inkonsistente Hash-Zustaende | Verworfen fuer V026 | Scope von ODIN-031. Schema-Migration haelt sich minimal. Test dokumentiert aktuelles permissives Verhalten. |
| Hash-Chain-Scope: Global oder pro Pipeline (run_id)? Composite Index sinnvoll? | Design-Frage fuer ODIN-031 | Als offener Punkt dokumentiert. |

## Test-Ergebnisse

- **Unit-Tests:** 12/12 grueen (EventRecordEntityTest, inkl. 5 neue Hash-Tests)
- **Integration-Tests:** 13/13 grueen (EventRecordHashChainIntegrationTest) — nach Failsafe `mvn verify`
- **Regressions-Tests:** 39/39 odin-audit Unit-Tests grueen

## Test-Abdeckung (Integration — 13 Tests)

| Test | Aspekt |
|------|--------|
| flywayMigration_v026_runsSuccessfully | V026 Migration laeuft fehlerfrei |
| save_withoutHashes_legacyBehavior | Records ohne Hashes (Legacy) persistieren korrekt |
| save_withHashes_persistsCorrectly | Records mit beiden Hash-Werten persistieren korrekt |
| existingRecords_remainReadableAfterMigration | Bestehende Records bleiben nach Migration lesbar |
| hashChain_firstRecordHasNullPreviousHash | Erster Record: previous_hash = NULL, current_hash gesetzt |
| hashChain_subsequentRecordsHaveBothHashes | Folge-Records: previous_hash = vorheriger current_hash |
| mixedRecords_hashedAndLegacy_coexist | Legacy und neue Records koexistieren |
| indexExists_currentHash | Partial Index `idx_event_record_current_hash` existiert |
| save_currentHash_length65_throwsDataIntegrityViolation | DB lehnt VARCHAR(64)-Ueberschreitung ab (currentHash) |
| save_previousHash_length65_throwsDataIntegrityViolation | DB lehnt VARCHAR(64)-Ueberschreitung ab (previousHash) |
| indexDefinition_currentHash_containsPartialPredicate | Index-Definition enthaelt IS NOT NULL Praedikat |
| save_inconsistentHashState_previousNotNull_currentNull_persists | Permissives Schema: inkonsistenter Hash-Zustand erlaubt (ODIN-031 Scope) |
| schema_hashColumns_areVarchar64_nullable | Schema-Metadaten: VARCHAR(64), nullable fuer beide Hash-Spalten |

## Offene Punkte

- Hash-Berechnung und -Verkettungslogik ist Scope von ODIN-031
- Index auf `previous_hash` fuer Vorwaerts-Traversierung — bei Bedarf spaeter ergaenzen
- Retroaktives Hash-Backfill fuer bestehende Records ist Out-of-Scope (gemaess Story-Definition)
- `CREATE INDEX CONCURRENTLY`: Fuer grosse Produktions-DBs empfohlen — muss ausserhalb einer Flyway-Transaktion laufen (separates Deployment-Skript oder Flyway-Konfiguration mit `executeInTransaction=false`)
- CHECK-Constraint fuer Hash-Zustandskonsistenz (previous NOT NULL => current NOT NULL) — Scope ODIN-031
- Hash-Chain-Scope-Definition (global vs. pro run_id) und entsprechende composite Index-Strategie — Design-Entscheidung fuer ODIN-031
