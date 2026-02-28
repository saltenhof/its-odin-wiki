# ODIN-037: Structured SSE Events (Monitoring Streams)

**Modul:** odin-app
**Phase:** 1
**Groesse:** L
**Abhaengigkeiten:** ODIN-004 (DegradationMode), ODIN-006 (CycleContext)
**Geschaetzter Umfang:** L

---

## Kontext

Die aktuelle SSE-Implementierung hat einen globalen Stream mit generischen Events. Das Konzept definiert strukturierte Monitoring-Streams pro Pipeline und global, mit definierten Event-Typen und Payloads. Das Frontend braucht diese strukturierten Events fuer Echtzeit-Dashboard-Updates mit typsicheren Diskriminatoren.

## Scope

**In Scope:**
- Per-Pipeline SSE-Stream: `GET /api/v1/stream/pipeline/{instrumentId}`
- Global SSE-Stream: `GET /api/v1/stream/global` (bestehend, erweitern)
- Strukturierte Event-Typen: `PIPELINE_STATE`, `POSITION_UPDATE`, `INDICATOR_UPDATE`, `LLM_ANALYSIS`, `TRADE_EXECUTED`, `ALERT`, `DEGRADATION_CHANGE`, `CYCLE_UPDATE`
- Type-sichere DTOs pro Event-Typ
- Backpressure-Handling: `SseRingBuffer` pro Client

**Out of Scope:**
- WebSocket (SSE bleibt die einzige Push-Technologie)
- Historische Event-Abfrage ueber SSE (nur Echtzeit)
- Client-seitiges Filtern von Event-Typen

## Akzeptanzkriterien

- [ ] Per-Pipeline-Stream mit `instrumentId` als Path-Variable
- [ ] Discriminated Union Pattern: Jedes SSE-Event hat `type` Feld als Diskriminator
- [ ] `PIPELINE_STATE`: `{state, previousState, instrumentId, timestamp}`
- [ ] `POSITION_UPDATE`: `{instrumentId, quantity, avgEntryPrice, unrealizedPnl, realizedPnl}`
- [ ] `INDICATOR_UPDATE`: `{instrumentId, ema50, ema100, rsi, atr, regime, subregime}`
- [ ] `LLM_ANALYSIS`: `{instrumentId, action, confidence, reasoning (truncated to 200 chars)}`
- [ ] `TRADE_EXECUTED`: `{instrumentId, side, quantity, price, reason}`
- [ ] `ALERT`: `{level (INFO/WARNING/CRITICAL/EMERGENCY), message, timestamp}`
- [ ] `DEGRADATION_CHANGE`: `{oldMode, newMode, reason}`
- [ ] `CYCLE_UPDATE`: `{instrumentId, cycleNumber, cycleStartTime, previousCyclePnl}`
- [ ] Heartbeat bleibt als Keep-Alive (alle 5s, named event `heartbeat`)
- [ ] `SseRingBuffer` verhindert Memory-Leak bei langsamen Clients

## Technische Details

**Dateien:**
- `odin-app/src/main/java/de/its/odin/app/controller/StreamController.java` (Erweiterung)
- `odin-app/src/main/java/de/its/odin/app/sse/SseEventType.java` (Erweiterung/neue Enum-Werte)
- `odin-app/src/main/java/de/its/odin/app/dto/PipelineStateEvent.java` (und analog fuer alle 8 Event-DTOs)
- `odin-app/src/main/java/de/its/odin/app/sse/SseEmitterManager.java` (Per-Pipeline-Stream-Logik)

**Konfiguration:** Heartbeat-Interval via `odin.app.sse.heartbeat-interval-ms=5000`

## Konzept-Referenzen

- `docs/concept/10-observability.md` — Abschnitt 1 "Monitoring Streams"
- `docs/concept/10-observability.md` — Abschnitt 2 "SSE + REST Kommunikationsmodell"
- `docs/concept/10-observability.md` — Abschnitt 5 "Snapshot und DecisionLog Schemas"

