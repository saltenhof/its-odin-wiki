# ODIN — Guardrail: Entwicklungsprozess (Agent Development Guide)

Version: 1.2
Stand: 2026-02-15
Review: ChatGPT R1. DDD-Modulschnitt: Referenzen aktualisiert (odin-persistence → Domaenen-Module, odin-audit)

---

## 1. Zweck

Dieser Guardrail definiert den verbindlichen Entwicklungsprozess fuer den KI-Agenten (Claude Code), wenn er Code fuer das ODIN-Projekt schreibt. Er beschreibt Phasen, Checkpoints, Entscheidungsmatrizen und Eskalationsregeln.

**Zielgruppe:** Der KI-Agent selbst — dieses Dokument wird bei jedem Coding-Auftrag konsultiert.

**Geltungsbereich:** Alle 8 Java-Module (odin-api bis odin-app). odin-frontend (React) hat eigene Konventionen und ist von den Java-spezifischen Regeln (R5, R6, R7, CSpec) ausgenommen.

**Referenzen:**
- Regelkatalog R1–R13 + CSpec (`.claude/instructions`)
- Modul-Struktur-Guardrail (`guardrail-module-structure.md`)
- Kapitel 0: Systemuebersicht, Kapitel 1: Modularchitektur

**Definition of Done:** Compile ok, erforderliche Tests gruen, Review-Regeln erfuellt, Doku falls noetig, Ergebnisbericht an Nutzer.

---

## 2. Phasenuebersicht

```
Phase 0: Auftragsanalyse ──> Phase 1: Design ──> Phase 2: Implementierung ──> Phase 3: Test-Design & -Implementierung ──> Phase 4: Build-Gates & Review ──> Phase 5: Abschluss
                │                    │                                                                                            │
                ▼                    ▼                                                                                            ▼
          Harter Stop          Human-in-the-Loop                                                                           ChatGPT-Review
          bei fehlenden        bei Breaking Changes                                                                        bei komplexer Logik
          Informationen        oder HOCH-Risiko
```

**Abgrenzung Phase 3 vs. Phase 4:**
- **Phase 3** = Tests entwerfen und schreiben (Test-Artefakte erzeugen)
- **Phase 4** = Tests und Build ausfuehren, Ergebnisse pruefen, Reviews durchfuehren (Gates)

---

## 3. Phase 0: Auftragsanalyse

### 3.1 Auftragsklassifikation

Bestimme den Auftragstyp:

| Typ | Beispiel | Typische Komplexitaet |
|-----|---------|----------------------|
| Feature | Neuer KPI-Indikator, neue Exit-Regel | Mittel–Hoch |
| Bugfix | Falscher Stop-Preis, fehlende Null-Pruefung | Niedrig–Mittel |
| Refactoring | Package-Umbau, Interface-Aenderung | Mittel–Hoch |
| Konfiguration | Neuer Property-Key, Namespace-Erweiterung | Niedrig |
| Infrastruktur | Maven-POM-Aenderung, Dependency-Update | Niedrig–Mittel |

### 3.2 Machbarkeitspruefung (R9 — PFLICHT)

Vor jeder weiteren Arbeit:

1. **Sind alle noetige Informationen vorhanden?** Fehlende Spezifikation → Harter Stop, nachfragen (R9.1)
2. **Liegt der betroffene Bestandscode vor?** Wenn nicht → Harter Stop, anfordern (R9.3)
3. **Ist der Auftrag innerhalb des ODIN-Architekturrahmens umsetzbar?** Wenn nicht → Stop mit Begruendung

### 3.3 Scope-Analyse

Bestimme die betroffenen Module:

- Welche Module werden geaendert?
- Werden public APIs betroffen?
- Gibt es Cross-Modul-Abhaengigkeiten?
- Werden neue Module angelegt?

### 3.4 Risikoeinstufung

