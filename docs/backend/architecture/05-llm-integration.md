# ODIN — Kapitel 5: LLM-Integration (Claude SDK, OpenAI)

Version: 0.3 DRAFT
Stand: 2026-02-15
Review: Tiefenreview R2 — Monotoner Store-Update (P0.1), Array-Canonicalization (P1.1), SimRunner-Wording (P1.2)

---

## 1. Zweck dieses Kapitels

Dieses Kapitel beschreibt die Integration von LLM-Providern (Claude Agent SDK, OpenAI API) in ODIN. Es umfasst die Provider-Abstraktion, den Safety Contract, die Prompt-Architektur, die Aufruf-Strategie, die Resilience-Mechanismen und die Simulation-Semantik.

**Modulzuordnung:** `odin-brain` (Package: `de.odin.brain.llm`)

**Zentrale Abhaengigkeiten:**
- `LlmAnalyst` (Port aus odin-api) — Abstrahiert den LLM-Zugriff (Live: Provider-Clients, Sim: CachedAnalyst)
- `MarketClock` (Port aus odin-api) — Zeitreferenz fuer TTL, Freshness, Timing-Modell
- `EventLog` (Port aus odin-api) — Persistierung aller LLM-Calls (non-droppable)
- `RunContext` (aus odin-api) — runId, promptVersion, modelId als Join-Keys
- `MarketSnapshot` (von odin-data) — Input fuer Prompt-Kontext

---

## 2. Architekturprinzip: LLM als Analyst, nicht Policy-Maker

Das LLM liefert ausschliesslich **strukturierte Features und Unsicherheiten** — nie Handlungsanweisungen (Kap 0, Abschnitt 3.4). Alle Entscheidungen (Entry, Exit, Sizing) werden von der deterministischen Rules Engine und dem Decision Arbiter getroffen.

**Technische Durchsetzung:**
- Strikte JSON-Schema-Validierung jeder LLM-Response
- **Whitelist** definiert, welche Felder als Features in die Entscheidungslogik einfliessen duerfen
- Freitext-Felder fliessen ausschliesslich ins Logging
- Rules Engine und Arbiter nutzen LLM-Output nur als **Input fuer Scoring/Veto/Regime** — nie als direkte Handlungsanweisung
- **Kein LLM-Output darf direkt einen Trigger ausloesen** (z.B. kein "Entry weil LLM sagt Entry"). LLM-Features fliessen in die Rules Engine, die eigenstaendig entscheidet

### Feature-Felder vs. Freitext-Felder

| Kategorie | Felder | Nutzung |
|-----------|--------|---------|
| **Decision-Features** | `regime`, `regime_confidence`, `pattern_candidates` | Werden von Rules Engine und Quant Validation als Input-Features genutzt (nie als Anweisung) |
| **Kontext-Features** | `opportunity_zones`, `entry_price_zone`, `target_price`, `hold_duration_bars`, `urgency_level` | Stehen der Rules Engine als **optionale Kontextsignale** zur Verfuegung. Die Rules Engine kann sie beruecksichtigen oder ignorieren. Sie loesen nie direkt eine Aktion aus |
| **Logging-Only** | `reasoning`, `key_observations`, `risk_factors`, `market_context_signals` | Nur Telemetrie/Logging. Nie in Entscheidungslogik, nie in nachfolgende Prompts als Fakten |

> **Abgrenzung zu v0.1:** Die Entry-Approaching-Trigger (§5 in v0.1) nutzten LLM-Opportunity-Zones direkt als Trigger fuer LLM-Refreshes. Das ist entfernt — alle Trigger sind jetzt rein markt-/regelbasiert (Preis, Volumen, Indikatoren, Pipeline-State).

---

## 3. Provider-Abstraktion

### Port und Implementierungen

Der `LlmAnalyst`-Port lebt in `de.odin.api.port` (Kap 1). Implementierungen:

| Implementierung | Modul | Package | Modus |
|----------------|-------|---------|-------|
| `ClaudeAnalystClient` | odin-brain | `de.odin.brain.live` | Live (primaer) |
| `OpenAiAnalystClient` | odin-brain | `de.odin.brain.live` | Live (alternativ) |
| `CachedAnalyst` | odin-brain | `de.odin.brain.sim` | Simulation (Default) |

### Provider-Auswahl

Der aktive Provider wird **vor Handelsstart per Konfiguration** gewaehlt und steht fuer den gesamten Handelstag fest. Es gibt **kein Runtime-Switching** zwischen Providern.

