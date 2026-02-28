# Protokoll: ODIN-018 — Pre-Market Instrument Profiling

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (31 Tests, alle gruen)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (8 Tests, alle gruen)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [ ] Commit & Push (KEIN Commit laut Aufgabe)

## Design-Entscheidungen

### 1. LlmProviderClient statt LlmAnalyst Port

Die Story beschrieb, das Profiling via `LlmAnalyst` (Port) zu machen. Das ist aber nicht korrekt:
- `LlmAnalyst.analyze()` gibt `LlmAnalysis` zurueck (schema-validiertes Tactical Output)
- Profiling braucht ein anderes JSON-Schema (5 Dimensionen statt Tactical Parameters)

**Entscheidung:** `InstrumentProfiler` injiziert `LlmProviderClient` direkt und baut seinen eigenen System-Prompt + User-Message. Das erlaubt ein eigenes Schema ohne Interference mit der Tactical-Output-Validierungskette.

### 2. Enum-Dimensionen statt Score 0.0-1.0

Die Story schlug "Score 0.0-1.0 oder Enum" vor. Konzept 04/8.2 definiert explizite Enums.

**Entscheidung:** Reine ENUMs (GapBehavior, RecoveryTendency, VolatilityProfile, TrendPersistence) gemaess Konzept. Keine Float-Scores - zu ambig, schwererer LLM-Instruction.

### 3. Fallback bei partieller LLM-Response

Wenn nur 3 von 5 Dimensionen im LLM-Response vorhanden sind (z.B. durch Halluzination oder verpasste Felder), werden die fehlenden Felder auf ihre Default-Werte gesetzt (`MIXED`, `MODERATE`, `EVEN`), aber `isDefault = false`. Das Profil ist NICHT als Default markiert, weil zumindest Teile echte LLM-Information enthalten.

Sonderfall: Wenn KEIN einziges Dimensions-Feld erkannt wird (z.B. komplett falsches JSON-Format oder freier Text), wird `isDefault = true` gesetzt. Die Pruefung erfolgt durch vier separate `matcher.find()`-Aufrufe vor dem eigentlichen Parsen.

### 4. AtomicReference fuer Thread-Safety

Gemaess Hinweis in der Story: Das gecachte Profil wird einmalig geschrieben und dann nur gelesen. `AtomicReference<InstrumentProfile>` genuegt - kein Lock benoetigt.

**CAS-Pattern (nach ChatGPT-Review):** `compareAndSet(null, profile)` statt `set()`. Garantiert, dass bei gleichzeitigem Zugriff (Race Condition im WARMUP) nur eine Instanz den LLM-Call gewinnt und als einzige das EventLog beschreibt. Die verlierenden Threads erhalten das Profil des Gewinners via `cachedProfile.get()`.

### 5. PromptBuilder.buildSystemPrompt() Overload

Neuer Overload `buildSystemPrompt(promptVersion, instrumentProfile)` statt Aenderung der bestehenden Signatur. Rueckwaertskompatibel. Der 1-Parameter-Aufruf delegiert an den neuen 2-Parameter-Aufruf mit `null`.

### 6. Default-Profile wird nicht ins System-Prompt injiziert

Default-Profile enthalten keine instrument-spezifischen Informationen. Ein Block "Gap behavior: MIXED, Recovery: MODERATE" waere reines Rauschen. Deshalb wird nur ein echter, nicht-default Profile in den System-Prompt eingebaut.

### 7. buildInstrumentProfileBlock als package-private static

Testbar ohne Refactoring, gleichzeitig kein oeffentliches API das unnoetig exponiert wird.

### 8. safeLogProfile fuer EventLog-Fehlertoleranz (nach ChatGPT-Review)

EventLog-Fehler duerfen den WARMUP nicht blockieren. `safeLogProfile()` kapselt den `eventLog.append()`-Aufruf in try/catch. Bei Fehler: WARN-Logging, WARMUP laeuft weiter.

### 9. REASONING_PATTERN ohne Laengenlimit im Regex (nach ChatGPT-Review)

Initiale Version hatte `{0,200}` im Regex — bei Reasoning > 200 Zeichen wuerde ein leerer String zurueckgegeben. Korrekte Loesung: Regex matcht unbegrenzt (`[^\"]*`), Kuerzen erfolgt im Code durch `substring(0, MAX_REASONING_LENGTH)`.

### 10. escapeJson mit vollstaendiger Control-Char-Behandlung (nach ChatGPT-Review)

