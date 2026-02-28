# ODIN-031: Audit Log Hash-Chain Integrity

**Modul:** odin-audit
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine
**Geschaetzter Umfang:** M

---

## Kontext

Das Konzept fordert tamper-proof Audit Logs mit Hash-Chain (jeder Eintrag enthaelt den Hash des vorherigen Eintrags, Blockchain-Prinzip). Die aktuelle `PostgresEventLog` speichert Events ohne Integritaetssicherung. Die Hash-Chain stellt sicher, dass nachtraegliche Manipulation erkannt wird.

## Scope

**In Scope:**
- Hash-Chain-Logik in `PostgresEventLog` oder `AuditEventDrainer`
- SHA-256 Hash ueber: vorheriger Hash + Timestamp + EventType + Payload
- Hash wird in `EventRecordEntity` gespeichert (neues Feld `previousHash`, `currentHash`)
- Validierungsmethode: Hash-Chain verifizieren (Konsistenz-Check)
- Erster Eintrag: `previousHash = "GENESIS"`
- Flyway-Migration fuer neue Felder

**Out of Scope:**
- Externe Kopie auf separaten Storage (separater Operations-Task)
- Taeglich signierte Exportdateien
- Kryptografische Signierung mit privatem Schluessel (nur Hash-Chain)

## Akzeptanzkriterien

- [ ] Jeder EventRecord enthaelt `currentHash` (SHA-256) und `previousHash`
- [ ] Hash-Berechnung: `SHA-256(previousHash + "|" + timestamp + "|" + eventType + "|" + payload)`
- [ ] Erster Eintrag des Tages: `previousHash = "GENESIS"`
- [ ] Validierungsmethode `verifyHashChain(runId)`: Prueft alle Events eines Runs auf Konsistenz
- [ ] Bei Hash-Inkonsistenz: ALERT mit betroffenem Zeitraum
- [ ] Flyway-Migration fuer neue Felder vorhanden und laeuft fehlerfrei
- [ ] Performance: Hash-Berechnung darf den Event-Durchsatz nicht merklich beeintraechtigen (< 1ms Overhead pro Event)

## Technische Details

**Dateien:**
- `odin-audit/src/main/java/de/its/odin/audit/service/PostgresEventLog.java` (Erweiterung)
- `odin-audit/src/main/java/de/its/odin/audit/service/HashChainValidator.java` (neue Klasse)
- `odin-audit/src/main/java/de/its/odin/audit/entity/EventRecordEntity.java` (neue Felder)
- Flyway-Migration: `V025__add_hash_chain_to_event_record.sql`

## Konzept-Referenzen

- `docs/concept/06-risk-management.md` — Abschnitt 7.3 "Audit-Logs (Tamper-Proof)"
- `docs/concept/10-observability.md` — Abschnitt 4 "Audit Trail"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — "DDD-Modulschnitt: Persistenz"
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln (R1-R13), Konfiguration (CSpec)

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-audit`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `GENESIS_HASH`, `HASH_DELIMITER`)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.audit.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `HashChainValidator`: korrekter Chain, manipulierter Chain, GENESIS-Eintrag
- [ ] Unit-Tests fuer Hash-Berechnung: Formel und Delimitierung korrekt
- [ ] Unit-Tests fuer Sequenzierung: Thread-Safety-Verhalten des AuditEventDrainers geprueft
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest mit realen Klassen: `PostgresEventLog` schreibt mehrere Events, `HashChainValidator` prueft die Chain End-to-End
- [ ] Integrationstest: Manipulation eines Events wird zuverlassig erkannt
- [ ] Integrationstest: GENESIS-Eintrag korrekt als Startpunkt behandelt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass HashChainValidator und PostgresEventLog korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

- [ ] Embedded-Postgres-Tests mit **Zonky** (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migration `V025__add_hash_chain_to_event_record.sql` wird im Test ausgefuehrt
- [ ] `EventRecordEntity` mit neuen Feldern (`currentHash`, `previousHash`) wird gegen echte DB getestet
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

**Referenz:** CLAUDE.md → Tests (`*IntegrationTest` fuer Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `HashChainValidator.java`, `PostgresEventLog.java` (Erweiterungen), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei leerem Run? Was bei Hash-Kollision? Concurrent-Write-Szenario? Performance bei grossen Audit-Logs?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Race Conditions (insbesondere bei previousHash-Sequenzierung), Null-Safety, Ressource-Leaks, Performance-Probleme"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/06-risk-management.md` Abschnitt 7.3 + `docs/concept/10-observability.md` Abschnitt 4 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Hash-Chain-Implementierung dem Konzept entspricht. Stimmen Hash-Formel, GENESIS-Konvention, Validierungslogik und Alert-Verhalten mit den Akzeptanzkriterien ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es bei Hash-Chain-Implementierungen in Produktionssystemen typische Probleme, die hier nicht behandelt sind? Welche realen Szenarien (z.B. Datenbankausfall mitten in der Chain, Replay-Attacken, Chain-Reset-Angriffe) koennten auftreten?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Wahl des Hash-Delimiters, Thread-Safety-Ansatz)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- Hash-Chain ist append-only: Einmal geschrieben, nie geaendert
- Thread-Safety: `previousHash` muss beim Draining korrekt sequenziert werden (AuditEventDrainer ist single-threaded — diesen Vorteil nutzen, aber in Tests absichern)
- Die externe Kopie (taeglich signierte Kopie auf separaten Storage) ist ein separater Operations-Task, nicht Teil dieser Story
- SHA-256 aus JDK (`MessageDigest.getInstance("SHA-256")`) — keine externe Abhaengigkeit noetig