> **Kein Provider-Failover:** Faellt der aktive Provider aus, greift der Circuit Breaker und der Agent wechselt in den Quant-Only-Modus (Abschnitt 8). Es wird nicht automatisch auf den anderen Provider gewechselt.

### Internes Schichten-Modell

```
LlmAnalyst (Port, odin-api)
       │
       v
LlmAnalystOrchestrator (odin-brain, pro Pipeline)
  ├── PromptBuilder         — Baut Kontext-Payload aus MarketSnapshot
  ├── SchemaValidator       — Validiert Response gegen JSON-Schema
  ├── PlausibilityChecker   — Prueft Features auf Plausibilitaet (ATR-relativ)
  ├── ConsistencyChecker    — Prueft Feld-Kombinationen
  └── CircuitBreaker        — Failure-Tracking, Quant-Only-Transition
       │
       v
Provider-Client (ClaudeAnalystClient | OpenAiAnalystClient | CachedAnalyst)
```

> **LlmAnalystOrchestrator** ist die pro-Pipeline instanziierte Klasse, die den LlmAnalyst-Port nutzt. Der HTTP-Client fuer die Provider ist ein **Singleton** (geteilte Rate-Limits), aber der Orchestrator ist **pro Pipeline** (eigener Kontext, eigener Circuit-Breaker-State, eigene TTL-Verwaltung).

---

## 4. LLM Safety Contract

### Schicht 1: Schema-Validierung

Jede LLM-Response wird gegen ein **striktes JSON-Schema** validiert:
- Enums mit geschlossener Wertemenge
- Numerische Bounds (z.B. `regime_confidence` 0.0–1.0)
- `additionalProperties = false`
- Unbekannte Enum-Werte, fehlende Felder, falsche Typen → **REJECT** (keine Toleranz)
- Bei Structured Outputs (Claude Tool-Use, OpenAI JSON-Mode) entfaellt der Schema-Retry — Retry nur bei Transport/Timeout-Fehlern

### Schicht 2: Plausibilitaetspruefung

| Halluzinationstyp | Pruefung | Aktion |
|-------------------|---------|--------|
| `entry_price_zone` weicht > 1.5x ATR vom Kurs ab | ATR-relative Pruefung gegen `MarketSnapshot.currentPrice` | Verwerfen |
| `target_price` > 4x ATR vom Entry | ATR-relative Pruefung | Target auf 2x ATR deckeln |
| `hold_duration_bars` > verbleibende Handelszeit | Zeitvergleich gegen `MarketClock` (RTH-Ende) | Kuerzen |
| `regime_confidence` = 1.0 bei unklarem Markt | Quant-Gegencheck (RSI nahe 50, ADX < 20) | Auf 0.7 deckeln |
| `opportunity_zones` ausserhalb Tagesrange + 1x ATR | ATR-erweiterte Range aus Snapshot | Verwerfen |

### Schicht 3: Konsistenzpruefung

| Kombination | Bewertung | Aktion |
|-------------|----------|--------|
| `regime = TREND_DOWN` + Entry-Opportunity-Zone | Inkonsistent | Zone verwerfen |
| `regime = UNCERTAIN` + `urgency = HIGH` | Inkonsistent | Urgency auf MEDIUM deckeln |
| `risk_factors` "starker Abwaertstrend" + `regime = TREND_UP` | Warnung | Logging (kein Eingriff — Freitext ≠ Decision-Feature) |

### Schicht 4: TTL/Freshness (MarketClock-basiert)

Jede LLM-Analyse traegt `requestMarketTime` (MarketClock-Zeitpunkt des Requests) und `availableFromMarketTime` (MarketClock-Zeitpunkt ab dem die Response nutzbar ist).

| Kontext | Max-Alter (MarketClock) | Reaktion bei Ueberschreitung |
|---------|------------------------|------------------------------|
| Entry-Entscheidung | 120s (`odin.llm.entry-freshness-max-s`) | Letzter LLM-Output verworfen, Entry blockiert bis frischer Output vorliegt |
| Position-Management | 900s (`odin.llm.management-freshness-max-s`) | Letzten gueltigen Output verwenden |
| Kein Output verfuegbar | — | Default: `regime = UNCERTAIN`, `regimeConfidence = 0.0` (Entries blockiert, siehe Abschnitt 8) |