Initiale Version escaped nur `\\` und `"`. Erweitert auf CR (`\r`), LF (`\n`), Tab (`\t`) — notwendig fuer gueltiges JSON im EventLog-Payload, besonders wenn das LLM Zeilenumbrueche im reasoning-Feld produziert.

## Offene Punkte

Keine kritischen offenen Punkte. Zur Diskussion (nicht blocking):

1. **QuantEngine-Validierung (Konzept 04/8.3):** Das Konzept beschreibt, dass LLM-Einschaetzungen durch quantitative Statistik validiert werden sollen (z.B. wenn LLM `recovery_tendency = HIGH` sagt, aber V-Recovery-Rate < 30%). Diese Schicht ist in ODIN-018 NICHT implementiert, da keine QuantEngine-Infrastruktur fuer historische Auswertungen existiert. Das ist ein bewusster Out-of-Scope.

2. **Re-Profiling bei extremen Events:** Was passiert wenn Trading Halt oder Circuit Breaker auftritt? Das Profil bleibt unveraendert (immutable fuer den Tag). Akzeptabel da das Profil nur historisches Verhalten beschreibt.

3. **LLM-Timeout blockiert WARMUP (Gemini-Finding):** `LlmProviderClient.callApi()` hat keinen eigenen Timeout-Mechanismus in `InstrumentProfiler`. Bei einem haengenden Provider (z.B. 30s HTTP-Timeout) blockiert der WARMUP entsprechend lang. Mitigation: Der Timeout liegt im HTTP-Client des jeweiligen Provider-Adapters. Akzeptiert fuer ODIN-018; ein dediziierter PROFILING_TIMEOUT koennte in einer Folge-Story ergaenzt werden.

4. **Staleness bei Catalyst-Events (Gemini-Finding):** Das Profil basiert auf historischen Daily Bars. Bei grossen Catalyst-Events (Earnings Surprise, FDA-Entscheid) am Profiling-Tag koennte es irrelevant oder sogar irrefuehrend sein. Akzeptiert als bekannte Einschraenkung; das Profil beschreibt historisches Normalverhalten, nicht den aktuellen Tag.

5. **Fleet-Start Rate-Limiting (Gemini-Finding):** Bei mehreren parallelen Pipelines starten alle `InstrumentProfiler`-Instanzen gleichzeitig und feuern je einen LLM-Call. Bei n=10+ Instrumenten koennte das Rate-Limits des LLM-Providers triggern. Relevant erst bei groesserer Fleet-Groesse; akzeptiert fuer ODIN-018.

## ChatGPT-Sparring

**Durchgefuehrt:** 2 Runden (Runde 1: Code-Review + Edge-Cases, Runde 2: Vertiefung der kritischen Findings)

### Runde 1 — Findings und Entscheidungen

**Finding 1: CAS-Pattern fehlt (KRITISCH — implementiert)**
- ChatGPT: Initiale Version verwendete `cachedProfile.set(profile)` nach dem LLM-Aufruf. Bei zwei gleichzeitigen Threads koennte das Profil doppelt gebaut und doppelt ins EventLog geschrieben werden.
- Entscheidung: `compareAndSet(null, profile)` + EventLog-Logging nur fuer `didSet == true`. Verliererthreads geben `cachedProfile.get()` zurueck (Gewinner-Profil).

**Finding 2: EventLog kann WARMUP blockieren (KRITISCH — implementiert)**
- ChatGPT: Wenn `eventLog.append()` eine Exception wirft (z.B. DB-Verbindungsfehler), wuerde der WARMUP abbrechen obwohl das Profil erfolgreich erstellt wurde.
- Entscheidung: `safeLogProfile()` mit try/catch; bei Fehler WARN-Log und WARMUP laeuft weiter.

**Finding 3: REASONING_PATTERN mit Laengenlimit (WICHTIG — implementiert)**
- ChatGPT: `{0,200}` im Regex gibt leeren String zurueck wenn Reasoning > 200 Zeichen. Das verschleiert Parsing-Fehler und loescht valide Information.
- Entscheidung: Regex ohne Laengenlimit, `substring()` im Code.

**Finding 4: escapeJson unvollstaendig (WICHTIG — implementiert)**
- ChatGPT: Fehlende CR/LF/Tab-Behandlung fuehrt zu ungueltigem JSON wenn das LLM Zeilenumbrueche im reasoning-Feld produziert.
- Entscheidung: Alle Standard-Control-Chars eskapen.

