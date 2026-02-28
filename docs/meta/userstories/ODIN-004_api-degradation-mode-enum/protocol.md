# Protokoll: ODIN-004 — Degradation-Mode-Enum und System-Health-Model

## Working State
- [x] Story + Konzepte gelesen
- [x] Bestehenden Code analysiert
- [x] Initiale Implementierung (Round 1)
- [x] Unit-Tests geschrieben (18 Tests nach Round 2, war 12)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases (Round 2)
- [x] Integrationstests geschrieben (DegradationModeIntegrationTest.java, 8 Tests)
- [x] Gemini-Review Dimension 1 (Code) — Round 2
- [x] Gemini-Review Dimension 2 (Konzepttreue) — Round 2
- [x] Gemini-Review Dimension 3 (Praxis) — Round 2
- [x] Review-Findings eingearbeitet
- [x] Kompilierung erfolgreich (mvn compile -pl odin-api EXIT 0)
- [x] Unit-Tests grun (mvn test: 83 Tests, 0 Failures, 0 Errors)
- [x] Integrationstests kompiliert und strukturell korrekt (Failsafe-Plugin aktiviert)
- [x] Commit & Push
- [x] QS-Round-2 bestanden (2026-02-21) — qa-report-r2.md: PASS

## Design-Entscheidungen

### 1. Funf Modi: NORMAL, DEGRADED_EXECUTION, QUANT_ONLY, DATA_HALT, EMERGENCY

Konzept 11-edge-cases.md Abschnitt 6.1 listet explizit 5 Modi in der Degradation-Tabelle.
Der erste Implementierungsversuch (Round 1) hatte nur 4 Modi. DEGRADED_EXECUTION wurde in
Round 2 hinzugefuegt. Konzept 06-risk-management.md Abschnitt 6 beschreibt nur 3 degradierte
Modi (kein DEGRADED_EXECUTION), da es das neuere und detailliertere 11-edge-cases.md Dokument
nicht einschliesst. Das normative Dokument fuer die vollstaendige Liste ist 11-edge-cases.md
Abschnitt 6.1 (5 Modi in der Degradation-Tabelle).

### 2. DEGRADED_EXECUTION(true, true) — Entries und Exits erlaubt

Repricing-Degradation ist kein vollstaendiger Stopp des Handels. Das Konzept beschreibt nur
eine Drosselung (Repricing-Intervall erhoeht, Zyklen reduziert). Daher: allowsNewEntries=true,
allowsExits=true. DEGRADED_EXECUTION ist semantisch unterschiedlich von NORMAL, hat aber
dieselben booleschen Flags. Der Unterschied liegt im Verhalten von odin-core und odin-execution,
nicht in der API-Schicht.

### 3. allowsExits=true fuer alle Modi

Gemini hatte vorgeschlagen, ein ExitPolicy-Enum einzufuehren (ALL vs RISK_REDUCING_ONLY).
Entscheidung: Boolean reicht fuer die API-Schicht. Die semantische Einschraenkung
(DATA_HALT nur risk-reducing Exits) wird vom Caller (RiskGate) durchgesetzt.
JavaDoc dokumentiert dies ausdruecklich. YAGNI-Prinzip — kein Over-Engineering in der API-Schicht.

### 4. DegradationMode orthogonal zu PipelineState (mit Einschraenkung)

Eine Pipeline kann POSITIONED sein waehrend das System in QUANT_ONLY ist.
Die beiden Zustandsmodelle sind grundsaetzlich unabhaengig. Ausnahme (dokumentiert):
kritische Modi (EMERGENCY, persistenter DATA_HALT) setzen PipelineState zwingend auf
DAY_STOPPED. Dies wird von odin-core durchgesetzt, nicht vom API-Modell selbst.
JavaDoc auf der Klasse wurde entsprechend praezisiert.

### 5. SystemHealthState: Null-Safety fuer mode und since

Gemini und ChatGPT empfahlen Objects.requireNonNull fuer mode und since.
Umgesetzt: Compact Constructor wirft NullPointerException bei null-mode oder null-since.
Entsprechende Tests ergaenzt (nullMode_throwsNpe, nullSince_throwsNpe in Unit und Integration Tests).

## Offene Punkte

### OP-1: Mode-Prioritaet bei gleichzeitigen Ausfaellen (Gemini Dimension 3)

Gemini hat auf einen konzeptuellen Gap hingewiesen: Was passiert, wenn das System
gleichzeitig in QUANT_ONLY ist (LLM-Ausfall) und der Data-Feed ausfaellt (DATA_HALT)?
Das Konzept beschreibt sequenzielle Kaskaden, nicht gleichzeitige Ausfaelle.
Ein Mode-Prioritaets-Matrix (EMERGENCY > DATA_HALT > QUANT_ONLY > DEGRADED_EXECUTION > NORMAL)
muss von odin-core implementiert werden. Scope dieser Story (ODIN-004): nur API-Modell.
Implementierung liegt bei ODIN-026 (core-degradation-mode-fsm).
**Keine Eskalation an Stakeholder notwendig — liegt in ODIN-026's Scope.**