> **MarketClock statt Systemzeit:** TTL-Berechnung basiert auf `MarketClock.now() - requestMarketTime`. In der Simulation ist das die vom Runner gesteuerte SimClock — eine 5-Minuten-alte Analyse verfaellt auch dann nach 120s MarketTime, wenn die Simulation in 100ms laeuft.

---

## 5. LLM-Analyse-Schema (v1.1)

### Output-Schema

| Feld | Typ | Kategorie | Beschreibung |
|------|-----|-----------|-------------|
| `regime` | Enum (TREND_UP, TREND_DOWN, RANGE_BOUND, HIGH_VOLATILITY, UNCERTAIN) | Decision-Feature | Aktuelle Marktphase |
| `regime_confidence` | Float 0.0–1.0 | Decision-Feature | Konfidenz der Regime-Einschaetzung |
| `pattern_candidates` | Object[] ({pattern: Enum, confidence: Float, phase: String}) | Decision-Feature | Erkannte Chart-Patterns und deren Phase |
| `opportunity_zones` | Object[] ({price_min, price_max, type: ENTRY/EXIT, reasoning}) | Kontext-Feature | Potentielle Zonen (Rules Engine entscheidet ob relevant) |
| `urgency_level` | Enum (LOW, MEDIUM, HIGH, CRITICAL) | Kontext-Feature | Zeitdruck-Einschaetzung |
| `entry_price_zone` | Object ({min, ideal, max}) | Kontext-Feature | Geschaetzter fairer Einstiegsbereich |
| `target_price` | Float oder null | Kontext-Feature | Geschaetztes Kursziel (mit Plausibilitaets-Cap) |
| `hold_duration_bars` | Int | Kontext-Feature | Geschaetzte Haltedauer in Decision-Bars |
| `market_context_signals` | String[] (max 5) | Logging-Only | Markt-Beobachtungen |
| `reasoning` | String (max 200 Zeichen) | Logging-Only | Kurze Begruendung |
| `key_observations` | String[] (max 5) | Logging-Only | Auffaelligkeiten |
| `risk_factors` | String[] | Logging-Only | Identifizierte Risiken |

### Prompt-Versionierung

| Aspekt | Regelung |
|--------|---------|
| Schema | Semantic Versioning (Major = Schema-Aenderung, Minor = Regel, Patch = Wording) |
| Changelog | Datum, Autor, Begruendung pro Aenderung |
| Rollback | Jederzeit auf vorherige Version per Konfigurationsschalter (kein Deployment) |
| Evaluation | 50+ Regressionstests, 10+ Halluzinations-Tests, Schema-Compliance 100%, Latenz p95 < 5s |
| Versionierung im RunContext | `promptVersion` ist Teil des `RunContext` (Kap 1) — jeder Run ist einer Prompt-Version zugeordnet |

---

## 6. Prompt-Architektur (Context Window)

### Dreistufiges Context Window

| Stufe | Inhalt | Aktualisierung |
|-------|--------|---------------|
| 1. System Prompt (fest) | Rollendefinition ("Du bist Marktanalyst"), Output-Schema, Confidence-Kalibrierungsregeln, Instrument-Profil (Name, Sektor, Beta, avg. Tagesvolatilitaet) | Fest pro Prompt-Version |
| 2. Kontext-Fenster (dynamisch) | Komprimierte Kurshistorie (20d Daily + 5d Intraday), heutiger Tagesplan, bisherige Trades heute, aktuelle Position und offene Stops | Pro LLM-Aufruf aktualisiert |
| 3. Aktueller Snapshot (pro Aufruf) | Letzte N Decision-Bars, aktuelle Indikatoren (aus IndicatorResult: RSI, ATR, EMA, ADX, Bollinger), VWAP (aus Snapshot), Volumen-Anomalien, Uhrzeit (MarketClock) und verbleibende Handelszeit | Pro LLM-Aufruf |

### Datenkompression

| Zeitfenster | Granularitaet | Datenpunkte |
|-------------|--------------|-------------|
| Letzte 15 Minuten | 1-Minuten-Bars (OHLCV) | ~15 |
| Letzte 2 Stunden | 5-Minuten-Bars (OHLCV) | ~24 |
| Rest des Tages | 15-Minuten-Bars (OHLCV) | ~16 |
| Letzte 5 Handelstage | 30-Minuten-Bars | ~65 |
| Letzte 20 Handelstage | Daily Bars | 20 |

