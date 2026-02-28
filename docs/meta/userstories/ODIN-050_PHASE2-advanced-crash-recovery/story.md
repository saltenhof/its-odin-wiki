# ODIN-050 [PHASE2]: Advanced Crash Recovery

**Modul:** odin-core, odin-app
**Phase:** 2 -- Nicht fuer V1 vorgesehen. Umsetzung erst nach explizitem Stakeholder-Beschluss.
**Groesse:** L
**Abhaengigkeiten:** Keine

---

## Kontext

V1 ist NICHT intraday-restartable. GTC-Stops beim Broker schuetzen Positionen bei Crash. Phase 2 soll Intraday-Recovery ermoeglichen: Zustand aus Datenbank rekonstruieren, Broker-Positionen abgleichen, und Trading fortsetzen.

## Scope

**In Scope:**
- State-Reconstruction aus DB (Pipeline-State, Position-State, OMS-State)
- Broker-Reconciliation: Positionen und Orders mit Broker abgleichen
- Orphan-Detection: Broker-Positionen ohne lokalen State
- Graceful Resume: Trading nach erfolgreicher Reconciliation fortsetzen
- Safe-Mode als Fallback wenn Reconciliation fehlschlaegt

**Out of Scope:**
- Automatischer Neustart ohne manuelle Intervention (immer User-Bestaetigung noetig)
- Cross-Day-Recovery (nur intraday, EOD-Flat bleibt V1-Regel)
- Recovery nach Broker-Crash (nur nach ODIN-Crash)

## Akzeptanzkriterien

- [ ] Startup-Reconciliation: ODIN prueft beim Start ob offene Positionen beim Broker existieren
- [ ] State-Reconstruction: Pipeline-State, Position-Tranchen, OMS-Status werden aus DB wiederhergestellt
- [ ] Broker-Reconciliation: Lokaler State wird mit Broker-State abgeglichen (Qty, AvgPrice, Stops)
- [ ] Orphan-Detection: Broker-Positionen ohne lokalen DB-Eintrag werden erkannt und gemeldet
- [ ] Graceful Resume: Bei erfolgreicher Reconciliation kann Trading fortgesetzt werden (nach Benutzer-Bestaetigung)
- [ ] Safe-Mode-Fallback: Bei Reconciliation-Fehler wird Safe-Mode aktiviert (kein neues Trading, nur Exit)
- [ ] Vollstaendige Audit-Log-Entries fuer alle Recovery-Aktionen
- [ ] Recovery laeuft in begrenzter Zeit (max. 30 Sekunden konfigurierbar)

## Technische Details

**Betroffene Klassen (Schaetzung):**
- `odin-core`: RecoveryOrchestrator, ReconciliationService, StateReconstructor
- `odin-app`: Startup-Lifecycle-Erweiterung (RecoveryCheck vor Trading-Start)
- `odin-api`: RecoveryResult-DTO, ReconciliationStatus-ENUM
- `odin-execution`: PositionReconstructor (aus OMS-Entity-Daten)
- Konfiguration: `odin.core.recovery.{property}`

## Konzept-Referenzen

- `docs/concept/06-risk-management.md` -- Abschnitt 5 "Crash-Recovery (Normativ)"
- `docs/concept/06-risk-management.md` -- Abschnitt 5.2 "Automatischer Neustart"
- `docs/concept/06-risk-management.md` -- Abschnitt 5.3 "Naechster regulaerer Tagesstart"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Modulabhaengigkeiten (odin-core orchestriert)
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints, HIGH-Risiko-Einordnung
- `T:/codebase/its_odin/CLAUDE.md` -- MarketClock-Pflicht, EventLog-Pflicht, Coding-Regeln

## Definition of Done

