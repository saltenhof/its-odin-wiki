# Protokoll: ODIN-072 — Chain-of-Thought Forcing

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

## Design-Entscheidungen

### 1. CoT Protocol Placement
The REASONING PROTOCOL section is placed between the SAFETY CONTRACT and RESPONSE SCHEMA sections in the system prompt. This ensures the LLM reads the CoT instructions before encountering the field definitions, maximizing protocol compliance.

### 2. Schema Field Reordering
`reasoning` and `alternativeScenario` are listed before `action` in the schema. This leverages autoregressive LLM token generation: when the model generates the reasoning field first, the subsequent decision fields benefit from the contextual computation of the preceding analysis (confirmed by Kim et al. 2024 and Gemini review).

### 3. Prompt Version 1.1.0 to 1.2.0
Minor version increment (not major) because: the `reasoning` and `alternativeScenario` fields already existed — only their schema position changed and a new rule (CoT protocol) was added. This aligns with the semantic versioning spec in Architecture 05 (Minor = Rule, Major = Schema change).

### 4. 200-char Limit for reasoning Field
The story explicitly excludes changes to `LlmTacticalOutput` (no new fields, no field modifications). The 200-char limit is defined in the record's `@Size(max = 200)` annotation. Both ChatGPT and Gemini flagged this as potentially too tight for a full 6-step CoT analysis. This is documented as an open point — a future story should evaluate increasing this limit. In the meantime, the prompt instructs abbreviated analysis, and the LLM has been working within this constraint.

### 5. Inline Schema Annotation
The reasoning field in the schema includes an inline annotation: `(FILL FIRST — step-by-step analysis per REASONING PROTOCOL)`. This cross-references the protocol section and reinforces the CoT instruction at the point of field definition.

## Offene Punkte

1. **reasoning Field Max Length (200 chars):** Both ChatGPT and Gemini independently identified that 200 characters is tight for a 6-step analysis chain. Recommended increase to 600-1200 chars in a future story. The current implementation works within this constraint by instructing abbreviated analysis, but a dedicated story should evaluate increasing it (requires changes to LlmTacticalOutput record, DB column, and validation — out of scope for ODIN-072).

2. **Structured Reasoning Alternative:** ChatGPT suggested changing the reasoning field to a structured type (e.g. `reasoningSteps: ["CTX:...", "IND:...", ...]`). This would improve step separation but requires schema changes and is out of scope. Should be considered for a future prompt version (2.0.0 — major).

## ChatGPT-Sparring

### Session Summary
ChatGPT was consulted for edge cases, prompt formulation improvements, and the 200-char limit concern.

### Suggested Test Scenarios
| Suggestion | Verdict | Rationale |
|------------|---------|-----------|
| Versioning test (new version present, old not) | Verworfen | Existing `buildSystemPrompt_containsVersionNumber` test already covers this; version is a parameter |
| Reasoning protocol appears exactly once | Umgesetzt | Added `buildSystemPrompt_reasoningProtocolSectionHeaderAppearsExactlyOnce` (scoped to section header) |
| Schema is valid JSON | Verworfen | The schema is a human-readable description, not machine-parseable JSON; validation would require a different format |
| Deterministic ordering test | Umgesetzt | Added `buildSystemPrompt_isDeterministic` — builds prompt twice, asserts equality |
| Schema reasoning max length annotation | Umgesetzt | Added `buildSystemPrompt_schemaReasoningHasMaxLengthAnnotation` |
| Response parser order-insensitivity | Out of scope | No parser changes in ODIN-072 |
| Overlength reasoning behavior | Out of scope | Validation is in SchemaValidator/LlmTacticalOutput, not PromptBuilder |

### Prompt Formulation Feedback
- ChatGPT suggested a compact template format (`CTX:...; IND: RSI=.. ADX=..; ...`) for the reasoning field to fit within 200 chars. Valid suggestion but would require documenting the format convention — deferred to the char-limit story.
- Suggested adding explicit noncompliance fallback instruction. Not adopted — the system already returns default NO_TRADE outputs when validation fails.
- Suggested renaming "reasoning" to "rationale" to bias toward concise output. Not adopted — would be a breaking schema change and the field name is established across the codebase.

### alternativeScenario Placement
ChatGPT raised concern that placing alternativeScenario before action might cause the LLM to "hedge early." Assessment: the story explicitly requires logging fields before decision fields. The LLM is instructed to fill reasoning FIRST (which includes the primary decision), so the alternativeScenario field is a secondary consideration.

## Gemini-Review

### Dimension 1: Code Review
- **String concatenation:** Well-formatted and readable. Newline usage correctly chunks the prompt into semantic blocks.
- **No structural Java syntax errors identified.**
- **200-char limit flagged as critical:** Gemini agrees this is physically insufficient for a full 6-step CoT. Documented in Offene Punkte.

### Dimension 2: Konzepttreue
- **Semantic Versioning:** 1.1.0 to 1.2.0 confirmed as correct (Minor = new rule).
- **Theme Backlog alignment:** Implementation matches SOLL-Zustand — prompt-only change, no architecture modifications.
- **AC string match concern:** Gemini noted the AC says "Fill the reasoning field FIRST" but the implementation uses "Before filling ANY decision fields, you MUST complete the reasoning field" plus inline "FILL FIRST". Assessment: both phrasings convey the same requirement. The unit tests verify both key phrases are present. The AC is met in substance.

### Dimension 3: Praxis-Review
- **Token budget impact:** System prompt grows by ~30-40 input tokens — negligible cost impact. Output tokens may increase slightly due to longer reasoning strings — also manageable within existing `max_tokens=2000` configuration.
- **Autoregressive generation benefit:** Gemini confirmed that placing reasoning before action empirically improves output quality due to autoregressive token generation. This is the most impactful change in the PR.
- **Prompt caching interaction:** Version change correctly invalidates the prompt cache. First batch of API calls after deployment will have slightly higher latency until cached — acceptable.
- **No critical findings requiring immediate action beyond the 200-char limit concern.**
