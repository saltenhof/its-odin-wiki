# ODIN-083: Prompt Caching (Agent SDK)

## Modul

odin-brain

## Phase

1

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

S

## Kontext

ODINs `ClaudeProviderClient` sendet den vollstaendigen Prompt (Tier 1 System-Prompt + Tier 2/3 Dynamic Context) bei jedem API-Call. Anthropic bietet natives Prompt-Caching: der fixe System-Prompt (~2.000 Tokens) wird gecacht, nur dynamische Teile werden pro Call neu berechnet. Cache lebt 5 Minuten und wird bei aktiven Calls automatisch refresht. Bei 78+ Calls pro Instrument pro Tag summieren sich ~30-40% Kostenreduktion plus Latenzverbesserung. ODIN nutzt aber nicht die HTTP-API direkt, sondern den Claude CLI (Agent SDK) — daher muss zuerst geprueft werden, ob Prompt-Caching ueber diesen Weg ueberhaupt aktivierbar ist.

## Scope

### In Scope

- **Machbarkeitspruefung:** Kann Prompt-Caching ueber den `claude` CLI / Agent SDK aktiviert werden?
- **Wenn ja:** `cache_control: {type: "ephemeral"}` im System-Prompt-Block aktivieren
- **Wenn nein:** Story wird als "nicht umsetzbar mit aktuellem SDK" geschlossen, Alternative dokumentieren (z.B. direkter HTTP-Client mit API-Key)
- **Kostenreduktions-Messung:** Logging ob Cache-Hits auftreten (API-Response enthaelt `cache_creation_input_tokens` und `cache_read_input_tokens`)

### Out of Scope

- Migration von CLI-basiertem Client zu HTTP-basiertem Client (separate Story falls noetig)
- Caching von Tier 2/3 dynamischem Kontext (nur System-Prompt wird gecacht)
- OpenAI Prompt-Caching (anderer Mechanismus, andere Story)
- Prompt-Caching-Monitoring im Frontend

## Akzeptanzkriterien

- [ ] Machbarkeitspruefung dokumentiert: Unterstuetzt `claude --print` bzw. Agent SDK `cache_control` oder aequivalenten Mechanismus?
- [ ] **Wenn umsetzbar:**
  - [ ] System-Prompt wird mit Cache-Control-Header/Flag an den Claude CLI uebergeben
  - [ ] Logging zeigt Cache-Hit/Miss-Information (wenn in CLI-Response verfuegbar)
  - [ ] Unit-Test verifiziert: Request-Builder setzt Cache-Control korrekt
  - [ ] Bestehende LLM-Calls funktionieren weiterhin identisch (keine Regression)
- [ ] **Wenn nicht umsetzbar:**
  - [ ] `protocol.md` dokumentiert: was wurde geprueft, warum geht es nicht, welche Alternative wird empfohlen
  - [ ] Story wird als "nicht umsetzbar" geschlossen (kein Code-Change)

## Technische Details

### Bestehende Klassen (potentielle Aenderungen)

| Klasse | Aenderung |
|--------|-----------|
| `ClaudeProviderClient` (`de.its.odin.brain.llm.ClaudeProviderClient`) | Wenn umsetzbar: System-Prompt mit Cache-Control-Mechanismus uebergeben. CLI-Aufruf (`--system-prompt` Flag) ggf. erweitern |
| `BrainProperties` (`de.its.odin.brain.config.BrainProperties`) | Wenn umsetzbar: neues Flag `odin.brain.llm.claude.prompt-caching-enabled=true` |

### Machbarkeitspruefung (Erster Schritt)

1. Claude CLI Dokumentation pruefen: `claude --help`, `claude --print --help`
2. Pruefen ob `--system-prompt` Inhalt Cache-Control-Marker unterstuetzt
3. Pruefen ob der CLI transparent die Messages-API nutzt und `cache_control` weiterleitet
4. Pruefen ob Agent SDK (`@anthropic/sdk` oder Java-Aequivalent) Prompt-Caching-Optionen hat
5. Alternative: Wenn CLI keinen Cache-Control-Support hat, kann der System-Prompt als erste User-Message mit `cache_control` getaggt werden?