| Risikostufe | Kriterium | Konsequenz |
|-------------|----------|------------|
| **HOCH** | Trading-kritischer Code (OMS, Broker, Risk), public API Aenderung, neues Modul, Cross-Modul-Aenderung | Human-in-the-Loop PFLICHT |
| **MITTEL** | Neue Geschaeftslogik innerhalb eines Moduls, neue Services | Agent darf innerhalb Leitplanken entscheiden, Review bei Bedarf |
| **NIEDRIG** | Bugfix, Konfiguration, interne Refactorings ohne API-Aenderung | Agent arbeitet autonom |

**Uebersteuerungsregel:** Risikostufe HOCH aus Phase 0 uebersteuert alle nachfolgenden Entscheidungsmatrizen. Wenn Phase 0 HOCH ergibt, ist Human-in-the-Loop PFLICHT — unabhaengig davon ob die Aenderung nach den Kriterien in 4.3 als "kein Human noetig" erscheint.

---

## 4. Phase 1: Design

### 4.1 Entscheidung: Design noetig oder direkt coden?

| Situation | Aktion |
|-----------|--------|
| Neuer Service/Engine/Port-Implementierung | Kurze Architekturentscheidung dokumentieren (R4.1) |
| Neues Modul | Architekturentscheidung + Modul-Struktur-Guardrail Checkliste |
| Bugfix in bestehender Logik | Kein Design noetig — direkt Phase 2 |
| Property hinzufuegen | Kein Design noetig — CSpec einhalten, direkt Phase 2 |
| Interface-Aenderung | Design + Human-Review |
| Internes Refactoring (private Methoden, lokale Variablen) | Kein Design noetig — direkt Phase 2 |

### 4.2 Architektur-Checkliste (R4.2)

Wenn Design noetig, pruefe:

- [ ] Modulzustaendigkeit klar (welches Modul ist verantwortlich)?
- [ ] Separation of Concerns eingehalten?
- [ ] Abhaengigkeitsrichtung korrekt (nur nach oben zu odin-api)?
- [ ] Singleton vs. Pro-Pipeline korrekt gewaehlt?
- [ ] Kein Verstoss gegen Dependency-Regeln (Kap 1, Abschnitt 6)?
- [ ] Bei Port-Implementierung: Live + Sim Varianten beruecksichtigt?

### 4.3 Human-in-the-Loop Entscheidungsmatrix

| Aenderung | Human noetig? | Begruendung |
|-----------|--------------|-------------|
| Public API Aenderung (odin-api Ports, DTOs, Events, Enums) | **JA, IMMER** | Betrifft alle konsumierenden Module (R3.2) |
| Neues Maven-Modul | **JA, IMMER** | Architekturelle Grundsatzentscheidung |
| Cross-Modul-Refactoring | **JA, IMMER** | Abhaengigkeitsgraph betroffen |
| Neuer Port oder Port-Aenderung | **JA, IMMER** | Vertrag zwischen Modulen |
| Trading-kritischer Code (OMS, RiskGate, BrokerGateway, Execution-Logik) | **JA, IMMER** | Fehler kosten echtes Geld — auch ohne API-Aenderung (HOCH-Uebersteuerung) |
| Greenfield innerhalb Leitplanken (neuer Service, neue Engine) | **NEIN** | Agent darf entscheiden (R3.2 Greenfield) |
| Modul-interner Refactoring | **NEIN** | Keine externen Auswirkungen |
| Neuer Property-Key | **NEIN** | CSpec regelt alles |
| Bugfix ohne API-Aenderung | **NEIN** | Scope begrenzt |

### 4.4 ChatGPT als Design-Review-Partner

ChatGPT-Review **einsetzen** bei:
- Komplexen Architekturentscheidungen mit Trade-offs
- Neuen Architektur-Patterns die es im ODIN-Projekt noch nicht gibt
- Sicherheitsrelevanten Design-Entscheidungen (Broker-Anbindung, Order-Logik)