**Finding 5: isDefault bei komplett unerkennbarem Response (MITTEL — implementiert)**
- ChatGPT: Wenn LLM eine Antwort gibt, die kein einziges Dimensions-Feld enthaelt (z.B. Prosatext), wuerden alle Felder auf Defaults fallen und `isDefault = false` zurueckgeben. Semantisch falsch.
- Entscheidung: Vier `matcher.find()`-Vorab-Checks; wenn kein einziges Feld matcht, `buildDefaultProfile()` mit `isDefault = true`.

**Finding 6: Regex-Fragilitaet (DOKUMENTIERT — nicht implementiert)**
- ChatGPT: Regex-Parsing ist anfaellig fuer Edge-Cases (Whitespace-Varianten, verschachtelte Strings, Unicode). Jackson waere robuster.
- Entscheidung: Abgelehnt. Jackson ist keine Compile-Abhaengigkeit in odin-brain (nur LLM-Utilities). Der Aufwand, Jackson einzufuehren, steht ausser Verhaeltnis. Regex genuegt fuer das erwartete LLM-Antwortformat; die bekannte Einschraenkung ist dokumentiert.

**Finding 7: Testabdeckung fuer neue Edge-Cases (MITTEL — implementiert)**
- ChatGPT schlug explizite Tests vor fuer: Reasoning > 200 Zeichen, Sonderzeichen in Payload (EventLog), EventLog wirft Exception.
- Entscheidung: Alle drei Tests implementiert in `InstrumentProfilerTest`.

### Runde 2 — Vertiefung

**Finding: Kein Timeout in InstrumentProfiler selbst**
- ChatGPT: Provider-seitige Timeouts sind nicht in ODIN-018 kontrollierbar. Ein lokaler `Future`/`CompletableFuture`-Wrapper mit fixem Timeout (z.B. 5s) wuerde WARMUP-Haenger verhindern.
- Entscheidung: Abgelehnt fuer ODIN-018. Das wuerde Threading-Komplexitaet einfuehren, die ausserhalb des Story-Scope liegt. Dokumentiert als offener Punkt 3.

**Finding: Nutzung von reasoning fuer Debug/Logging**
- ChatGPT: Das `reasoning`-Feld sollte im DEBUG-Log des `buildProfile()`-Pfads sichtbar sein, nicht nur im EventLog-Payload.
- Entscheidung: Akzeptiert. Das bestehende `LOG.warn()` im Fehlerfall gibt die Exception-Message aus. Fuer den Erfolgsfall ist das reasoning im EventLog-Payload (JSON) sichtbar. Ausreichend fuer Operations.

**Gesamtbewertung ChatGPT:** Code ist strukturell solide. Die CAS- und safeLogProfile-Fixes sind die einzigen kritischen Aenderungen. Alle anderen Findings sind Verbesserungen oder dokumentierte Einschraenkungen.

## Gemini-Review

**Durchgefuehrt:** 3 Dimensionen

### Dimension 1 — Code-Qualitaet

**Assessment: Gut. Zwei Findings, eines kritisch.**

**Finding 1: isDefault-Semantik bei Null-Feldern (KRITISCH — implementiert)**
- Gemini: Wenn das LLM eine Antwort liefert, die zwar Felder enthaelt, aber keine davon erkennbar ist (z.B. `"gap_behavior": "UNKNOWN_VALUE"`), werden alle Felder auf Defaults gesetzt und `isDefault = false` zurueckgegeben. Das ist korrekt per Design-Entscheidung 3. Gemini hinterfragte aber: was passiert wenn GAP_BEHAVIOR_PATTERN matcht, aber `Enum.valueOf()` fehlschlaegt (unrecognized value)? In diesem Fall zaehlt das Feld als "gematcht" (gapMatched = true), aber der Wert ist der Default-Fallback.
- Analyse: Das ist tatsaechlich korrekt. Wenn das Pattern matcht, hat das LLM *versucht*, das Feld zu liefern — auch wenn der Wert unbekannt ist. `isDefault = false` ist semantisch korrekt: Es ist ein echtes LLM-Ergebnis, nur mit unbekanntem Enum-Wert.
- Entscheidung: Kein Code-Change. Das Verhalten ist korrekt und dokumentiert.

**Finding 2: Regex-Parsing Fragilitaet (DOKUMENTIERT — kein Change)**
- Gemini: Identisch mit ChatGPT-Finding 6. Jackson waere robuster.
- Entscheidung: Identische Begruendung. Abgelehnt. Dokumentiert.