## Guardrail-Referenzen

- `docs/frontend/guardrails/frontend.md` — "SSE: Named Events, addEventListener"
- `T:\codebase\its_odin\CLAUDE.md` — "SSE (Monitoring) + REST POST (Controls) — kein WebSocket"
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-app`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `MAX_REASONING_LENGTH`, `HEARTBEAT_EVENT_NAME`)
- [ ] Records fuer alle Event-DTOs
- [ ] `SseEventType` als ENUM mit allen 8 definierten Event-Typen + HEARTBEAT
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.app.sse.*` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer Event-Serialisierung: Alle 8 Event-DTOs korrekt als JSON serialisiert mit `type`-Feld als Diskriminator
- [ ] Unit-Tests fuer Per-Pipeline-Stream-Routing: `instrumentId` korrekt als Path-Variable geparst und Stream korrekt adressiert
- [ ] Unit-Tests fuer `SseRingBuffer`: Ueberlaeufe korrekt behandelt (aelteste Events verworfen)
- [ ] Unit-Tests fuer Heartbeat-Intervall-Konfiguration
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: SSE-Verbindung zu `/api/v1/stream/global` herstellen, Event senden, Event korrekt empfangen (nicht nur Serialisierung, sondern voller HTTP-SSE-Flow)
- [ ] Integrationstest: Per-Pipeline-Stream mit spezifischer `instrumentId` — nur Events fuer dieses Instrument werden empfangen
- [ ] Integrationstest: Heartbeat kommt alle 5s
- [ ] Integrationstest: Disconnect und Reconnect — kein Memory-Leak, neuer Stream korrekt aufgebaut
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt (SSE-Verbindung + Event-Empfang)

**Ziel:** Sicherstellen, dass StreamController, SseEmitterManager und SseRingBuffer korrekt zusammenarbeiten.

### 2.4 Tests — Datenbank

Nicht zutreffend. SSE-Streams sind reine In-Memory-Echtzeit-Kommunikation ohne Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `StreamController.java`, `SseEmitterManager.java`, alle Event-DTO-Klassen, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Was passiert bei sehr vielen gleichzeitigen SSE-Clients? Wie verhaelt sich der SseRingBuffer bei Pausen (z.B. Backend-Restart)? Koennen Events verloren gehen und ist das akzeptabel?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Thread-Safety bei SseEmitterManager, Memory-Leaks bei SSE-Client-Disconnect, korrekte SseEmitter-Timeout-Konfiguration, Null-Safety bei fehlenden Instrument-Daten"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` Abschnitte 1, 2, 5 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle Event-Typen und Payloads mit dem Konzept uebereinstimmen. Sind alle definierten Event-Felder implementiert? Stimmt das Discriminated-Union-Pattern mit der Frontend-Erwartung ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Welche praktischen Probleme entstehen bei SSE in Produktionssystemen mit Load-Balancern, Proxies (Nginx-Timeouts), oder vielen gleichzeitigen Clients? Was fehlt im aktuellen Design?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. SseRingBuffer-Groesse, Truncation-Laenge fuer LLM-Reasoning)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- Bestehende `SseEmitterManager` und `SseRingBuffer` nutzen/erweitern — nicht neu erstellen
- Named SSE Events (`addEventListener` statt `onmessage`) — Frontend erwartet named events (MEMORY.md SSE-Bugs)
- Heartbeat-Interval: 5000ms — aus MEMORY.md (war 15000ms, auf 5000ms reduziert)
- `SseEmitter` Timeout MUSS explizit gesetzt werden (no-arg Konstruktor = Spring Default ~30s, nicht verwenden!)
- `configureAsyncSupport()` in WebConfig mit `setDefaultTimeout(-1L)` muss gesetzt sein
- React StrictMode-kompatibel: Deferred disconnect (100ms grace period) ist Frontend-Angelegenheit, Backend muss sauber reconnecten