ChatGPT-Review **NICHT** bei:
- Standard-CRUD-Services
- Property-Konfiguration
- Triviale Bugfixes
- Implementierungen die einem etablierten Pattern folgen

---

## 5. Phase 2: Implementierung

### 5.1 Guardrails einhalten

Vor dem Schreiben von Code sicherstellen:

**Modul-Struktur (guardrail-module-structure.md):**
- Package im richtigen Modul-Root (`de.its.odin.{modul}`)
- Sub-Package korrekt (config/, model/, service/, live/, sim/, ib/)
- Namenskonventionen (Abschnitt 3 des Modul-Guardrails)
- Singleton/Pro-Pipeline-Zuordnung beachten

**Code-Qualitaet (R5, R6):**
- Kein `var` (R5.4)
- Keine Magic Numbers — `private static final` Konstanten (R6.1)
- Keine Reflection (R6.8)
- Kein explizites Boxing/Unboxing (R5.8)
- Records fuer DTOs, Switch-Expressions, `.toList()` (R6.3)
- `List.getFirst()` statt `List.get(0)` (R6.4)
- Sprechende Variablennamen, Laenge > 1 Zeichen (R6.7)
- ENUM statt String fuer endliche Mengen (R6.2, R11.5)
- Methoden: Verb-first Benennung (R11.4)
- Einheitliche Namenskonventionen — kein Mix aus `*Impl` und `*Adapter` (R11.3)

**Dokumentation (R7):**
- JavaDoc auf allen public Klassen/Methoden/Attributen — kompakt aber vollstaendig
- Inline-Kommentare (englisch) max. 10% des Methodenrumpfs, nur bei komplexer Logik

**Konfiguration (CSpec, R10, R13):**
- Defaults in `.properties`, nicht in Java
- Namespace: `odin.{modul}.{komponente}.{property}`
- `@ConfigurationProperties` als Record + `@Validated`
- Secrets nie in Properties-Dateien
- Zeitwerte mit Suffix (`-ms`, `-s`), Booleans mit `enable-`/`is-` Prefix

### 5.2 ODIN-spezifische Regeln

- **MarketClock verwenden:** Kein `Instant.now()` oder `System.currentTimeMillis()` im Trading-Codepfad — ausschliesslich `MarketClock.now()`
- **Port-Abstraktion:** Nie gegen konkrete Implementierungen programmieren, immer gegen Port-Interfaces aus `de.its.odin.api.port`
- **Keine Fachmodul-Querverweise:** brain darf nicht broker importieren, execution darf nicht data importieren — nur odin-api
- **Pro-Pipeline-Module:** Keine Spring-Annotations (`@Component`, `@Service`, `@Repository`) in odin-data, odin-brain, odin-execution
- **Events sind immutable:** Nie mutieren, bei Aenderung neues Event erzeugen
- **Idempotenz:** Jeder OrderIntent braucht eine `clientOrderId`

### 5.3 Observability bei Trading-kritischem Code

Bei Aenderungen an OMS, RiskGate, BrokerGateway oder Execution-Logik ist minimale Observability PFLICHT:

- **Structured Logging** fuer jeden Order-Lifecycle-Schritt (Submit, Ack, Fill, Cancel, Reject) mit `clientOrderId` als Correlation-Key
- **Risk-Entscheidungen loggen** (RiskGate Pass/Reject mit Begruendung)
- **State-Transitions loggen** (PipelineState, Kill-Switch-Trigger)
- **Latenz-relevante Stellen** mit Timing-Marker versehen (Decision-Cycle-Dauer, Broker-Roundtrip)

Kein Metrics-Framework (Micrometer) proaktiv einbauen — das kommt spaeter. Structured Logs genuegen fuer v1.

### 5.4 Lean-Prinzip bei der Implementierung

