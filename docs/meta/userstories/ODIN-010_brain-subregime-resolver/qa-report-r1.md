# QA Report — ODIN-010: Subregime-Resolver im Brain-Modul

**Runde:** 1
**Datum:** 2026-02-22
**QS-Agent:** Claude Sub-Agent (QS-Rolle)
**Ergebnis:** FAIL

---

## Zusammenfassung

Die Implementierung von ODIN-010 ist technisch solide, kompiliert fehlerfrei, und alle 365 Unit-Tests (inkl. 49 neuer RegimeResolverTest-Tests) laufen grün. Die ChatGPT-Review-Pflicht wurde durch den Implementation-Agent in 2 Runden erfüllt. Jedoch bestehen zwei Kategorien von Findings, die in Summe zum FAIL führen:

1. **Prozess-FAIL**: Gemini-Review (DoD 2.6, alle 3 Dimensionen) fehlt vollständig im protocol.md — laut QS-Task-Anweisung ist dies ein explizites FAIL-Kriterium.
2. **Code-FAIL**: Zwei KRITISCH-Findings verbleiben nach dem ChatGPT-Review der Implementation-Phase, die durch die QS-eigene ChatGPT-Prüfung neu identifiziert wurden.

---

## DoD-Prüfung: Systematische Checkliste

### 2.1 Code-Qualität

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Implementierung vollständig gemäß Akzeptanzkriterien | PASS | Alle 20 Subregimes implementiert; Enum-Namen abweichend vom Story-AC aber im protocol.md begründet dokumentiert |
| Code kompiliert fehlerfrei | PASS | `mvn compile -pl odin-brain -am` → BUILD SUCCESS |
| Kein `var` | PASS | Keine `var`-Nutzung gefunden |
| Keine Magic Numbers | PASS | Alle Schwellenwerte als `private static final` Konstanten oder in BrainProperties konfigurierbar |
| Records für DTOs | PASS | `ResolvedRegime` als Record implementiert |
| ENUM statt String | PASS | `Regime`, `Subregime` Enums genutzt |
| JavaDoc auf allen public Klassen/Methoden/Attributen | PASS | Vollständiges JavaDoc vorhanden inkl. KPI-Kriterien in Methoden-JavaDoc |
| Keine TODO/FIXME | PASS | Keine verbleibenden TODO/FIXME-Kommentare |
| Code-Sprache: Englisch | PASS | Code und JavaDoc durchgehend Englisch |
| Namespace-Konvention `odin.brain.rules.regime.*` | PASS | Alle 16 neuen Properties korrekt als `odin.brain.rules.regime.*` |
| Keine Abhängigkeit von Fachmodulen | PASS | Nur odin-api Import; kein Spring-Annotation in Pro-Pipeline-POJO |
| `RegimeResolver` modular erweitert | PASS | 5 separate Subregime-Methoden, nicht alles in eine Methode |

### 2.2 Tests — Unit (Surefire)

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Alle 20 Subregimes mit synthetischen Werten | PASS | RegimeResolverTest: 49 Tests, alle 20 Subregimes abgedeckt |
| TREND_UP_STRONG (ADX=45, +DI=30, -DI=10) | HINWEIS | Story-DoD-Benennung "STRONG" = implementiertes "EXHAUSTION"; Test vorhanden aber benötigt vorherigen Bar (declining ADX), nicht statisch ADX=45 |
| TREND_UP_MODERATE (ADX=30) → MATURE | PASS | `resolve_subregime_trendUpMature_adxAboveMatureThreshold` vorhanden |
| RANGE_BOUND_NARROW → MEAN_REVERT | PASS | `resolve_subregime_rangeBoundMeanRevert_ordinaryConditions` vorhanden |
| UNCERTAIN_LOW_VOLUME → LOW_LIQUIDITY | PASS | `resolve_subregime_uncertainLowLiquidity_volumeBelowThreshold` vorhanden |
| Hysterese: Einmalige Signal-Umkehr reicht nicht | PASS | `resolve_confirmationLag_singleBarChangeDoesNotSwitchRegime` |
| Hysterese: Nach 2 Bars → Regime-Wechsel | PASS | `resolve_confirmationLag_twoConsecutiveBarsConfirmChange` |
| Subregime-Wechsel innerhalb Regime: sofort | PASS | `resolve_subregime_immediateChangeWithinSameRegime_noHysteresis` |
| Testklassen-Namenskonvention `RegimeResolverTest` | PASS | Korrekt benannt |
| 365 Tests gesamt grün | PASS | `mvn test -pl odin-brain` → Tests run: 365, Failures: 0, Errors: 0 |

### 2.3 Tests — Integrationstests (Failsafe)

