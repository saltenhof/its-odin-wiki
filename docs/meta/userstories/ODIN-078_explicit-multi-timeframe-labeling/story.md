# ODIN-078: Explicit Multi-Timeframe Labeling

**Modul:** odin-brain
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** S

---

## Kontext

Der LLM-Prompt enthaelt OHLCV-Bars aus verschiedenen Timeframes (1m, 5m, Daily), die aber ohne explizite Hierarchie-Beschriftung uebergeben werden. Prompt-Engineering-Forschung zeigt, dass explizite Strukturierung mit klaren Abschnitts-Labels ("=== MICRO CONTEXT (1m) ===", "=== DECISION CONTEXT (5m) ===") das LLM-Verstaendnis messbar verbessert. Die Aenderung ist rein kosmetisch im Prompt-Template — keine Aenderungen an der Code-Architektur oder Datenstrukturen.

## Scope

**In Scope:**

- Anpassung von `PromptBuilder.buildUserMessage()` fuer explizite Timeframe-Section-Labels
- Prompt-Version-Increment (triggert Cache-Invalidierung)
- Hierarchische Beschriftung: MACRO (Daily), DECISION (5m), MICRO (1m), TICK (Echtzeit-Werte)

**Out of Scope:**

- Aenderungen an den uebergebenen Datenpunkten (Anzahl Bars, Indikatorwerte bleiben gleich)
- Neuer Timeframe (z.B. 15m) — nur Labeling der bestehenden Timeframes
- Aenderungen am System-Prompt (Tier 1) — nur User Message (Tier 2+3)
- Messung des LLM-Verstaendnis-Effekts (kein A/B-Test in dieser Story)

## Akzeptanzkriterien

- [ ] User Message im LLM-Prompt enthaelt explizite Section-Labels:
  - `=== DAILY BIAS (1D) ===` fuer Tages-Kontext (Vortageshoch/-tief, Daily-Bars)
  - `=== DECISION CONTEXT (5m) ===` fuer 5-Minuten-Bars und deren Indikatoren
  - `=== MICRO CONTEXT (1m) ===` fuer 1-Minuten-Bars
  - `=== CURRENT TICK ===` fuer Echtzeit-Werte (Last Price, Bid/Ask, VWAP, Volume, Spread)
- [ ] Jede Section enthaelt eine einzeilige Beschreibung ihres Zwecks:
  - Daily: "Prior-day reference levels and multi-day trend context."
  - Decision: "Primary analysis timeframe. Entry/exit signals operate on this scale."
  - Micro: "Short-term price action for execution timing and pattern confirmation."
  - Tick: "Real-time market state at decision point."
- [ ] Prompt-Version in `BrainProperties.LlmProperties.promptVersion` wird inkrementiert (z.B. "1.1.0" → "1.2.0")
- [ ] Bestehende Unit-Tests in `PromptBuilderTest` werden angepasst (neue Labels in Expected-Output)
- [ ] Neuer Unit-Test verifiziert: alle 4 Section-Labels sind im generierten User Message enthalten
- [ ] Neuer Unit-Test verifiziert: Section-Reihenfolge ist DAILY → DECISION → MICRO → TICK (makro-zu-mikro)
- [ ] S/R-Level-Section (falls vorhanden) wird mit eigenem Label `=== SUPPORT / RESISTANCE LEVELS ===` versehen
- [ ] Opening-Pattern-Section (falls vorhanden) wird mit `=== OPENING PATTERN ===` versehen

## Technische Details

### Bestehende Klassen (werden geaendert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `PromptBuilder` | `odin-brain/.../llm/PromptBuilder.java` | Section-Labels in `buildUserMessage()`, ggf. `buildTier2Context()` und `buildTier3Context()` |
| `PromptBuilderTest` | `odin-brain/.../llm/PromptBuilderTest.java` | Anpassung der Expected-Strings an neue Labels |

### Keine neuen Klassen

Diese Story ist eine reine Prompt-Template-Aenderung in bestehenden Methoden.

### Prompt-Struktur (SOLL)

```
=== DAILY BIAS (1D) ===
Prior-day reference levels and multi-day trend context.
Prior Day: High=42.50 Low=40.20 Close=41.80
[... daily bars ...]

=== DECISION CONTEXT (5m) ===
Primary analysis timeframe. Entry/exit signals operate on this scale.
[... 5m bars with indicators ...]

=== MICRO CONTEXT (1m) ===
Short-term price action for execution timing and pattern confirmation.
[... 1m bars ...]

=== SUPPORT / RESISTANCE LEVELS ===
[... S/R zones with confidence ...]

=== OPENING PATTERN ===
[... opening pattern result ...]

=== CURRENT TICK ===
Real-time market state at decision point.
Last=41.95 Bid=41.94 Ask=41.96 VWAP=41.72 Volume=1234567 Spread=0.05%
```

### Konfiguration

Keine neuen Properties noetig. Prompt-Version-Increment in bestehender Property:
```properties
odin.brain.llm.prompt-version=1.2.0
```

## Konzept-Referenzen

- `theme-backlog.md` Thema 16: "Explicit Multi-Timeframe-Labeling im LLM-Prompt" — IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/05-llm-integration.md` Abschnitt 6: 3-Tier Context Window, Prompt-Aufbau
- Research-LLM Empfehlung 5: Explizite Strukturierung verbessert LLM-Verstaendnis

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints
- `CLAUDE.md` — Coding-Regeln (englische Strings, keine Magic Numbers)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten fuer Labels
- [ ] JavaDoc auf geaenderten Methoden aktualisiert
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Bestehende `PromptBuilderTest` Tests angepasst
- [ ] Neue Tests fuer Section-Labels und Reihenfolge
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Bestehende Integrationstests verifizieren, dass Prompt-Generierung weiterhin funktioniert
- [ ] Mindestens 1 Integrationstest: Vollstaendiger Prompt mit allen Sections

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Prompt-Template und Akzeptanzkriterien
- [ ] ChatGPT nach optimaler Section-Reihenfolge und Label-Formulierung gefragt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (String-Handling, Encoding)
- [ ] Dimension 2: Konzepttreue-Review (Labels matchen Thema 16)
- [ ] Dimension 3: Praxis-Review (Token-Overhead durch Labels, LLM-Kompatibilitaet)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Reine Prompt-Aenderung:** Keine neuen Klassen, keine neuen Abhaengigkeiten, keine Architektur-Aenderungen. Der Grossteil der Arbeit liegt im sorgfaeltigen Refactoring der String-Concatenation in `PromptBuilder`.
- **Token-Overhead:** Die Section-Labels addieren ca. 50-80 Tokens zum Prompt. Bei einem Gesamtbudget von ~2000 Tokens Antwort und ~4000 Tokens Prompt ist das vernachlaessigbar.
- **Prompt-Version-Increment:** Wenn ODIN Prompt-Caching nutzt (Thema 9, separate Story ODIN-083), invalidiert ein Version-Increment den Cache. Das ist gewollt — der neue Prompt soll vollstaendig zum LLM geschickt werden.
- **Label-Style:** Die `===` Syntax (Markdown-Section-Header-Style) ist bewusst gewaehlt, weil sie von LLMs gut als Strukturierungs-Element erkannt wird. Alternative waeren XML-Tags (`<section name="...">`), aber die `===` Variante ist in der Forschung besser belegt.