**Finding 3: Package-private Methoden per Tests direkt zugreifbar (OK)**
- Gemini bemerkte, dass `buildProfilingUserMessage()`, `parseProfilingResponse()`, `buildDefaultProfile()`, `buildProfilePayload()` alle package-private sind und direkt von Tests aufgerufen werden.
- Assessment: Kein Problem. Package-private statt public ist der richtige Kompromiss (Design-Entscheidung 7 analog). Testbar ohne Reflection, kein oeffentliches API-Leck.

### Dimension 2 — Konzepttreue (Konzept 04/8)

**Assessment: Sehr gut. Keine kritischen Abweichungen.**

**Konzept-Check 04/8.1 — Profiling-Zeitpunkt:** WARMUP-Phase, einmalig pro Tag. Implementiert: `getOrCreate()` mit `AtomicReference`-Cache. Korrekt.

**Konzept-Check 04/8.2 — 5 Dimensionen:** gap_behavior, recovery_tendency, typical_reversal_windows, volatility_profile, trend_persistence. Alle 5 implementiert. Enum-Werte entsprechen exakt dem Konzept.

**Konzept-Check 04/8.2 — Fallback:** LLM-Fehler fuehren zu Default-Profile, Session wird nicht geblockt. Implementiert. Korrekt.

**Konzept-Check 04/8.3 — QuantEngine-Validierung:** EXPLIZIT Out-of-Scope fuer ODIN-018. Im Protokoll dokumentiert. Konzept-konform.

**Konzept-Check 04/8.4 — System-Prompt-Integration:** Nur nicht-default Profile werden injiziert. Korrekt (Design-Entscheidung 6). `buildInstrumentProfileBlock()` gibt leeren String fuer Default-Profile zurueck.

**Konzept-Check Tier-1 Context Window:** Profil landet im System-Prompt (Tier 1 = statischer Kontext). Korrekt laut Konzept-Architektur.

**Einzige Anmerkung:** Konzept spricht von "historischen 20 Tagesbars". `DAILY_BAR_COUNT = 20` ist als package-private Konstante korrekt gesetzt. Die Integrationstests verwenden `createDailyBars(20)`. Konzept-konform.

### Dimension 3 — Praxis und Produktionstauglichkeit

**Assessment: Gut mit drei dokumentierten Einschraenkungen.**

**Praxis-Finding 1: Kein Profiling-Timeout (DOKUMENTIERT — offener Punkt 3)**
- Im Produktionsbetrieb kann ein haengender LLM-Provider den WARMUP blockieren. Der HTTP-Timeout liegt im Provider-Adapter, nicht in InstrumentProfiler. Fuer einen 5-Instrument-Fleet waere ein 10-15s WARMUP-Haenger tolerierbar.
- Entscheidung: Akzeptiert. Dokumentiert.

**Praxis-Finding 2: Catalyst-Event-Staleness (DOKUMENTIERT — offener Punkt 4)**
- Profil basiert auf historischen Tagesbars. Bei Earnings, FDA-Entscheiden etc. am selben Tag koennte das historische Profil irrelevant sein.
- Entscheidung: Akzeptiert. Das Profil beschreibt historisches Normalverhalten. Das Intraday-LLM (Tactical Analysis) hat aktuellen Kontext und kann das ueberstimmen. Dokumentiert.

**Praxis-Finding 3: Fleet-Start Rate-Limiting (DOKUMENTIERT — offener Punkt 5)**
- Bei vielen parallelen Instrumenten starten alle Profiler gleichzeitig. Bei einem ODIN-018-Scope von 2-3 Instrumenten kein Problem. Bei zukuenftiger Skalierung relevant.
- Entscheidung: Akzeptiert fuer aktuellen Scope. Dokumentiert.

**Praxis-Finding 4: EventLog-Payload als non-indexed JSON (OK)**
- Das INSTRUMENT_PROFILE-Event speichert den Payload als JSON-String. Gemini fragte ob dieses Feld indexed sein sollte fuer spaetere Auswertungen.
- Assessment: EventLog ist ein Audit-Log, kein Analyse-Store. JSON-Payload ist fuer menschliche Lesbarkeit und Log-Aggregation ausreichend. Kein Change.

**Gesamtbewertung Gemini:** Implementierung ist produktionstauglich fuer den definierten Scope. Drei Einschraenkungen sind bekannt und bewusst dokumentiert. Kein blocking Finding.