| Kriterium | Status | Befund |
|-----------|--------|--------|
| `RegimeResolverIntegrationTest` existiert | PASS | Datei vorhanden mit 3 Integrationstests |
| Konsistenz Regime/Subregime (parentRegime() == Regime) | PASS | `resolvedRegime_subregimeParentRegimeConsistency_acrossAllRegimes` |
| Vollständiger Datenstrom mit Hysterese | PASS | `fullStream_regimeChangeWithHysteresis_twoBarConfirmation` |
| Alle 20 Subregimes in einem Sessiondurchlauf erreichbar | PASS | `allTwentySubregimesAreReachable_inSyntheticSession` |
| `*IntegrationTest` Namenskonvention (Failsafe) | PASS | Korrekt benannt |
| Integrationstests ausgeführt (`mvn verify`) | NICHT GEPRÜFT | `mvn verify -pl odin-brain` konnte durch Sandbox-Beschränkung nicht ausgeführt werden; Code-Review zeigt keine syntaktischen/semantischen Fehler in den IT-Tests |

### 2.4 Tests — Datenbank

Nicht zutreffend (kein DB-Zugriff). PASS.

### 2.5 Test-Sparring mit ChatGPT

| Kriterium | Status | Befund |
|-----------|--------|--------|
| ChatGPT-Session gestartet mit Code + Tests | PASS | 2 Runden dokumentiert im protocol.md |
| ChatGPT nach Grenzfällen gefragt | PASS | Ja, Boundary-Werte, NaN-Szenarien, DI-Division etc. |
| Relevante Vorschläge bewertet und umgesetzt | PASS | 6 CRITICAL-Fixes in Runde 1, 4 CRITICAL-Fixes in Runde 2 dokumentiert |
| Ergebnis im protocol.md dokumentiert | PASS | ChatGPT Review-Abschnitt vorhanden |
| Mind. 2 Runden | PASS | Runde 1 + Runde 2 dokumentiert |

### 2.6 Review durch Gemini — Drei Dimensionen

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Dimension 1: Code-Review an Gemini übergeben | **FAIL** | Fehlt vollständig im protocol.md |
| Dimension 2: Konzepttreue-Review | **FAIL** | Fehlt vollständig im protocol.md |
| Dimension 3: Praxis-Review | **FAIL** | Fehlt vollständig im protocol.md |
| Findings bewertet und eingearbeitet | **FAIL** | Kein Gemini-Review-Abschnitt im protocol.md |

### 2.7 Protokolldatei

| Kriterium | Status | Befund |
|-----------|--------|--------|
| `protocol.md` existiert | PASS | Vorhanden |
| Working State bei jedem Meilenstein aktualisiert | **FAIL (Minor)** | Kein expliziter "Working State"-Abschnitt mit Checkbox-Liste vorhanden (laut user-story-specification.md Pflichtabschnitt) |
| Design-Entscheidungen dokumentiert | PASS | Gut dokumentiert (Subregime gegen confirmed Regime, UNCERTAIN TRANSITION Proxy, etc.) |
| ChatGPT-Sparring-Abschnitt ausgefüllt | PASS | Vorhanden und detailliert |
| Gemini-Review-Abschnitt ausgefüllt | **FAIL** | Fehlt vollständig |

### 2.8 Abschluss

| Kriterium | Status | Befund |
|-----------|--------|--------|
| Commit mit aussagekräftiger Message | Ausstehend | Noch nicht committed (korrekt — QS muss erst PASS ergeben) |
| Push auf Remote | Ausstehend | Folgt nach PASS |

---

## Findings-Katalog

### [FAIL — Prozess] Gemini-Review fehlt vollständig (DoD 2.6)

- **Schwere:** Major / Process-Fail
- **Beschreibung:** Die DoD-Sektion 2.6 fordert verpflichtend ein Gemini-Review in drei Dimensionen (Code-Review, Konzepttreue, Praxis). Das protocol.md enthält keinen "Gemini-Review"-Abschnitt. Weder die Findings noch deren Bewertung wurden dokumentiert. Laut QS-Task-Anweisung ist fehlender Gemini-Review ein explizites FAIL-Kriterium.
- **Betroffene Datei:** `T:/codebase/its_odin/temp/userstories/ODIN-010_brain-subregime-resolver/protocol.md`
- **Aktion:** Implementierungs-Agent muss Gemini-Review in allen 3 Dimensionen nachholen und im protocol.md dokumentieren.

### [FAIL — Bug] NaN in `volumeRatio` führt zu falscher Subregime-Klassifikation in RANGE_BOUND

- **Schwere:** KRITISCH (Silent Misclassification)
- **Beschreibung:** In `resolveRangeBoundSubregime()` wird `volumeRatio` nicht auf NaN geprüft. Wenn `volumeRatio` NaN ist, ergeben `nearBoundary && volumeElevated` und `volumeRatio < config.rangeChopVolumeRatioMax()` beide `false`, was zum Default-Fallback `MEAN_REVERT` führt — statt zu `DATA_QUALITY` oder `CHOP`. Das System klassifiziert fehlerhaft ohne Fehlerindikation.
- **Betroffene Datei:** `RegimeResolver.java`, Methode `resolveRangeBoundSubregime()` (Zeile 492-533)
- **Aktion:** `volumeRatio` auf `Double.isFinite()` prüfen; bei nicht-finitem Wert defensiver Fallback.

### [FAIL — Bug] `previousAdx` wird mit NaN überschrieben — State-Korruption über mehrere Bars