> **Hinweis Phase 2:** Diese Story wird erst umgesetzt nach explizitem Stakeholder-Beschluss. Die nachfolgende DoD gilt dann vollstaendig und ohne Ausnahme.

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-core,odin-app,odin-execution`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- Konstante fuer Max-Recovery-Timeout, Retry-Limits
- [ ] Records fuer DTOs (RecoveryResult, ReconciliationStatus)
- [ ] ENUM fuer ReconciliationStatus (SUCCESS, PARTIAL, FAILED, ORPHAN_DETECTED)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace: `odin.core.recovery.{property}`
- [ ] MarketClock verwenden: kein `Instant.now()` im Trading-Codepfad
- [ ] Alle Recovery-Aktionen werden in EventLog protokolliert (keine stillen Fehler)
- [ ] Safe-Mode-Fallback: Implementierung setzt Safe-Mode via GlobalRiskManager (nicht direkt)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer ReconciliationService: Match/Mismatch-Szenarien (Qty stimmt, Qty weicht ab, kein lokaler State)
- [ ] Unit-Tests fuer StateReconstructor: Rekonstruktion aus DB-Entities
- [ ] Unit-Tests fuer Orphan-Detection: Broker-Position ohne DB-State korrekt erkannt
- [ ] Unit-Tests fuer RecoveryOrchestrator-Zustandsmaschine (Recovery-Erfolg, Recovery-Fehlschlag → Safe-Mode)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet
- [ ] RecoveryOrchestrator + ReconciliationService + StateReconstructor Integration: Vollstaendiger Recovery-Durchlauf mit gemocktem BrokerGateway
- [ ] Startup-Integration: Recovery laeuft vor Trading-Start, korrekte Reihenfolge im Lifecycle
- [ ] Safe-Mode-Fallback-Integration: Reconciliation schlaegt fehl → Safe-Mode wird korrekt gesetzt
- [ ] Mindestens 1 Integrationstest fuer Erfolgs- und Fehlschlag-Szenario

### 2.4 Tests -- Datenbank

- [ ] Embedded-Postgres-Tests mit Zonky fuer State-Reconstruction aus DB
- [ ] StateReconstructor liest Pipeline-State, Position, OMS-State korrekt aus Embedded-DB
- [ ] Flyway-Migrationen werden im Test ausgefuehrt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Implementierungscode, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. Crash genau beim Order-Fill, Partial-Fill vor Crash, Broker-Connection nicht verfuegbar beim Recovery, mehrere Pipelines gleichzeitig in Recovery, Broker-State stimmt mit DB ueberein aber Timestamps weichen ab)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben: "Pruefe auf Bugs, Race Conditions beim gleichzeitigen Recovery mehrerer Pipelines, Deadlock-Risiken, fehlerhafte Rollback-Logik, unvollstaendige Exception-Behandlung, Null-Safety bei fehlenden DB-Eintraegen"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/06-risk-management.md` Abschnitt 5 an Gemini
- [ ] Auftrag: "Pruefe ob Implementierung dem Crash-Recovery-Konzept entspricht -- insb. Reconciliation-Schritte gemaess Abschnitt 5.2, Safe-Mode-Fallback, Audit-Pflicht fuer alle Recovery-Aktionen"
- [ ] Abweichungen bewertet

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es Themen die im Konzept nicht behandelt sind? Z.B. Was passiert bei Netzwerk-Split (ODIN laeuft, Broker-Connection unterbrochen)? Wie geht man mit GTC-Stops um, die der Broker behaelt aber ODIN vergessen hat? Timing-Fenster fuer Recovery (laeuft Recovery noch wenn Market Open ist)?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State live aktualisiert waehrend der Arbeit
- [ ] Design-Entscheidungen dokumentiert (z.B. ob automatischer oder manuell-bestaetiger Resume)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- HIGH-Risiko-Story: Recovery-Logik ist fehleranfaellig und kann bei Fehler Positionen unkontrolliert hinterlassen -- sehr sorgfaeltig testen
- Safe-Mode ist immer der konservative Fallback -- im Zweifel Safe-Mode aktivieren, nicht Trading fortsetzen
- Benutzer-Bestaetigung vor Resume: ODIN darf nach Recovery NICHT automatisch weitertraden ohne explizite Bestaetigung (Sicherheitsprinzip)
- Recovery-Timeout: Max. 30 Sekunden konfigurierbar -- danach Safe-Mode, kein endloses Warten auf Broker
- GTC-Stops: V1 setzt GTC-Stops als Schutz -- bei Recovery sicherstellen dass GTC-Stops NICHT doppelt gesetzt werden
- Aus MEMORY.md: V1 ist NICHT intraday-restartable -- das ist ein bewusster Entscheid. Diese Story hebt das fuer Phase 2 auf.