- Keine proaktiven Performance-Optimierungen (kein Caching, kein Object-Pooling) ohne bewiesenes Problem (R11.2)
- Keine Abstraktionsschichten ohne Nutzerauftrag (R11.1)
- Keine Kompatibilitaetsschichten oder Deprecated Code (R3.5)
- Keine eigenen Annahmen ins Design einfliessen lassen (R3.6)

---

## 6. Phase 3: Test-Design & Test-Implementierung

Phase 3 definiert welche Tests geschrieben werden und erzeugt die Test-Artefakte. Die Ausfuehrung erfolgt in Phase 4.

### 6.1 Test-Entscheidungsmatrix

| Was wurde geaendert? | Welcher Test? | Pflicht? |
|----------------------|--------------|----------|
| Neue Geschaeftslogik (Service, Engine, Gate) | Unit-Test | **PFLICHT** |
| Neue Berechnungslogik (KPI, Scoring, Sizing) | Unit-Test mit Randfaellen | **PFLICHT** |
| Neuer Repository/DB-Zugriff | Integrationstest (Testcontainer/H2) | **PFLICHT** |
| Neuer Properties-Record | Binding-Test | **PFLICHT** |
| Broker/API-Anbindung (Live-Adapter) | Zweistufig (siehe 6.4) | **PFLICHT** |
| DTO/Record/Enum (nur Datentyp, keine Logik) | Kein Test | — |
| @Configuration / Bean-Wiring | Context-Load Smoke-Test empfohlen | Optional (empfohlen bei neuen Beans/Profilen) |
| Property-Key hinzugefuegt (kein neuer Record) | Kein separater Test | — |
| Bugfix | Test der den Bug reproduziert | **PFLICHT** |
| Internes Refactoring (gleiche Semantik) | Bestehende Tests muessen gruen bleiben | **PFLICHT** |

### 6.2 Naming-Konventionen

| Test-Typ | Suffix | Beispiel | Maven-Phase |
|----------|--------|---------|-------------|
| Unit-Test | `*Test` | `KpiEngineTest` | Surefire |
| Integrationstest | `*IntegrationTest` | `BrokerPropertiesIntegrationTest` | Failsafe |
| Ende-zu-Ende-Test | `*E2ETest` | `TradingPipelineE2ETest` | Failsafe |

### 6.3 Test-Qualitaet

- Tests muessen deterministische Ergebnisse liefern (keine Abhaengigkeit von Systemzeit, Netzwerk, Zufallswerten)
- Pro-Pipeline-Komponenten testen: Plain Java, keine Spring-Kontexte noetig
- Mocks/Stubs fuer Port-Interfaces verwenden
- Test-Properties in `odin-{modul}-test.properties`
- Tests gegen externe APIs (LLM, TWS) per Maven-Profil oder JUnit-Tag deaktivierbar

### 6.4 Broker/API-Anbindung: Zweistufiges Test-Modell

| Stufe | Test-Typ | Pflicht | Trigger |
|-------|---------|---------|---------|
| **Stufe 1: Contract/Adapter-Test** | Unit-/Integrationstest gegen Mock/Simulator/Recorded Sessions | **IMMER PFLICHT** | Jede Aenderung an Broker- oder API-Adaptern |
| **Stufe 2: E2E gegen echte API** | E2E-Test (Tag/Profil-gesteuert) | **PFLICHT vor Release** | Nur bei HOCH-Risiko-Aenderungen oder vor Produktivsetzung |

Stufe 1 blockiert nicht den Entwicklungsfluss. Stufe 2 ist fuer den Lean-Prozess nur bei hohem Risiko oder vor Release erforderlich.

### 6.5 Concurrency-Tests: Empfohlene Patterns

Wenn Concurrency-relevanter Code getestet wird:

- **Deterministischer Scheduler:** Tests kontrollieren Thread-Reihenfolge explizit (CountDownLatch, CyclicBarrier), keine `Thread.sleep()`-basierte Synchronisation
- **Race-Reproduktion:** Kritische Abschnitte gezielt mit parallelen Threads belasten
- **Timeout-Guards:** Jeder Concurrency-Test hat einen harten Timeout (JUnit `@Timeout`), damit Deadlocks den Build nicht blockieren
- **Scope begrenzen:** Concurrency-Tests nur fuer shared-mutable-state Komponenten (GlobalRiskManager, IbDispatcher, EventLog-Writer), nicht fuer Pro-Pipeline-Logik (die ist single-threaded)

---

## 7. Phase 4: Build-Gates & Review

Phase 4 fuehrt die in Phase 3 erzeugten Tests aus und prueft Build-Stabilitaet und Review-Anforderungen.

### 7.1 Compile-Check (PFLICHT)

Nach jedem Implementierungsblock:

| Aenderungsscope | Compile-Befehl |
|----------------|---------------|
| Aenderungen in einem Modul (kein API) | `mvn compile -pl odin-{modul}` |
| Aenderungen an odin-api oder public API eines Moduls | `mvn compile` (Gesamtprojekt) |
| Neues Modul angelegt | `mvn compile` (Gesamtprojekt) |

### 7.2 Test-Gates (PFLICHT)

| Aenderungsscope | Unit-Tests | Integrationstests | E2E-Tests |
|----------------|-----------|-------------------|-----------|
| Modul-intern (kein API) | `mvn test -pl odin-{modul}` (ohne Integration/E2E) | Nur wenn DB/Repo betroffen | Nein |
| API-Aenderung / Cross-Modul | `mvn test` (Gesamtprojekt, ohne Integration/E2E) | Wenn DB/Repo betroffen | Nein |
| Broker/OMS/Risk-Aenderung | Unit-Tests des Moduls | Adapter-Tests (Stufe 1) PFLICHT | Stufe 2 bei HOCH-Risiko |
| Neues Modul / Cross-Modul | Unit-Tests Gesamtprojekt | Context-Load Smoke-Test empfohlen | Nein |

Integrations-/E2E-Tests-Excludes: `-Dtest.excludes="**/*IntegrationTest,**/*E2ETest"` fuer reine Unit-Test-Laeufe.

### 7.3 ChatGPT-Review als Verifikation

ChatGPT-Review **einsetzen** bei:

| Szenario | Begruendung |
|----------|-------------|
| Trading-kritischer Code (OMS, RiskGate, BrokerGateway-Impl) | Fehler kosten echtes Geld |
| Broker-Anbindung (IB TWS API Interaktion) | Komplexes asynchrones Protokoll, Fehlerbehandlung kritisch |
| P&L-/Risk-/Order-Entscheidungen (Signal→Order, Sizing, Risk-Limits) | Direkte finanzielle Auswirkung — Fehlerkosten hoch |
| State-Machine-Logik (PipelineStateMachine, Lifecycle) | Zustandsuebergaenge muessen vollstaendig und korrekt sein |
| Concurrency-relevanter Code (Dispatcher, Barriers, GlobalRisk) | Race Conditions schwer zu testen |

ChatGPT-Review **NICHT** bei:

| Szenario | Begruendung |
|----------|-------------|
| Standard-CRUD (Entity, Repository, simple Service) | Etablierte Patterns, geringes Risiko |
| Konfiguration (Properties, Binding) | CSpec regelt das, strukturell einfach |
| Triviale Bugfixes (Null-Check, Off-by-one) | Overhead uebersteigt Nutzen |
| Reine Datentypen (Records, Enums) | Kein Verhalten zum Reviewen |
| Boilerplate (Test-Setup, Fixtures) | Kein fachliches Risiko |
| KPI/Reporting ohne Rueckwirkung auf Entscheidungslogik | Kein finanzielles Risiko |

### 7.4 ChatGPT-Review Protokoll

Wenn ChatGPT-Review durchgefuehrt wird:

1. Zu reviewenden Code als Datei uebergeben (`file_path`)
2. Prompt mit Kontext: Was tut der Code, welches Modul, welche Risiken
3. Findings klassifizieren: KRITISCH / WICHTIG / HINWEIS / GO
4. KRITISCH und WICHTIG einarbeiten
5. HINWEIS bewerten — einarbeiten wenn sinnvoll, ablehnen wenn Over-Engineering
6. Ergebnis dem Nutzer berichten

---

## 8. Phase 5: Abschluss

### 8.1 Git-Operationen

- **git add** auf alle neuen Dateien (R14)
- **Commit:** Nur auf explizite Nutzeranweisung — nie eigenmaechtig committen
- **Kein Force-Push, kein Reset**

### 8.2 Dokumentations-Updates

| Aenderung | Dokumentation noetig? |
|-----------|----------------------|
| Neues Modul | Modul-Steckbrief in Kap 1, Checkliste durchlaufen |
| Neuer Port oder Port-Aenderung | Kap 0 (Port-Tabelle) und Kap 1 (Port-Alignment) |
| Neue Architekturentscheidung | WORKING-STATE.md aktualisieren |
| Abgeschlossener Meilenstein | PLAYBOOK.md aktualisieren |
| Wiki-relevante Aenderung | Wiki-Update nach Nutzerfreigabe |

### 8.3 Ergebnisbericht

Nach Abschluss eines Auftrags dem Nutzer berichten:
- Was wurde umgesetzt (Dateien, Module)
- Welche Designentscheidungen wurden getroffen
- Welche Tests geschrieben
- Offene Punkte oder Folgearbeiten

---

## 9. Besondere Szenarien

### 9.1 Trading-kritischer Code (OMS, Broker, Risk)

```
Auftrag → Phase 0 (HOCH) → Phase 1 (Design PFLICHT)
  → Human-Review des Designs
  → Phase 2 (Implementierung + Observability)
  → Phase 3 (Unit-Tests PFLICHT, Adapter-Tests PFLICHT, E2E bei Release)
  → Phase 4 (Compile + Tests + ChatGPT-Review PFLICHT)
  → Human-Review des Codes
  → Phase 5
```

**Besonderheiten:**
- Jede Order-Submission muss idempotent sein (clientOrderId)
- Stop-Orders muessen GTC sein (Crash-Recovery-Schutz)
- RiskGate darf nie umgangen werden
- BrokerGateway-Fehlerbehandlung: Circuit-Breaker-Pattern, Heartbeat-Loss → Kill-Switch
- Kein Silent-Fail — jeder Fehler muss eskaliert werden (Event, Log, Alert)

**Trading-Szenario-Checkliste** — bei Aenderungen an OMS/Broker/Risk muessen folgende Szenarien bedacht werden:

| Szenario | Betroffene Komponenten | Test-Abdeckung |
|----------|----------------------|----------------|
| Partial Fills / Multi-Fill | OMS (Durchschnittspreis, Restmenge, Gebuehren) | Unit-Test PFLICHT |
| Reject / Cancel-Reject / Replace-Reject | OMS, BrokerGateway | Unit-Test PFLICHT |
| Disconnect / Reconnect Broker | IbSession, IbDispatcher, OMS (Re-Sync) | Adapter-Test + manueller Test |
| Out-of-order Events / Duplicate Events | IbDispatcher, OMS (Idempotenz) | Unit-Test PFLICHT |
| Trading-Halt / Market closed / Early close | MarketClock, LifecycleManager | Unit-Test |
| Stop-Edgecases (Gap-Risk, Stop→Market vs StopLimit) | OMS, RiskGate | Unit-Test PFLICHT |
| RiskGate Fail-Fast vs Degraded Mode | RiskGate, GlobalRiskManager | Unit-Test PFLICHT |
| Circuit-Breaker Reset-Regeln | LLM Circuit-Breaker, Broker-Heartbeat | Unit-Test |
| Restart nach Crash (Safe-Mode) | LifecycleManager, Crash-Recovery | Integrationstest |

