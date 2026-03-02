# ODIN-099: LLM-Flipping-Hysterese fuer entry_timing_bias und tactic

> **GitHub Issue:** saltenhof/its-odin-backend#100
> **Story-ID:** ODIN-099
> **Status:** Todo
> **Size:** S
> **Module:** odin-brain
> **Epic:** Brain Pipeline
> **Erstellt:** 2026-03-02

---

## Kontext

Mit der Einfuehrung des Adaptive LLM Sampling (ODIN-098) steigt die LLM-Call-Frequenz deutlich — in der Opening-Phase wird bei jedem Decision-Bar gesampled. Das ChatGPT-Review des Konzepts warnt vor taktischem "Flipping": Das LLM koennte bei aufeinanderfolgenden Bars widersprüchliche Einschaetzungen liefern (z.B. `entry_timing_bias` wechselt zwischen ALLOW_NOW und WAIT_PULLBACK). Fuer `trail_mode` existiert bereits eine Hysterese-Logik (Wechsel nur bei 2x konsekutiven gleichen Werten, 04-llm-integration.md §13). Diese Story erweitert das Hysterese-Pattern auf `entry_timing_bias` und `tactic`, um Flipping-Artefakte bei hoher Sampling-Frequenz zu unterdruecken.

## Modul

odin-brain

## Abhaengigkeiten

Keine (technisch unabhaengig, aber motiviert durch ODIN-098)

## Geschaetzter Umfang

S

---

## Scope

### In Scope

- `entry_timing_bias` Hysterese: Wechsel nur bei 2x konsekutiven gleichen Werten (analog `trail_mode`)
- `tactic` Hysterese: Wechsel nur bei 2x konsekutiven gleichen Werten (analog `trail_mode`)
- Ggf. generisches Hysterese-Pattern extrahieren (wenn `trail_mode`, `entry_timing_bias` und `tactic` identische Logik verwenden)

### Out of Scope

- Adaptive LLM Sampling Policies (ODIN-098)
- ATR-normalisierte Event-Schwellen (ODIN-097)
- Regime-Hysterese (bereits vorhanden, nicht Teil dieser Story)
- Andere LLM-Response-Felder ohne Flipping-Risiko

## Akzeptanzkriterien

- [ ] `entry_timing_bias` Wechsel (z.B. ALLOW_NOW → WAIT_PULLBACK) erfordert 2x konsekutive gleiche neue Werte bevor er wirksam wird
- [ ] `tactic` Wechsel (z.B. BREAKOUT_FOLLOW → WAIT_PULLBACK) erfordert 2x konsekutive gleiche neue Werte bevor er wirksam wird
- [ ] Bestehende `trail_mode` Hysterese bleibt unveraendert funktional
- [ ] Wenn das LLM 3x hintereinander denselben neuen Wert liefert, wechselt die Hysterese spaetestens beim 2. Mal
- [ ] Hysterese-Zustand wird pro Trading-Session zurueckgesetzt (kein Carry-Over zwischen Tagen)
- [ ] Logging: Hysterese-Unterdrueckungen werden geloggt (DEBUG-Level) fuer Nachvollziehbarkeit

## Technische Details

### Vorgehen

1. **Bestehende `trail_mode` Hysterese finden und analysieren:** Diese dient als Referenz-Implementierung. Pruefe ob die Logik generalisierbar ist.

2. **Option A — Generisches Hysterese-Pattern:**
   Wenn `trail_mode`, `entry_timing_bias` und `tactic` identische Hysterese-Logik verwenden, extrahiere ein generisches `HysteresisFilter<T>` das fuer beliebige Enum-Werte funktioniert:
   ```java
   public class HysteresisFilter<T> {
       private T currentValue;
       private T candidateValue;
       private int consecutiveCount;
       private static final int REQUIRED_CONSECUTIVE = 2;

       public T apply(T newValue) { ... }
   }
   ```

3. **Option B — Einzelne Erweiterungen:**
   Falls die bestehende `trail_mode` Logik zu spezifisch ist, erweitere punktuell `entry_timing_bias` und `tactic` nach dem gleichen Muster.

