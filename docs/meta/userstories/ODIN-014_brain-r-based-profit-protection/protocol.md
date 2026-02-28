# Protokoll: ODIN-014 — R-basierte Profit Protection

## Working State

- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (28 Tests → nach QS-Fixes: 31 Tests)
- [x] ChatGPT-Sparring fuer Test-Edge-Cases
- [x] Integrationstests geschrieben (7 Tests → nach QS-Fixes: 8 Tests)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Commit & Push

**Implementierungs-Agent:** Sub-Agent (Implementierung), QS-Agent (diese Runde)

---

## Design-Entscheidungen

### 1. Stateless Engine mit externem Highwater-Tracking
Der `ProfitProtectionEngine` ist stateless. Der Caller (ExitRules / Pipeline-State) ist
verantwortlich fuer die Persistenz des `rFloorHighwater`-Wertes zwischen Aufrufen.
Vorteil: Kein Singleton-State, leicht testbar, keine Synchronisierungsprobleme.

### 2. 5-stufige Floor-Tabelle (vereinfacht gegenueber Konzept)
Das Konzept (05-stops-profit-protection.md) beschreibt eine komplexere Tabelle mit
Trail-ATR-Faktoren und MFE-Lock-Prozenten. Die Implementation folgt der Story-Spezifikation
(story.md), die eine vereinfachte "Floor-only"-Tabelle mit 5 Niveaus definiert. Diese
Vereinfachung ist bewusst (Story Out-of-Scope: MFE-Lock-Prozent und dynamische Trail-Faktoren).

### 3. "Break-Even + Puffer" vs. "Entry-Preis"
Story AC sagt "Break-Even (Entry-Preis)", die Implementierung nutzt `entry + 0.10R`.
Das ist ein 10%-Puffer fuer Slippage/Fees. Dieses Design-Muster ist explizit in
Konzept §4 Zeile ">= 1.0R: stop >= entry + 0.10R" dokumentiert.

### 4. Profile-Semantik: RELAXED statt OFF
Das Konzept (§4.3) beschreibt Profile als OFF/STANDARD/AGGRESSIVE. Die Implementierung
nutzt RELAXED/MODERATE/AGGRESSIVE — kein "OFF" Profil, da der Deaktivierungsfall
ueber `ProfitProtectionProperties.enabled` gehandhabt wird. Begruendung: Ein separates
`OFF`-Profil wuerde die Semantik der Highwater-Erhaltung komplizieren.

