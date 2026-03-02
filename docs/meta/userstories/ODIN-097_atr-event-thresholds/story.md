# ODIN-097: ATR-normalisierte Event-Schwellen fuer LLM-Sampling

> **GitHub Issue:** saltenhof/its-odin-backend#98
> **Story-ID:** ODIN-097
> **Status:** Todo
> **Size:** S
> **Module:** odin-brain
> **Epic:** Brain Pipeline
> **Erstellt:** 2026-03-02

---

## Kontext

Die `EventDrivenSamplingPolicy` nutzt aktuell eine fixe Prozent-Schwelle (`eventThresholdPct=1.5%`) um signifikante Kursbewegungen zu erkennen und darauf einen LLM-Call auszuloesen. Das bestehende Konzept (04-llm-integration.md §10.2) stellt klar: fixe Prozent-Schwellen sind in hochvolatilen Regimen systematisch falsch — ein 1.5%-Move bei einem Small-Cap mit ATR=2.00 ist Normalrauschen, waehrend derselbe Move bei einem Large-Cap mit ATR=0.30 hochsignifikant ist. Die Schwellen muessen in ATR-Einheiten formuliert werden, damit sie automatisch mit der Volatilitaet des Instruments skalieren. Diese Umstellung ist Voraussetzung fuer die phasenspezifischen Event-Schwellen in ODIN-098 (Adaptive LLM Sampling).

## Modul

odin-brain

## Abhaengigkeiten

Keine

## Geschaetzter Umfang

S

---

## Scope

### In Scope

- `EventDrivenSamplingPolicy` von fixer Prozent-Schwelle (`thresholdPct`) auf ATR-normalisierte Schwelle (`thresholdAtrFactor`) umstellen
- Schwelle wird berechnet als: `threshold = thresholdAtrFactor × ATR(14)` aus dem MarketSnapshot
- Konfiguration anpassen: `odin.brain.llm.sampling.event-threshold-pct` ersetzen durch `odin.brain.llm.sampling.event-threshold-atr-factor`
- `BrainProperties.LlmSamplingProperties` entsprechend anpassen
- Default-Wert: `event-threshold-atr-factor=0.6` (entspricht bei ATR=0.40 ca. $0.24 — vergleichbar mit dem bisherigen 1.5%-Verhalten bei einem $16-Instrument)

### Out of Scope

- Phasenspezifische Schwellen (verschiedene ATR-Faktoren pro Marktphase) — das ist ODIN-098
- Neue Sampling-Policies (NeedDriven, PhaseAware, Adaptive) — das ist ODIN-098
- LLM-Flipping-Hysterese — das ist ODIN-099

## Akzeptanzkriterien

- [ ] `EventDrivenSamplingPolicy` akzeptiert `thresholdAtrFactor` (double) statt `thresholdPct` (double)
- [ ] Schwelle wird dynamisch als `thresholdAtrFactor × snapshot.atr14()` berechnet (oder aequivalentes ATR-Feld im Snapshot)
- [ ] Bei fehlendem/NaN ATR-Wert wird ein sicherer Fallback verwendet (z.B. Policy gibt `false` zurueck — kein Trigger bei unbekannter Volatilitaet)
- [ ] `BrainProperties.LlmSamplingProperties` hat Feld `eventThresholdAtrFactor` (statt `eventThresholdPct`)
- [ ] Property `odin.brain.llm.sampling.event-threshold-atr-factor=0.6` als Default in Properties-Datei
- [ ] Bestehende `CompositeSamplingPolicy`-Integration funktioniert unveraendert (Policies sind austauschbar)
- [ ] Alle bestehenden EventDriven-Tests wurden auf ATR-basierte Schwellen migriert und bestehen