Diese Liste ist nicht abschliessend — sie dient als Mindest-Pruefung. Neue Szenarien werden bei Bedarf ergaenzt.

### 9.2 LLM-Integration (odin-brain)

```
Auftrag → Phase 0 → Phase 1 (Design fuer Prompt/Schema-Aenderungen)
  → Phase 2 (Implementierung)
  → Phase 3 (Unit-Tests fuer Parsing/Validation, Tag-geschuetzte E2E-Tests fuer Live-API)
  → Phase 4 (Compile + Tests)
  → ChatGPT-Review bei Prompt-Aenderungen oder neuem Schema
  → Phase 5
```

**Besonderheiten:**
- LLM-Output ist IMMER strukturiertes JSON — keine Freitext-Interpretation in Geschaeftslogik
- Schema-Validierung PFLICHT — ungueltige Responses werden verworfen, nicht geraten
- Whitelist fuer Features die in Entscheidungslogik einfliessen
- Token-Limits und Timeouts als Properties konfigurierbar
- Circuit-Breaker: 3 Failures → Quant-Only-Modus (keine neuen Positionen)

### 9.3 Datenmodell-Aenderungen (Domaenen-Module + odin-audit)

```
Auftrag → Phase 0 → Phase 1 (Schema-Design)
  → Human-Review des Schemas
  → Phase 2 (Entity + Repository im zustaendigen Domaenen-Modul + Flyway-Migration in odin-persistence)
  → Phase 3 (Integrationstest PFLICHT)
  → Phase 4 (Compile + Tests)
  → Phase 5
```

**Besonderheiten:**
- Schema-Migrationen (Flyway in odin-persistence): Human-Review PFLICHT
- Entities leben im zustaendigen Domaenen-Modul (brain, execution, core, audit) — nicht zentral in odin-persistence
- Entities sind keine DTOs — nie moduluebergreifend als Datentraeger
- EventLog-Aenderungen (odin-audit): Besondere Vorsicht — Replay-Kompatibilitaet sicherstellen
- Naming: `{Name}Entity`, `{Name}Repository`, `{Name}Embeddable`
- Bei neuer Entity: `@EnableJpaRepositories` und `@EntityScan` in odin-app erweitern

### 9.4 Concurrency und Thread-Modell

```
Auftrag → Phase 0 → Phase 1 (Design PFLICHT, Thread-Modell dokumentieren)
  → Human-Review des Designs
  → Phase 2 (Implementierung)
  → Phase 3 (Unit-Tests + Concurrency-spezifische Tests, siehe 6.5)
  → Phase 4 (Compile + Tests + ChatGPT-Review PFLICHT)
  → Phase 5
```

**Besonderheiten:**
- ODIN hat ein klar definiertes Thread-Modell (Kap 0, Abschnitt 6) — Aenderungen nur mit Human-Freigabe
- Decision Loop ist strikt sequentiell pro Pipeline — keine Parallelisierung ohne Auftrag
- GlobalRiskManager muss thread-safe sein (aggregiert ueber Pipelines)
- Bar-Close-Barrier in Simulation: Determinismus nicht brechen

### 9.5 Frontend (odin-frontend)

Frontend-Code unterliegt eigenen Konventionen:
- React/TypeScript, kein Java
- Java-Regeln (R5, R6, R7, CSpec) gelten NICHT
- Kommunikation mit Backend ueber SSE + REST — Aenderungen an den Endpunkten erfordern Koordination mit odin-app
- Frontend-spezifische Bauvorschriften: `guardrail-frontend.md`

---

## 10. Entscheidungsbaum: Schnellreferenz