### OP-2: since-Timestamp bei Mode-Flapping (Gemini Dimension 3)

Gemini hat auf das Problem des State-Flappings hingewiesen: Bei intermittierendem
Data-Feed koennte das System schnell zwischen NORMAL und DATA_HALT wechseln.
Der since-Timestamp in SystemHealthState muss dann korrekt den Eintrittszeitpunkt
in den aktuellen kontinuierlichen Mode repraesentieren, nicht jeden Micro-Transition-Reset.
Dies erfordert eine Debouncing/Hysterese-Logik in odin-core.
**Scope von ODIN-026. Keine Aenderung am API-Modell notwendig.**

### OP-3: activeAlerts als strukturierte Typen (Gemini Dimension 3)

Gemini empfiehlt, activeAlerts von List<String> zu List<SystemAlert> aufzuwerten,
wobei SystemAlert einen AlertSeverity-Enum und eine ID enthielte.
Entscheidung: ABGELEHNT fuer ODIN-004. Scope ist das Mindest-API-Modell.
Strukturierte Alerts waeren ein separates Feature (ggf. ODIN-005 oder spaetere Phase).
Strings sind fuer das aktuelle Ziel (Frontend-Anzeige, RiskGate-Logging) ausreichend.

### OP-4: Rueckgabetyp von allowsExits() bei DATA_HALT

Der Ruckgabewert true fuer allowsExits() bei DATA_HALT verschleiert die
"risk-reducing only"-Einschraenkung. Die aktuelle Loesung ist akzeptabel (Caller-Enforcement,
JavaDoc), aber zukuenftige Implementierer des RiskGate muessen diese Semantik explizit
beachten. Empfehlung fuer zukuenftige Review: ob ein `isRiskReducingOnly()`-Methode
Wert haette ohne die boolean-API zu brechen.

## ChatGPT-Sparring

**Session:** impl-odin-004-chatgpt (Round 2, 2026-02-21)

### Kontext an ChatGPT gesendet
- DegradationMode.java, SystemHealthState.java, DegradationModeTest.java
- Geplante Aenderungen: DEGRADED_EXECUTION, null-safety, IntegrationTests

### Vorschlaege von ChatGPT

**Q1: Unit-Test Edge Cases**

1. **degradedExecution_isDistinctFromNormal()** — umgesetzt: DEGRADED_EXECUTION darf nicht
   equals NORMAL sein, trotz gleicher boolean-Flags.
2. **valueOf_acceptsExpectedNames()** — umgesetzt: API-Stabilitaets-Guard gegen Umbenennung.
3. **modeFlagMatrix_matchesSpec()** — umgesetzt: EnumSet-basierter Matrixtest als
   Single-Source-of-Truth.
4. **allowsNewEntries_impliesAllowsExits()** — umgesetzt: Domain-Invariant als Test.
5. **systemHealthState_nullMode_throwsNpe()** — umgesetzt.
6. **systemHealthState_nullSince_throwsNpe()** — umgesetzt.
7. **Sweep-Test anpassen** fuer DEGRADED_EXECUTION — umgesetzt:
   onlyNormalAllowsNewEntries() → modeFlagMatrix_matchesSpec() (EnumSet-basiert).
8. **List.copyOf() wirft NPE bei null-Elementen** — dokumentiert: kein separater Test
   umgesetzt (Aufwand > Nutzen fuer V1; Upstream-Caller-Verantwortung).

**Q2: Integration Test Szenarien**

1. **Golden matrix fuer alle 5 Modi** — umgesetzt als goldenMatrix_allModes_correctFlagCombinations().
2. **NORMAL mit active alerts, Entries weiterhin erlaubt** — umgesetzt als normal_withActiveAlerts_stillPermitsEntries().
3. **DEGRADED_EXECUTION repricing-throttle path** — umgesetzt innerhalb der Golden Matrix.
4. **QUANT_ONLY → NORMAL Recovery** — umgesetzt als quantOnlyToNormal_recovery_entriesRestored().
5. **DATA_HALT since-Timestamp Prazision** — umgesetzt als dataHalt_sinceTimestamp_preservedExactly().
6. **Constructor contract NPE** — umgesetzt als constructorContract_nullModeAndNullSince_throwNpe().
7. **Zwei States aus derselben mutablen Liste** — umgesetzt als twoStates_fromSameMutableList_areIndependent().
8. **Record Equality Semantik** — umgesetzt als equalStates_withSameFields_areEqual().

### Bewertung
ChatGPT-Feedback war gut und praxisorientiert. Alle 8 Unit-Test-Vorschlaege und alle
8 Integration-Test-Vorschlaege wurden umgesetzt oder bewusst abgelehnt (mit Begruendung).
Die null-Element-in-List Randbedingung wurde als Out-of-Scope eingestuft (kein Test umgesetzt).

## Gemini-Review

**Session:** impl-odin-004-gemini (Round 2, 2026-02-21)

### Dimension 1: Code-Review (Bugs & Implementierungsfehler)

**Finding 1.1 (IMPORTANT): Fehlende Null-Safety in SystemHealthState**
- mode und since hatten keine requireNonNull-Checks
- **Umgesetzt:** Objects.requireNonNull(mode, "mode must not be null") und
  Objects.requireNonNull(since, "since must not be null") in Compact Constructor eingefuegt