Zusaetzlich: VWAP, kumulatives Volumen, Intraday-High/Low, aktuelle Indikator-Werte aus dem letzten IndicatorResult.

> **Daten aus MarketSnapshot:** Alle Prompt-Daten kommen aus dem `MarketSnapshot` (Bars, VWAP, SpreadStats) und dem letzten `IndicatorResult` (KPI-Werte, Kap 4). Keine direkten Buffer-Zugriffe — der Prompt-Builder arbeitet nur mit immutable Snapshots.

---

## 7. Aufruf-Strategie

### Trigger-Typen

Alle Trigger sind **rein markt-/regelbasiert** — kein Trigger nutzt direkt LLM-Output als Ausloeser:

| Typ | Bedingung | Prioritaet |
|-----|-----------|-----------|
| Periodisch (Timer) | Intervall abhaengig von Volatilitaet (ATR vs. Daily-ATR, MarketClock-basiert) | Normal |
| Event-Driven | VWAP-Durchbruch, Volumen-Spike > 2x Avg, Stop-Loss-Annaeherung (< 0.5x ATR), Profit-Target erreicht | Hoch |
| State-Transition | Zustandswechsel der Pipeline-FSM (z.B. OBSERVING → POSITIONED) | Hoch |

> **Entfernt gegenueber v0.1:** Der "Entry-Approaching"-Trigger (Preis betritt LLM-Opportunity-Zone) ist entfernt. Dieser nutzte LLM-Output direkt als Trigger — das widerspricht dem Analyst-Prinzip. Stattdessen triggert die Rules Engine bei Bedarf einen LLM-Refresh ueber den Orchestrator, wenn sie zusaetzlichen Kontext benoetigt.

### Periodisches Intervall (volatilitaetsabhaengig, MarketClock)

| Volatilitaet (ATR vs. Daily-ATR) | LLM-Intervall |
|----------------------------------|---------------|
| Hoch (> 120% Daily-ATR) | Alle 3 Minuten |
| Normal (80–120%) | Alle 5 Minuten |
| Niedrig (< 80%) | Alle 15 Minuten |

### Single-Flight-Semantik

Nie mehr als ein laufender LLM-Call gleichzeitig **pro Pipeline**. Verschiedene Pipelines duerfen parallel LLM-Calls ausfuehren (isolierte Pipelines, geteilter HTTP-Client). Weitere Trigger innerhalb einer Pipeline werden koalesziert (zusammengefasst und beim naechsten Call beruecksichtigt).

---

## 8. Asynchrone Interaktion mit dem Decision-Cycle

### Grundprinzip

Der LLM-Call ist **asynchron** und blockiert den Decision-Cycle **nicht** (Kap 0, Abschnitt 6: LLM-IO asynchron, Single-Flight). Der Decision Arbiter verwendet den **letzten verfuegbaren** LLM-Output (Last-Write-Wins).

```
       LLM-Call (async, im Hintergrund)
           │
           v
  ┌──────────────────────────┐
  │ LlmAnalysisStore         │  ← AtomicReference (pro Pipeline)
  │ (letzter validierter     │     Monotoner Update:
  │  LLM-Output + Metadaten) │     nur wenn new.requestMarketTime
  └──────────────────────────┘       >= current.requestMarketTime
           │
           v (gelesen von)
  Decision-Cycle (synchron):
  KPI → Rules → Quant → Arbiter
```

**Monotoner Store-Update (Last-Write-Wins mit Zeitgarantie):**

Der `LlmAnalysisStore` akzeptiert ein Update **nur wenn** `newAnalysis.requestMarketTime >= currentAnalysis.requestMarketTime`. Spaet eintreffende Responses auf aeltere Requests (z.B. durch Netzwerk-Jitter oder Retry) werden **verworfen** (logged als `STALE_RESPONSE_DISCARDED`). Dadurch kann der Store nie auf einen aelteren MarketTime-Stand zurueckfallen — die monotone Eigenschaft ist garantiert.

Implementierung: `AtomicReference.getAndUpdate(current -> new.requestMarketTime >= current.requestMarketTime ? new : current)`. Bei Discard wird ein Debug-Level EventLog-Eintrag geschrieben.

### LLM-Output-Lifecycle

Jeder validierte LLM-Output wird als `LlmAnalysis` (Record in odin-api) gespeichert:

```
LlmAnalysis (Record, immutable, in odin-api)
├── runId               : String
├── instrumentId        : String
├── requestMarketTime   : Instant (MarketClock-Zeit des Requests)
├── availableFromMarketTime : Instant (requestMarketTime + Δt_model)
├── regime              : Regime (Enum)
├── regimeConfidence    : double (0.0–1.0)
├── patternCandidates   : List<PatternCandidate>
├── opportunityZones    : List<OpportunityZone> (Kontext-Feature)
├── urgencyLevel        : UrgencyLevel (Enum)
├── entryPriceZone      : EntryPriceZone (Kontext-Feature)
├── targetPrice         : Double (nullable, mit Plausibilitaets-Cap)
├── holdDurationBars    : int
├── promptVersion       : String
├── modelId             : String
├── provider            : LlmProvider (Enum)
├── cacheHit            : boolean (true wenn aus CachedAnalyst)
├── latencyMs           : long (Provider-Response-Time)
├── inputTokens         : int
├── outputTokens        : int
└── validationStatus    : ValidationStatus (VALID, PLAUSIBILITY_ADJUSTED, REJECTED)
```

### Freshness-Check im Decision-Cycle

Der Decision Arbiter prueft vor Nutzung eines LLM-Outputs:

1. `MarketClock.now() - availableFromMarketTime < entryFreshnessMax` (fuer Entry-Entscheidungen)
2. `MarketClock.now() - availableFromMarketTime < managementFreshnessMax` (fuer Position-Management)
3. Wenn stale oder nicht vorhanden → Default-Werte (siehe unten)

### Default-Werte bei fehlendem/stale LLM-Output

| Situation | Default | Auswirkung auf Kap 4 (Quant Validation) |
|-----------|---------|----------------------------------------|
| Kein LLM-Output vorhanden (Startup, erster Call noch ausstehend) | `regime = UNCERTAIN`, `regimeConfidence = 0.0` | Entries blockiert (regimeConfidence < 0.5, Kap 4 §8) |
| LLM-Output stale (TTL ueberschritten) | `regime = UNCERTAIN`, `regimeConfidence = 0.0` | Entries blockiert |
| Circuit Breaker aktiv (Quant-Only-Modus) | `regime = UNCERTAIN`, `regimeConfidence = 0.0` | Entries blockiert. Position-Management regelbasiert (Stops, EOD-Flat) |
| LLM-Response REJECTED (Schema/Plausibilitaet) | Letzter gueltiger Output beibehalten (falls innerhalb TTL), sonst Default | — |

> **Konsistenz mit Kap 4:** `regimeConfidence = 0.0` fuehrt in der Quant Validation (Kap 4, Abschnitt 8) zu "Entry verboten (unabhaengig vom Score)". Das ist das gewuenschte Verhalten — ohne frischen LLM-Kontext keine neuen Positionen.

---

## 9. Circuit Breaker und Quant-Only-Modus

### Timeout-Policy

| Ebene | Schwelle | Reaktion | Recovery |
|-------|----------|----------|----------|
| Per-Call-Timeout | Response > 10s (MarketClock) | Call abbrechen. Failure-Counter +1 | — |
| Consecutive Failures | 3 Timeouts/5xx in Folge | **Quant-Only-Modus** | Erster erfolgreicher Call → Reset |
| Prolonged Outage | > 30 Min (MarketClock) ohne Erfolg | **Trade-Halt** | Manueller Reset oder erster Erfolg |

### Circuit-Breaker-Policy-Matrix

| Stufe | Neue Entries | Position-Management | Stops/Forced-Close |
|-------|-------------|--------------------|--------------------|
| Normal | Erlaubt (frischer LLM-Kontext) | LLM-gestuetzt | Immer aktiv |
| Quant-Only (3 Failures) | **Blockiert** (`regimeConfidence = 0.0`) | Regelbasiert (Trailing-Stops, Scaling-Out) | Immer aktiv |
| Trade-Halt (> 30 Min Outage) | **Blockiert** | Nur passive Verwaltung (Stops bleiben) | Immer aktiv |

> **Quant-Only und Trade-Halt sind aufeinander aufbauend.** Bestehende Positionen werden in beiden Stufen nicht zwangsgeschlossen — sie laufen mit ihren bestehenden Stops weiter. EOD-Flat bleibt aktiv.

---

## 10. Simulation-Semantik

### CachedAnalyst (Default-Sim-Profil)

Der `CachedAnalyst` implementiert den `LlmAnalyst`-Port und liefert gecachte LLM-Responses fuer reproduzierbare Backtests:

**Cache-Key:** `Hash(canonicalizedRequestPayload + promptVersion + modelId)`

| Bestandteil | Beschreibung |
|-------------|-------------|
| `canonicalizedRequestPayload` | Stabile JSON-Serialisierung des Request-Payloads (deterministische Feld-Reihenfolge, Float-Rounding auf 6 Dezimalstellen). **Array-Behandlung:** Mengenartige Listen (z.B. aktive Indikatoren, Feature-Flags) werden alphabetisch sortiert. Reihenfolge-tragende Listen (z.B. Bars in Zeitreihe, komprimierte Kurshistorie) behalten ihre Erzeugungsreihenfolge — diese muss bereits deterministisch sein (aufsteigende Zeitstempel). Identisch mit dem an den Provider gesendeten Payload |
| `promptVersion` | Schema-Version (z.B. "1.1.0") aus dem `RunContext` |
| `modelId` | Model-ID (z.B. "claude-sonnet-4-5-20250929") aus dem `RunContext` |

> **Kein Provider im Key:** Der Cache ist provider-spezifisch (Claude-Responses gelten nicht fuer OpenAI-Anfragen). Das ist durch `modelId` implizit abgedeckt.

> **Invalidierung:** Bei Prompt-Version-Wechsel (Major oder Minor) wird der Cache fuer neue Version-Anfragen leer sein — Responses werden dann live geholt und fuer zukuenftige Sim-Laeufe gespeichert.

### LLM-Timing-Modell (Δt_model)

Jeder LLM-Request wird bei MarketTime `T` gestellt. Die Response gilt als verfuegbar bei MarketTime `T + Δt_model` (Kap 0, Abschnitt 3.3):

| Sim-Profil | Δt_model | Einsatz |
|------------|----------|---------|
| **Cached (Default)** | Konfigurierbar (`odin.simulation.llm.delta-t-model-ms`, Default: 0) | 0 fuer Massen-Backtests, realistischer Wert (z.B. 3000ms) fuer Timing-Effekte |
| **Live-LLM** | Tatsaechliche Antwortzeit des Providers | Qualitative Analyse, Prompt-Evaluation |

> **SimulationRunner-Interaktion:** Der SimulationRunner steuert die MarketClock — er setzt `MarketClock.now()` auf den Zeitpunkt des naechsten Events. Der CachedAnalyst traegt `availableFromMarketTime = requestMarketTime + Δt_model` in die Response ein. Der Decision-Cycle **blockiert nicht** auf LLM-Responses (Kap 0, §6); stattdessen prueft der Arbiter ob `MarketClock.now() >= availableFromMarketTime` — nur dann gilt die Analyse als verfuegbar. Bei `Δt_model = 0` ist das sofort der Fall. Der Runner muss nicht explizit "warten", weil die Clock-Semantik die Verfuegbarkeit deterministisch steuert.

### Reproduzierbarkeit

| Garantie | Mechanismus | Einschraenkung |
|----------|-------------|---------------|
| **Harte Reproduzierbarkeit** | CachedAnalyst (Cache-Hit) | Nur fuer bereits gecachte Request-Payloads |
| **Best-Effort Reproduzierbarkeit** | `temperature = 0`, `seed` (wo unterstuetzt) | Provider-abhaengig. OpenAI: "mostly identical". Claude: kein Seed-Parameter, temperature=0 reduziert Varianz. **Nicht zuverlaessig ueber Model-Updates** |

> **Fazit:** Fuer reproduzierbare Backtests ist **CachedAnalyst Pflicht**. Live-LLM in Sim ist fuer qualitative Analyse, nicht fuer deterministische Reproduktion.

### Provider-Capabilities

| Setting | Claude (Agent SDK) | OpenAI (REST API) |
|---------|-------------------|-------------------|
| `temperature` | Unterstuetzt (0 = minimal) | Unterstuetzt (0 = minimal) |
| `seed` | **Nicht unterstuetzt** | Unterstuetzt (best-effort) |
| `max_tokens` | Unterstuetzt | Unterstuetzt |
| Structured Output | Tool-Use (strict schema) | JSON-Mode / Structured Outputs |
| Determinismus-Garantie | Keine (temperature=0 reduziert Varianz) | "Mostly identical" mit Seed |

---

## 11. EventLog-Integration

Alle LLM-Calls werden ins EventLog geschrieben (**non-droppable**, Kap 0 Abschnitt 10):