```
Neuer Auftrag eingegangen
│
├── Alle Infos vorhanden?
│   ├── NEIN → STOP, nachfragen (R9)
│   └── JA ──┐
│             │
├── Risikostufe HOCH? (Phase 0, Abschnitt 3.4)
│   ├── JA → Human-in-the-Loop PFLICHT (uebersteuert alles Folgende)
│   └── NEIN ──┐
│              │
├── Betrifft public API / odin-api?
│   ├── JA → Human-in-the-Loop PFLICHT
│   └── NEIN ──┐
│              │
├── Neues Modul?
│   ├── JA → Human-in-the-Loop PFLICHT
│   └── NEIN ──┐
│              │
├── Cross-Modul-Aenderung?
│   ├── JA → Human-in-the-Loop PFLICHT
│   └── NEIN ──┐
│              │
├── Trading-kritisch (OMS, Broker, Risk)?
│   ├── JA → Design + Human-Review + ChatGPT-Review
│   └── NEIN ──┐
│              │
├── Neue Geschaeftslogik?
│   ├── JA → Architekturentscheidung (R4), dann autonom implementieren
│   └── NEIN ──┐
│              │
└── Bugfix / Config / internes Refactoring
    → Autonom implementieren, Compile + Tests
```

---

## 11. Anti-Patterns (was der Agent NICHT tun soll)

| Anti-Pattern | Regel |
|-------------|-------|
| Code schreiben ohne Auftrag | R5.1 |
| Annahmen treffen bei fehlenden Infos | R3.6, R8.2, R9.1 |
| Public APIs aendern ohne Human-Freigabe | R3.2 |
| `Instant.now()` im Trading-Code | Kap 0, MarketClock-Prinzip |
| Fachmodul importiert anderes Fachmodul | Kap 1, Dependency-Regeln |
| Spring-Annotations in Pro-Pipeline-Modulen | Guardrail-Modul-Struktur, Abschnitt 7 |
| Caching/Pooling ohne bewiesenes Problem | R11.2 |
| Abstraktion ohne Auftrag | R11.1 |
| `var` verwenden | R5.4 |
| Workarounds um Luecken zu schliessen | R9.2 |
| Eigenmaechtig committen | Nur auf explizite Nutzeranweisung |
| ChatGPT-Review fuer triviale Aenderungen | Lean-Prinzip — Prozess wo er Wert bringt |
| Silent-Fail in Trading-Code | Jeder Fehler eskalieren (Log, Event, Alert) |

---

## 12. Zusammenfassung der Checkpoints

| Phase | Checkpoint | Pflicht | Bedingung |
|-------|-----------|---------|-----------|
| 0 | Machbarkeitspruefung bestanden | IMMER | — |
| 0 | Risikostufe bestimmt | IMMER | HOCH uebersteuert alles |
| 1 | Architekturentscheidung dokumentiert | Wenn Design noetig | Neuer Service, Engine, Port |
| 1 | Human-Freigabe eingeholt | Bei HOCH oder Breaking Changes | Public API, neues Modul, Cross-Modul, Trading-kritisch |
| 2 | Modul-Guardrail-Konformitaet | IMMER | — |
| 2 | Code-Qualitaetsregeln eingehalten | IMMER | — |
| 2 | Observability bei Trading-Code | Bei HOCH | Structured Logging fuer Order-Lifecycle, Risk, State |
| 3 | Tests geschrieben | Bei neuer Logik | Siehe Test-Matrix |
| 3 | Trading-Szenario-Checkliste geprueft | Bei OMS/Broker/Risk | Siehe 9.1 |
| 4 | Compile erfolgreich | IMMER | — |
| 4 | Unit-Tests gruen | IMMER | — |
| 4 | Integrationstests gruen | Bei DB/Repo/Broker-Aenderung | — |
| 4 | ChatGPT-Review durchgefuehrt | Bei Trading-kritischem Code | OMS, Broker, Risk, Concurrency |
| 5 | git add neuer Dateien | IMMER | R14 |
| 5 | Ergebnisbericht | IMMER | — |
