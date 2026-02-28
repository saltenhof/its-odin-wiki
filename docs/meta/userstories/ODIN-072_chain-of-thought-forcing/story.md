# ODIN-072: Chain-of-Thought Forcing

## Modul

odin-brain

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

S

## Kontext

Studien zeigen +5 bis +17% Alpha durch Chain-of-Thought (CoT) gegenueber direkten Vorhersagen (Kim et al. 2024). ODINs `reasoning`- und `key_observations`-Felder im `LlmTacticalOutput` existieren bereits, werden aber nur als Logging-Only-Felder genutzt. Der System-Prompt weist den LLM nicht explizit an, diese Felder als Denkprozess VOR den Decision-Feldern zu befuellen. Diese Story aendert den Prompt, um CoT-Forcing zu implementieren — minimaler Aufwand, messbarer Qualitaetsgewinn.

## Scope

### In Scope

- **Prompt-Aenderung in `PromptBuilder.buildSystemPrompt()`:** Explizite Anweisung, `reasoning`-Feld als schrittweisen Denkprozess zu verwenden (Kontext -> Indikatoren -> Muster -> Regime -> Entscheidung)
- **Prompt-Version-Increment:** Neue Version triggert Cache-Invalidierung (falls Prompt-Caching aktiv)
- **Schema-Reihenfolge im Prompt:** `reasoning` und `key_observations` vor `action`, `regime`, `confidenceScore` auflisten
- **Validierung:** Unit-Test prueft, dass der generierte System-Prompt die CoT-Anweisung enthaelt

### Out of Scope

- Aenderungen am `LlmTacticalOutput`-Record (keine neuen Felder)
- Aenderungen am LLM-Response-Parser
- Aenderungen an der LlmAnalystOrchestrator-Logik
- A/B-Test-Framework fuer CoT vs. Non-CoT (wird ueber Backtest-Vergleich manuell gemacht)
- Multi-Step-CoT mit separaten API-Calls (nur Prompt-Level CoT)

## Akzeptanzkriterien

- [ ] System-Prompt enthaelt explizite CoT-Anweisung: "Fill the reasoning field FIRST with step-by-step analysis before populating any decision fields"
- [ ] System-Prompt definiert CoT-Schritte: (1) Current market context, (2) Key indicator readings, (3) Pattern recognition, (4) Regime assessment, (5) Decision with rationale
- [ ] JSON-Schema im Prompt listet `reasoning` und `key_observations` VOR `action` und `confidenceScore`
- [ ] Prompt-Version ist inkrementiert (gegenueber aktuellem Wert)
- [ ] Unit-Test verifiziert: `PromptBuilder.buildSystemPrompt()` enthaelt "reasoning field FIRST" oder aequivalente CoT-Anweisung
- [ ] Unit-Test verifiziert: Im generierten Schema erscheint `reasoning` vor `action`
- [ ] Bestehende `PromptBuilderTest`-Tests laufen weiterhin gruen (keine Regression)

## Technische Details

### Bestehende Klassen (Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `PromptBuilder` (`de.its.odin.brain.llm.PromptBuilder`) | `buildSystemPrompt()` erweitern: (1) CoT-Anweisung einfuegen, (2) Schema-Reihenfolge anpassen, (3) Prompt-Version inkrementieren |

### Prompt-Aenderung (Inhalt)

Im System-Prompt nach der Role-Definition und vor dem Schema einfuegen:

```
REASONING PROTOCOL (mandatory):
Before filling ANY decision fields, you MUST complete the reasoning field with
a step-by-step analysis following this exact sequence:
1. CONTEXT: Current session state, time-of-day, recent price action
2. INDICATORS: Key readings (RSI, ADX, VWAP position, volume ratio, ATR)
3. PATTERNS: Active patterns (opening consolidation, breakout, exhaustion)
4. S/R LEVELS: Relevant support/resistance levels and proximity
5. REGIME: Regime assessment with evidence (not just label)
6. DECISION: Action recommendation with explicit rationale

The reasoning field drives your decision — do not skip steps or fill decision
fields before completing the full analysis chain.
```

### Schema-Reihenfolge

Aktuelle Reihenfolge im Schema (vermutlich):
```json
{ "action": ..., "regime": ..., "confidenceScore": ..., "reasoning": ..., "key_observations": ... }
```

Neue Reihenfolge:
```json
{ "reasoning": ..., "key_observations": ..., "action": ..., "regime": ..., "confidenceScore": ... }
```

### Konfiguration

Keine neue Konfiguration erforderlich. Die Prompt-Aenderung ist nicht konfigurierbar — CoT ist immer aktiv (kein Feature-Flag noetig, da es keine negativen Nebenwirkungen hat).

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 10: Chain-of-Thought-Forcing im System-Prompt, Abschnitte IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/05-llm-integration.md` — Prompt-Architektur, Tier-1 System-Prompt, Prompt-Versionierung

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, JavaDoc-Pflicht

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] JavaDoc auf allen modifizierten public Methoden (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Test: System-Prompt enthaelt CoT-Anweisung
- [ ] Unit-Test: Schema-Reihenfolge korrekt (reasoning vor action)
- [ ] Bestehende PromptBuilderTest-Tests laufen gruen
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstest: PromptBuilder generiert vollstaendigen Prompt mit CoT (End-to-End mit realen Parametern)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit neuem Prompt-Text und Akzeptanzkriterien
- [ ] ChatGPT nach Prompt-Formulierungs-Optimierungen gefragt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (String-Handling, keine Regression)
- [ ] Dimension 2: Konzepttreue-Review (CoT-Best-Practices, Kim et al. 2024)
- [ ] Dimension 3: Praxis-Review (Prompt-Laenge, Token-Budget-Impact)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Minimaler Code-Change:** Diese Story ist primaer eine Prompt-Text-Aenderung. Es gibt keine neuen Klassen, keine Architekturanpassungen. Der Schwerpunkt liegt auf der Qualitaet des Prompt-Textes.
- **Prompt-Version:** `PromptBuilder` hat vermutlich eine Versions-Konstante (z.B. `PROMPT_VERSION = "1.x"`). Diese muss inkrementiert werden, damit ggf. aktives Prompt-Caching (ODIN-083) invalidiert wird.
- **Schema-Reihenfolge:** JSON-Objekte haben technisch keine garantierte Reihenfolge. Aber: Empirisch folgen LLMs der Reihenfolge im Schema-Template. Daher `reasoning` vor `action` auflisten.
- **Keine Token-Budget-Erhoehung:** CoT nutzt die existierenden `reasoning`- und `key_observations`-Felder. Der Output wird nicht laenger — er wird nur strukturierter. Die max_tokens-Konfiguration muss nicht angepasst werden.
- **Validierung:** Da es keine Code-Logik-Aenderung gibt, ist der Unit-Test ein String-Contains-Check. Der echte Validierungswert kommt aus dem Backtest-Vergleich (vorher/nachher), der aber nicht Teil dieser Story ist.