### Geloggte Felder

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `runId` | String | Join-Key zum RunContext |
| `instrumentId` | String | Pipeline-Zuordnung |
| `requestMarketTime` | Instant | MarketClock-Zeit des Requests |
| `availableFromMarketTime` | Instant | MarketClock-Zeit ab der Response nutzbar |
| `promptVersion` | String | Schema-Version |
| `modelId` | String | Model-ID |
| `provider` | Enum | CLAUDE / OPENAI / CACHED |
| `cacheHit` | boolean | true bei CachedAnalyst |
| `latencyMs` | long | Provider-Response-Time (0 bei Cache-Hit) |
| `inputTokens` | int | Token-Verbrauch (Input) |
| `outputTokens` | int | Token-Verbrauch (Output) |
| `validationStatus` | Enum | VALID / PLAUSIBILITY_ADJUSTED / REJECTED |
| `errorReason` | String | Fehlergrund (null bei Erfolg) |
| `responsePayload` | String | Vollstaendige Response (fuer Replay/Forensik) |

### Drop-Policy und Failure-Semantik

| Modus | Schreibverhalten | Failure-Reaktion |
|-------|-----------------|-----------------|
| **Simulation** | **Synchron** (innerhalb Bar-Close-Barrier) | Failure = harter Fehler → Simulation abgebrochen |
| **Live** | **Asynchron** (non-blocking). Non-droppable: Bounded Spool (Kap 2) | Spool-Overflow → ESCALATE → odin-core entscheidet |

> **Replay-Relevanz:** Die vollstaendige Response im EventLog ermoeglicht: (1) Post-Mortem-Analyse bei falschen Trades, (2) Cache-Aufbau fuer zukuenftige Sim-Laeufe, (3) Regression-Tests bei Prompt-Aenderungen.

---

## 12. Rate-Limiting und Kosten-Governor

### Scope: Global pro Run

Rate-Limits und Token-Budget gelten **global ueber alle Pipelines**, da sie einen geteilten HTTP-Client und API-Key nutzen:

| Aspekt | Scope | Default | Regelung |
|--------|-------|---------|---------|
| Max Calls/Minute | Global | 10 | Globaler Rate-Limiter im HTTP-Client. Pipelines teilen sich das Budget fair (Round-Robin bei gleichzeitigem Bedarf) |
| Tages-Token-Budget | Global | 500.000 | Bei Ueberschreitung: Quant-Only fuer alle Pipelines |
| Prioritaet | Pro Pipeline | — | Position-Management-Calls haben Vorrang vor periodischen Calls. Innerhalb einer Pipeline: Event-Driven > Periodisch |
| Telemetrie | Pro Call | — | Latenz, Token-Verbrauch, Fehlercode. Aggregiert pro Session und Pipeline |

---

## 13. News-Input-Sicherheit

Externe Textdaten (Nachrichten, Social Media) werden als **untrusted input** behandelt:

| Massnahme | Details |
|-----------|---------|
| Whitelist-Felder | Nur: Headline (max 200 Chars), Source, Timestamp, Sentiment-Score |
| Keine Rohtexte | Artikel-Body wird NIE in den Prompt eingespeist |
| Character-Sanitization | Nur ASCII + Standard-Unicode. Control Characters werden entfernt |
| Length Limits | Striktes Max-Length pro Feld, Truncation bei Ueberschreitung |
| Isolation | Externe Texte im dedizierten JSON-Block (`external_context`), nicht als System/User-Prompt |
| Anomalie-Detection | Bei signifikanter Abweichung nach News-Einspeisung: Antwort verwerfen, ohne News wiederholen |

> **v1-Default:** `odin.llm.news.enabled = false`. News-Integration ist vorbereitet aber deaktiviert.

---

## 14. Datenminimierung und Provider-Controls

- **Kein Account-Bezug im Prompt:** Keine Kontogroesse, keine persoenlichen Daten, keine historische P&L
- **Marktdaten:** Nur aggregierte/komprimierte Daten (Bars, Indikatoren), keine Roh-Ticks
- **Provider-Retention:** Nutzung von Provider-seitigen "no training"-Optionen (Claude: Default, OpenAI: API-TOS). Dokumentiert in Konfiguration
- **LLM-Freitext-Felder** (reasoning, observations, risk_factors) werden **nie** in nachfolgende Prompts als Fakten uebernommen — nur Telemetrie/Logging. Verhindert self-reinforcing Hallucinations