- **Schwere:** KRITISCH (State-Drift)
- **Beschreibung:** In `resolve()` Zeile 139: `previousAdx = indicators.adx14_5m();` — dieser Wert wird auch geschrieben wenn ADX NaN ist. Danach ist `adxIsDeclining` für die Folge-Bar wirkungslos (NaN-Guard greift), und die LATE/EXHAUSTION-Erkennung fällt dauerhaft aus bis ein gültiger ADX-Wert kommt. Bei realen NaN-Glitches (Provider-Fehler, Sessionsübergänge) kann dies die TREND-Phase-Klassifikation über mehrere Minuten verfälschen.
- **Betroffene Datei:** `RegimeResolver.java`, Zeile 139 in `resolve()`
- **Fix:** `previousAdx` nur updaten wenn `Double.isFinite(indicators.adx14_5m())`.

### [FAIL — Prozess] `protocol.md` fehlt pflichtgemäßer "Working State"-Abschnitt

- **Schwere:** Minor (Prozess)
- **Beschreibung:** Laut `user-story-specification.md` Abschnitt 2.7 ist ein "Working State"-Abschnitt mit Checkbox-Liste Pflicht im protocol.md. Der aktuelle Stand enthält stattdessen eine freie Zusammenfassung.
- **Betroffene Datei:** `protocol.md`
- **Aktion:** Working State Checkbox-Liste gemäß user-story-specification.md Abschnitt 2.7 nachtragen.

### [WICHTIG — Verbesserung] Null-Preconditions an öffentlicher API fehlen

- **Schwere:** WICHTIG
- **Beschreibung:** `resolve(IndicatorResult indicators, ...)` prüft `indicators` nicht auf null (NPE-Risiko). `config` im Konstruktor ebenfalls ohne `Objects.requireNonNull`.
- **Betroffene Datei:** `RegimeResolver.java`, Konstruktor und `resolve()`
- **Aktion:** `Objects.requireNonNull()` in Konstruktor und `resolve()` hinzufügen.

### [WICHTIG — Design] `LLM_RELEVANCE_THRESHOLD` ist hardcoded

- **Schwere:** WICHTIG
- **Beschreibung:** `LLM_RELEVANCE_THRESHOLD = 0.5` ist als Konstante definiert, ist aber fachlich ein Tuning-Parameter der in BrainProperties konfigurierbar sein sollte (analog zu allen anderen Schwellenwerten).
- **Betroffene Datei:** `RegimeResolver.java`, `BrainProperties.java`
- **Aktion:** In `RegimeProperties` als konfigurierbares Feld aufnehmen.

---

## QS-eigene ChatGPT-Review (2 Runden)

Als Teil dieser QS-Prüfung wurde eine eigene ChatGPT-Begutachtung durchgeführt:

**Runde 1 — Code-Review:**
- KRITISCH: NaN in `volumeRatio` → silent MEAN_REVERT in RANGE_BOUND (oben beschrieben)
- KRITISCH: `Math.max(di, DI_EPSILON)` schützt nicht gegen NaN (NaN propagiert durch Math.max)
- KRITISCH: `previousAdx = indicators.adx14_5m()` überschreibt auch mit NaN
- WICHTIG: Null-Preconditions fehlen an öffentlicher API
- WICHTIG: `LLM_RELEVANCE_THRESHOLD` sollte konfigurierbar sein
- HINWEIS: `ResolvedRegime` als Record passend; Modularisierung gut

**Runde 2 — Gesamtbewertung:**
- ChatGPT-Empfehlung: **FAIL** — zwei unabhängige Gründe: (1) fehlendes Gemini-Review (DoD 2.6), (2) verbleibende KRITISCH-Findings (NaN-Gates unvollständig, State-Korruption bei NaN-ADX)

---

## Bewertung und Empfehlung

**Gesamtbewertung: FAIL**

Zwei unabhängige FAIL-Ursachen:

1. **Prozess-Fail**: Das Gemini-Review (DoD 2.6) wurde vom Implementation-Agent nicht durchgeführt und ist im protocol.md nicht dokumentiert. Laut QS-Task-Anweisung ist dies ein explizites FAIL-Kriterium.

2. **Code-Fail**: Zwei KRITISCH-Bugs verbleiben nach dem Implementation-Agent-Review:
   - NaN-Durchschlag in `resolveRangeBoundSubregime()` → silent Misclassification
   - State-Korruption durch NaN in `previousAdx`-Update → dauerhafter Drift der Trend-Phase-Erkennung

Die Basisimplementierung ist gut strukturiert und hat durch die 2 ChatGPT-Runden des Implementation-Agents bereits viele wichtige Bugs behoben. Der Code ist modular, JavaDoc vollständig, keine Magic Numbers, alle Tests grün.

**Für PASS in Runde 2 benötigt:**
1. Gemini-Review durchführen (alle 3 Dimensionen) und im protocol.md dokumentieren
2. NaN-Guard in `resolveRangeBoundSubregime()` für `volumeRatio`
3. `previousAdx` nur bei `Double.isFinite(adx)` updaten
4. `Objects.requireNonNull` in Konstruktor und `resolve()` (WICHTIG)
5. Working State-Abschnitt in protocol.md nachtragen