## Technische Details

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Package | Aenderung |
|--------|-------|---------|-----------|
| `EventDrivenSamplingPolicy` | odin-brain | `de.its.odin.brain.llm.sampling` | Constructor: `thresholdPct` → `thresholdAtrFactor`. `shouldSample()`: Schwelle dynamisch aus `thresholdAtrFactor × ATR(14)` berechnen statt fixen Prozentwert. Fallback bei NaN/fehlendem ATR |
| `BrainProperties` | odin-brain | `de.its.odin.brain.config` | `LlmSamplingProperties`: `eventThresholdPct` → `eventThresholdAtrFactor` (double, @Min(0.0), default 0.6) |
| `PipelineFactory` | odin-core | `de.its.odin.core.pipeline` | EventDrivenSamplingPolicy-Instanziierung anpassen (neuer Parametername) |

### Konfiguration

```properties
# Vorher:
# odin.brain.llm.sampling.event-threshold-pct=1.5

# Neu — ATR-relative Schwelle:
# Trigger bei Intra-Bar-Move > factor × ATR(14)
# 0.6 × ATR(14)=0.40 → trigger at $0.24 move
odin.brain.llm.sampling.event-threshold-atr-factor=0.6
```

## Konzept-Referenzen

- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 2.3: Event-Schwellen ATR-normalisiert (Tabelle mit phasenspezifischen ATR-Faktoren, Begruendung)
- `docs/concept/04-llm-integration.md` — §10.2: ATR-basierte Schwellen (Guardrail gegen fixe Prozentwerte)

## Guardrail-Referenzen

- `CLAUDE.md` — Coding-Regeln R1-R13, CSpec
- `CLAUDE.md` — "Program against interfaces in de.its.odin.api.port"
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

## Notizen fuer den Implementierer

1. **ATR-Quelle pruefen:** Stelle sicher, dass der `MarketSnapshot` ein ATR(14)-Feld exponiert (z.B. `snapshot.atr14()` oder `snapshot.kpis().atr14()`). Falls nicht vorhanden, muss der Snapshot erweitert werden.

2. **NaN-Handling:** Beim allerersten Bar (wenn ATR noch nicht berechnet werden kann) koennte ATR NaN sein. In diesem Fall sollte die Policy `false` zurueckgeben (kein Event-Trigger bei unbekannter Volatilitaet). Keine Exception werfen.

3. **Rueckwaertskompatibilitaet:** Die `CompositeSamplingPolicy` ruft `EventDrivenSamplingPolicy.shouldSample()` auf — das Interface bleibt gleich. Nur der interne Berechnungsweg aendert sich.

4. **Default-Wert Kalibrierung:** `0.6 × ATR(14)` bei typischem ATR=0.40 ergibt $0.24. Bisheriges Verhalten mit 1.5% bei $16-Aktie = $0.24. Der Default ist also rueckwaertskompatibel fuer typische ODIN-Instrumente.

## Definition of Done

### Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] Namespace-Konvention: `odin.brain.llm.sampling.*`

### Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `EventDrivenSamplingPolicy` mit ATR-basierten Schwellen
- [ ] Test: Move > thresholdAtrFactor × ATR → triggers
- [ ] Test: Move < thresholdAtrFactor × ATR → does not trigger
- [ ] Test: ATR = NaN → safe fallback (no trigger)
- [ ] Test: Verschiedene ATR-Werte (low-vol vs high-vol Instrument)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### Tests — Komponentenebene (Integrationstests)
- [ ] Mindestens 1 Integrationstest: CompositeSamplingPolicy mit ATR-basierter EventDriven-Policy
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klasse, Tests, Akzeptanzkriterien
- [ ] Edge Cases und Grenzfaelle abgefragt
- [ ] Ergebnis im `protocol.md` dokumentiert

### Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Null-Safety, NaN-Handling)
- [ ] Dimension 2: Konzepttreue-Review (ATR-Normalisierung vs. Konzept §2.3 und §10.2)
- [ ] Dimension 3: Praxis-Review (unbehandelte Szenarien)
- [ ] Findings bewertet und berechtigte Findings behoben

### Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