---

## 15. Scope und Lifecycle

### Pro-Pipeline-Instanziierung

Der `LlmAnalystOrchestrator` wird **pro Pipeline** instanziiert (Kap 1, Abschnitt 7):
- Eigener Circuit-Breaker-State
- Eigene TTL-Verwaltung und LlmAnalysisStore (AtomicReference)
- Eigener Trigger-Timer (MarketClock-basiert)
- Pipeline-spezifischer Kontext (Instrument-Profil, bisherige Trades, Position)

**Geteilte Ressourcen (Singleton):**
- HTTP-Client (Rate-Limits, Connection-Pooling)
- Token-Budget-Zaehler (global)

**Konfigurationsweitergabe:** Spring laedt `LlmProperties` (Record mit `@ConfigurationProperties`) einmal beim Start. PipelineFactory reicht sie an jeden Orchestrator weiter (kein Spring-Bean, Kap 1).

### Lifecycle

| Phase | LLM-Verhalten |
|-------|--------------|
| **INITIALIZING** | Erster LLM-Call wird vorbereitet. Kein Output verfuegbar → Default-Werte (`regimeConfidence = 0.0`) |
| **WARMUP** | Erster periodischer LLM-Call wird abgesetzt. Output wird in LlmAnalysisStore gespeichert sobald verfuegbar |
| **ACTIVE** | Regulaere periodische und event-driven Calls. Circuit Breaker aktiv |
| **EOD** | Letzter LLM-Call (optional, fuer Tages-Zusammenfassung). Danach keine weiteren Calls |

---

## 16. Konfiguration

```properties
# odin-brain.properties (LLM-Abschnitt)

# Provider-Auswahl (CLAUDE oder OPENAI)
odin.llm.active-provider=CLAUDE

# Claude
odin.llm.claude.model=claude-sonnet-4-5-20250929
odin.llm.claude.temperature=0
odin.llm.claude.max-tokens=2000

# OpenAI
odin.llm.openai.model=gpt-4o
odin.llm.openai.temperature=0
odin.llm.openai.max-tokens=2000
odin.llm.openai.seed=42

# Timeouts
odin.llm.call-timeout-s=10
odin.llm.circuit-breaker-threshold=3
odin.llm.prolonged-outage-threshold-min=30

# Freshness (MarketClock-basiert)
odin.llm.entry-freshness-max-s=120
odin.llm.management-freshness-max-s=900

# Aufruf-Intervalle (MarketClock-basiert)
odin.llm.interval-high-volatility-s=180
odin.llm.interval-normal-volatility-s=300
odin.llm.interval-low-volatility-s=900

# Rate-Limiting (global)
odin.llm.max-calls-per-minute=10
odin.llm.daily-token-budget=500000

# Prompt-Version
odin.llm.prompt-version=1.1.0

# News-Security
odin.llm.news.headline-max-length=200
odin.llm.news.enabled=false

# Simulation
odin.simulation.llm.delta-t-model-ms=0
```

> **Kein `odin.llm.claude.seed`:** Claude unterstuetzt keinen Seed-Parameter. Reproduzierbarkeit ausschliesslich ueber CachedAnalyst (Abschnitt 10).

---

## 17. Abhaengigkeiten und Schnittstellen

### Konsumiert

- `LlmAnalyst` (Port aus odin-api) — Abstrahiert Provider-Zugriff
- `MarketSnapshot` (von odin-data) — Prompt-Daten (Bars, VWAP, SpreadStats, Preise)
- `IndicatorResult` (von KPI-Engine, Kap 4) — Indikator-Werte fuer Prompt-Kontext
- `MarketClock` (Port) — TTL-Berechnung, Intervall-Steuerung, Timing-Modell
- `EventLog` (Port) — Persistierung aller LLM-Calls (non-droppable)
- `RunContext` — runId, promptVersion, modelId

### Produziert

- `LlmAnalysis` (DTO in odin-api) — Validierter LLM-Output mit Metadaten (im LlmAnalysisStore pro Pipeline)
- Wird von Rules Engine (regime, pattern_candidates, Kontext-Features) und Quant Validation (regimeConfidence, Kap 4 §8) konsumiert

### Nicht in Scope

- Rules Engine → Kap 6
- Quant Validation → Kap 4
- Decision Arbiter → implizit in Kap 6/7
- BrokerGateway → Kap 3