### Anthropic Prompt Caching (HTTP API Referenz)

```json
{
  "model": "claude-sonnet-4-20250514",
  "system": [
    {
      "type": "text",
      "text": "...",
      "cache_control": {"type": "ephemeral"}
    }
  ],
  "messages": [...]
}
```

Cache-Verhalten:
- Cache lebt 5 Minuten
- Bei aktivem Polling (78 Calls/Tag = ~1 Call/5 Min) bleibt Cache dauerhaft aktiv
- Cached Tokens kosten 90% weniger als ungecachte
- Response enthaelt: `cache_creation_input_tokens`, `cache_read_input_tokens`

### Konfiguration

```properties
odin.brain.llm.claude.prompt-caching-enabled=true
```

## Konzept-Referenzen

- `docs/concept/overall-research-hybrid-tradingsystem-20262802/theme-backlog.md` — Thema 9: Anthropic Prompt-Caching fuer Tier-1-System-Prompt, Abschnitte IST-Zustand, SOLL-Zustand
- `docs/backend/architecture/05-llm-integration.md` — LLM-Client-Architektur, ClaudeProviderClient, Prompt-Tiers

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen
- `docs/backend/guardrails/development-process.md` — Phasen, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln, Tech-Stack (Claude Agent SDK)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Machbarkeitspruefung vollstaendig dokumentiert
- [ ] Wenn Code-Aenderung: kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] JavaDoc auf modifizierten Klassen/Methoden
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Wenn Code-Aenderung: Unit-Test fuer Cache-Control-Setting
- [ ] Bestehende ClaudeProviderClient-Tests laufen gruen
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Wenn Code-Aenderung: Integrationstest der LLM-Call mit Prompt-Caching durchfuehrt
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session zur Machbarkeitsbewertung (CLI vs. HTTP API Prompt Caching)
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (falls Code-Aenderung)
- [ ] Dimension 2: Konzepttreue-Review (Anthropic Caching-Semantik korrekt?)
- [ ] Dimension 3: Praxis-Review (Cache-TTL vs. Call-Frequenz, Kosten-Nutzen)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten, insbesondere Machbarkeitsergebnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **MACHBARKEITSPRUEFUNG IST DER ERSTE SCHRITT.** Nicht direkt implementieren. Zuerst pruefen ob der `claude` CLI (der als Subprocess gestartet wird — siehe `ClaudeProviderClient`) ueberhaupt Prompt-Caching unterstuetzt. Der CLI verwendet intern die Messages API, aber leitet moeglicherweise `cache_control` nicht durch.
- **Bedingt freigegeben:** Diese Story ist NUR freigegeben wenn das Ergebnis der Machbarkeitspruefung positiv ist. Wenn Prompt-Caching ueber den CLI nicht moeglich ist, wird die Story als "nicht umsetzbar" geschlossen — kein Workaround implementieren.
- **ClaudeProviderClient Architektur:** Der Client startet `claude --print --output-format json --model ... --system-prompt "..."` als Subprocess. System-Prompt wird als CLI-Argument uebergeben. Pruefen ob es eine Moeglichkeit gibt, den System-Prompt-Block mit Cache-Control zu taggen.
- **Alternative HTTP-Client:** Falls CLI nicht geeignet: ein direkter HTTP-Client gegen die Anthropic Messages API waere die Alternative. Das ist aber eine groessere Aenderung (API-Key-Management, HTTP-Client-Infrastruktur) und sollte eine separate Story sein.
- **Kosten-Nutzen:** Bei 78 Calls/Instrument/Tag und 3 Instrumenten = ~234 Calls/Tag. System-Prompt ~2000 Tokens. Einsparung: ~2000 * 234 * 0.9 * Preis-pro-Token/Tag. Bei Claude Sonnet ($3/MTok input): ~$1.26/Tag Einsparung. Relevant bei Scale, marginal bei aktuellem Volumen.