**Bevorzugt:** Option A, wenn die bestehende Logik es hergibt. Kein Over-Engineering — nur extrahieren wenn mindestens 3 Nutzer (trail_mode, entry_timing_bias, tactic) dieselbe Logik teilen.

### Bestehende Klassen (Aenderungen)

| Klasse | Modul | Aenderung |
|--------|-------|-----------|
| Klasse mit `trail_mode` Hysterese | odin-brain | Analysieren, ggf. generisches Pattern extrahieren |
| Klasse die `entry_timing_bias` verarbeitet | odin-brain | Hysterese-Logik hinzufuegen (2x konsekutiv) |
| Klasse die `tactic` verarbeitet | odin-brain | Hysterese-Logik hinzufuegen (2x konsekutiv) |

**Hinweis:** Die genauen Klassen muessen bei Implementierungsbeginn identifiziert werden — der Implementierer muss die bestehende `trail_mode` Hysterese im Code finden und als Ausgangspunkt verwenden.

## Konzept-Referenzen

- `docs/concept/adaptive-llm-sampling/concept.md` — Abschnitt 3.5: LLM-Flipping-Risiko bei hoher Frequenz (bestehende Mitigationen + zusaetzlich noetige Hysterese)
- `docs/concept/04-llm-integration.md` — §13: trail_mode Hysterese (Referenz-Implementierung)

## Guardrail-Referenzen

- `CLAUDE.md` — Coding-Regeln R1-R13
- `CLAUDE.md` — "ENUM over String for finite sets"
- `docs/backend/guardrails/module-structure.md` — Package-Konventionen

## Notizen fuer den Implementierer

1. **Erst analysieren, dann implementieren.** Finde die bestehende `trail_mode` Hysterese im Code (vermutlich in einer Klasse die LLM-Responses verarbeitet, z.B. `LlmResponseHandler`, `LlmAnalysisProcessor` oder aehnlich). Verstehe das Pattern bevor du es erweiterst.

2. **Generisches Pattern nur bei klarer Passung.** Wenn alle drei Felder (trail_mode, entry_timing_bias, tactic) exakt dieselbe Logik brauchen (2x konsekutiv, gleicher Datentyp-Handling), dann lohnt sich `HysteresisFilter<T>`. Wenn die bestehende Logik Sonderbehandlungen hat, bleib bei punktueller Erweiterung.

3. **Session-Reset nicht vergessen.** Die Hysterese-Zaehler muessen bei Session-Start (neuer Handelstag) zurueckgesetzt werden. Sonst traegt ein "WAIT_PULLBACK" vom Vortag in den naechsten Tag ueber.

4. **Testbarkeit:** Die Hysterese ist reiner Zustandsautomat — perfekt fuer Unit-Tests. Teste alle Transitionen: gleicher Wert, neuer Wert 1x, neuer Wert 2x, Rueckwechsel, Reset.

## Definition of Done

### Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)

### Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer Hysterese-Logik (generisch oder pro Feld)
- [ ] Test: Wert bleibt gleich → sofort uebernommen
- [ ] Test: Wert wechselt 1x → NICHT uebernommen (Hysterese haelt)
- [ ] Test: Wert wechselt 2x konsekutiv → uebernommen
- [ ] Test: Wert wechselt, dann Rueckwechsel → Zaehler reset
- [ ] Test: Session-Reset → Zustand zurueckgesetzt
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### Tests — Komponentenebene (Integrationstests)
- [ ] Mindestens 1 Integrationstest: Hysterese im Kontext der LLM-Response-Verarbeitung
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Hysterese-Logik, Tests, Edge Cases
- [ ] Ergebnis im `protocol.md` dokumentiert

### Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Zustandsautomat korrekt, kein State-Leak)
- [ ] Dimension 2: Konzepttreue-Review (vs. concept.md §3.5 und 04-llm-integration.md §13)
- [ ] Dimension 3: Praxis-Review (unerwartete LLM-Response-Sequenzen)
- [ ] Findings bewertet und berechtigte Findings behoben

### Protokolldatei
- [ ] `protocol.md` mit allen Pflichtabschnitten

### Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