### 5. Highwater-Erhaltung bei degenerierten Inputs (QS-Fix)
Urspruenglich: Bei `initialStop >= entryPrice` wurde `R_FLOOR_INACTIVE` zurueckgegeben,
was eine bereits aktive Highwater droppen wuerde. Fix (QS R1): Die bestehende Highwater
wird bei degenerierten Inputs erhalten. Dies ist das fail-safe Verhalten ("floor kann
nur steigen, nie fallen").

### 6. `enabled`-Flag Enforcement (QS-Fix)
Das `ProfitProtectionProperties.enabled`-Flag wurde in den Properties definiert und
in der Config dokumentiert, aber im Engine nie geprueft. Fix (QS R1): Engine prueft
`config.enabled()` als erstes. Bei `enabled=false` wird die bestehende Highwater
erhalten (kein Floor berechnet, aber bestehender Schutz nicht gelo:scht).

### 7. intradayHigh NaN Gap in ExitRules (QS-Fix, BLOCKING)
Urspruenglich: Wenn ATR verfuegbar war aber `intradayHigh = NaN`, wurde weder der
Trail noch der R-Floor geprueft. Dies ist ein "silent missed exit" — das R-Floor-Exit
wird uebersprungen. Fix (QS R1): `trailComputationPossible` wird auf beide Bedingungen
gesetzt (`isFinite(trailingAtr) && isFinite(intradayHigh)`). Der R-Floor-only-Check
im `else if` Branch laeuft dann korrekt als Fallback.

---

## Offene Punkte

### P1 (Information, nicht kritisch): Floating-Point Epsilon
Gemini und ChatGPT haben auf moegliche Floating-Point-Ungenauigkeiten bei R-Schwellen-
Vergleichen hingewiesen (>= Operatoren mit double). In der Praxis ist das Risiko minimal
da `1R = entryPrice - initialStop` ueblicherweise ganzzahlig oder in klaren Dezimalen
ausgedrueckt wird. Fuer kuenftige Robustheit sollte ein Epsilon-Ansatz erwogen werden.
Aktuell: MINOR, kein Handlungsbedarf fuer ODIN-014.

### P2 (Information): Profil-Wechsel waehrend laufender Position (Dead Zone)
Wenn das LLM das Profil von AGGRESSIVE auf MODERATE aendert, entsteht eine "Dead Zone":
Der bestehende Highwater wird erhalten, aber die naechste Aktivierung findet erst bei
MODERATE-Schwellen statt. Das ist korrekte Semantik (Highwater kann nur steigen),
aber ein undokumentiertes Verhalten. Wird in ODIN-014 als "erwartet" akzeptiert.

### P3 (Eskalation): MFE-Lock-Prozent und dynamische Trail-Faktoren
Das Konzept beschreibt MFE-Lock-Prozente (35%, 60%, 75%) und dynamische Trail-ATR-Faktoren
(2.75, 2.00, 1.25, 1.00) pro R-Stufe. Diese Logik ist Out-of-Scope fuer ODIN-014 und
implementiert die vereinfachte Floor-only-Variante. Stakeholder-Entscheidung erforderlich
ob MFE-Lock in einer separaten Story nachgeruested werden soll.

---

## ChatGPT-Sparring

### Session: QS-Agent Round 1, 2026-02-22

**Kontext:** ProfitProtectionEngine, ExitRules, Tests, BrainProperties, story.md

**ChatGPT-Findings (Round 1):**
1. CRITICAL: intradayHigh NaN gap — ATR valid + intradayHigh NaN = R-floor exit wird uebersprungen
2. IMPORTANT: `enabled` Flag ignoriert — Config-Vertrag gebrochen
3. IMPORTANT: Degenerate 1R nach Aktivierung — Highwater wird gelo:scht statt erhalten
4. HINT: `trailingStopHighwaterMark` Naming ist etwas irrefuehrend (jetzt effektiver Stop)

**Round 2 (Prioritaet-Assessment):**
- #1 intradayHigh NaN Gap: BLOCKING (must-fix) — silent missed exit ist High-Severity
- #2 enabled flag: IMPORTANT (likely blocking fuer QS-Vertrag) — fix
- #3 degenerate 1R: IMPORTANT hardening — fix (cheap fix)

**Umgesetzt:**
- intradayHigh NaN Gap: UMGESETZT (ExitRules, beide checkExit-Methoden)
- enabled Flag: UMGESETZT (ProfitProtectionEngine, fruehes Return mit Highwater-Erhaltung)
- Degenerate 1R Highwater-Erhaltung: UMGESETZT (ProfitProtectionEngine)
- Naming: NICHT umgesetzt — Umbenennung wuerde alle bestehenden Caller brechen. JavaDoc klargestellt.

---

## Gemini-Review

### Session: QS-Agent Round 1, 2026-02-22

**Dimension 1 — Code-Review:**
- Floating-Point Precision: `>=` Vergleiche fuer R-Schwellen koennen bei bestimmten Werten
  (z.B. 0.1/0.1 = 0.9999...) ungenau sein. MINOR — bei ganzzahligen 1R-Werten nicht relevant.
- State Intertwining: `trailingStopHighwaterMark = effectiveTrailingStop` fusioniert
  den reinen Trail-Highwater mit dem R-Floor. By Design, kein Bug. Leicht unklares Naming.
- Signal Persistence: `lastComputedRFloorHighwater` wird bei early exits (KILL_SWITCH etc.)
  nicht aktualisiert. MINOR — Trade endet, Stale-State hat keinen Trading-Impact.

**Dimension 2 — Konzepttreue:**
- Break-even Diskrepanz: AC sagt "entry-Preis", Code nutzt entry+0.10R. Bewusste Abweichung
  (Slippage-Puffer), entspricht Konzept §4. Begruendet akzeptiert.
- Schwellen (1.0, 1.5, 2.0, 3.0, 4.0) fuer MODERATE: KORREKT implementiert.
- MFE-Lock Highwater: applyHighwaterMark korrekt — Invariante erhalten.
- Gap-Handling: Top-down Threshold-Check handled Price-Gaps korrekt.

**Dimension 3 — Praxisfragen:**
- Dynamic Profile Switching Dead Zone: dokumentiert in "Offene Punkte" oben.
- Long-Only Assumption: hart verdrahtet. Bei kuenftigem Short-Trading muss die Engine
  grundlegend ueberarbeitet werden (negative oneR-Semantik). Aktuell kein Problem.

**Bewertung:** Alle BLOCKING/IMPORTANT Findings aus Gemini wurden bewertet.
Die BLOCKING-Findings wurden aus ChatGPT-Review eskaliert und gefixt.
Gemini's eigene Findings sind MINOR oder by Design.