- Entsprechende Tests hinzugefuegt (nullMode_throwsNpe, nullSince_throwsNpe)

**Finding 1.2 (MINOR): Semantische Ambiguitat von allowsExits() bei DATA_HALT**
- allowsExits() = true fuer alle Modi macht den boolean redundant und verschleiert
  die "risk-reducing only" Semantik fuer DATA_HALT
- **Entscheidung: Abgelehnt** (ExitPolicy-Enum wuerde odin-api-Schicht ueberengineeren).
  Caller-Enforcement + JavaDoc ist ausreichend fuer V1. Vermerkt unter Offene Punkte OP-4.

**Finding 1.3 (MINOR): Testluecken wenn null-safety nachgeruestet wird**
- **Umgesetzt:** Tests in DegradationModeTest und DegradationModeIntegrationTest ergaenzt.

**Gesamtbewertung Dimension 1:** 1 Finding umgesetzt, 1 abgelehnt (begruendet), 1 via Tests adressiert.

### Dimension 2: Konzepttreue-Review

**Finding 2.1: boolean-Flags korrekt per Konzept**
- NORMAL(true,true), QUANT_ONLY(false,true), DATA_HALT(false,true), EMERGENCY(false,true):
  Alle vier Modes stimmen mit 06-risk-management.md und 11-edge-cases.md ueberein.
- **Kein Handlungsbedarf.**

**Finding 2.2: DATA_HALT allowsExits() semantisch schwach**
- Gleiche Diskussion wie Dimension 1.2. Gemini empfiehlt ExitPolicy-Enum.
- **Entscheidung: Abgelehnt** (identisch zu 1.2). Vermerkt unter OP-4.

**Finding 2.3 (KRITISCH): DEGRADED_EXECUTION fehlt**
- 11-edge-cases.md Abschnitt 6.1 listet explizit 5 Modi. DEGRADED_EXECUTION war nicht
  implementiert.
- **Umgesetzt:** DEGRADED_EXECUTION(true, true) hinzugefuegt. JavaDoc mit vollstaendiger
  Beschreibung (Repricing-Throttling, Recovery-Bedingung, Konzept-Verweis).
- Alle Tests aktualisiert (hasFiveModes, modeFlagMatrix_matchesSpec, valueOf).

**Gesamtbewertung Dimension 2:** Kritisches Finding (DEGRADED_EXECUTION) umgesetzt.
Konzepttreue hergestellt.

### Dimension 3: Praxis-Review (unbehandelte Themen)

Gemini identifizierte 5 praxisrelevante Themen, die im Konzept nicht vollstaendig behandelt sind:

**Thema 3.1: Mode-Prioritaet bei gleichzeitigen Ausfaellen**
Keine Prioritaets-Matrix definiert. → Vermerkt unter OP-1. Scope ODIN-026.

**Thema 3.2: "Orthogonalitaet" von DegradationMode und PipelineState**
Nicht vollstaendig orthogonal: EMERGENCY erzwingt DAY_STOPPED. → JavaDoc praezisiert.
Kein Aenderungsbedarf am Enum selbst. Scope ODIN-026.

**Thema 3.3: since-Timestamp bei State-Flapping**
Debouncing/Hysterese-Anforderung fuer odin-core. → Vermerkt unter OP-2. Scope ODIN-026.

**Thema 3.4: activeAlerts als strukturierte Typen**
List<SystemAlert> mit AlertSeverity waere sauberer. → Abgelehnt (Scope ODIN-004).
Vermerkt unter OP-3.

**Thema 3.5: Asymmetrische Recovery-Pfade**
QUANT_ONLY kann auto-recovern, QUANT_ONLY nach Halluzinations-Kaskade nicht.
SystemHealthState bildet dies nicht ab (kein isAutoRecoverable).
Entscheidung: Aus Scope (API-Modell). odin-core entscheidet Recovery-Logik.
Vermerkt implizit in DegradationMode JavaDoc (Hinweis auf Halluzinations-Kaskade).

**Gesamtbewertung Dimension 3:** 5 praxisrelevante Themen identifiziert.
2 direkt im Code adressiert (JavaDoc-Praezisierung), 3 unter Offene Punkte dokumentiert.
Keine Eskalation an Stakeholder erforderlich (alle liegen in ODIN-026's Scope).

## Dateien

### Geaendert (Round 2)
- `odin-api/src/main/java/de/its/odin/api/model/DegradationMode.java` — DEGRADED_EXECUTION hinzugefuegt, JavaDoc praezisiert
- `odin-api/src/main/java/de/its/odin/api/dto/SystemHealthState.java` — Null-Safety (mode, since) ergaenzt
- `odin-api/src/test/java/de/its/odin/api/model/DegradationModeTest.java` — 18 Tests (war 12), neue Tests fuer DEGRADED_EXECUTION und Null-Safety

### Neu erstellt (Round 2)
- `odin-api/src/test/java/de/its/odin/api/model/DegradationModeIntegrationTest.java` — 8 Integrationstests (Failsafe)
