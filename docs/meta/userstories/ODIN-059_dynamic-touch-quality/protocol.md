# Protokoll: ODIN-059 — Dynamic Touch Quality Scoring

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (42 Tests in TouchQualityTest)
- [x] ChatGPT-Sparring (tatsaechliche Session durchgefuehrt)
- [x] Gemini-Review Dimension 1 (Bugs / null-safety)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis-Szenarien)
- [x] Review-Findings eingearbeitet
- [ ] Commit & Push (QS-Agent)

---

## Design-Entscheidungen

### D1: BounceDetector-Kompatibilitaet — candle-based Methoden beibehalten
Die urspruenglichen Methoden `computeForSupport()` und `computeForResistance()` (Wick/CLV/Separation/Penetration)
werden parallel zu `computeDynamic()` beibehalten. `BounceDetector` nutzt den candle-based Pfad fuer
Bounce-Signal-Qualitaet beim Live-Erkennen. Die Constants wurden mit `CANDLE_`-Prefix versehen zur Abgrenzung.

### D2: satExp-Signatur — divisor-form vs. rate-form
Konzeptdokument sagt `satExp(x, k) = 1 - exp(-k * x)` (k als Rate).
`SrMath.satExp(x, k)` verwendet `1 - exp(-x / k)` (k als Saettigungskonstante).
Numerisch verifiziert: `SrMath.satExp(1.0, 2.0) = 1 - exp(-0.5) ≈ 0.394` entspricht exakt dem
Konzept-Statement "doppeltes Volumen → ~0.39". Die bestehende `satExp`-Funktion wird verwendet (k=2.0).
`satExpRate()` wurde hinzugefuegt aber ist in ODIN-059 nicht genutzt — nur fuer kuenftige Verwendung.

### D3: rthStartBarIndex-Indexkonvention
`TouchEvent.barIndex()` ist RTH-relativ (0-based vom ersten RTH-Bar).
`currentBarIndex` und `oneMinBars`-Liste sind absolut (Sessions-Start-basiert).
`absTouchIndex = touch.barIndex() + rthStartBarIndex` brueckt die beiden Raeume.
`rthStartBarIndex < 0` bedeutet RTH noch nicht gestartet → nur T=0-Scoring, kein Follow-Through.

### D4: null-Bars-Fallback
Wenn `oneMinBars == null` oder leer: `maturityBlend = 0.0` → pure T=0-Score statt Fallback-Konstante 0.5.
T=0 ist selbst-enthalten im `TouchEvent` (priceVelocity, volumeRatio sind beim Schreiben berechnet).
Keine Bar-Daten fuer T=0 notwendig — nur fuer Follow-Through.

### D5: Direktionalitaet von Follow-Through (ODIN-060)
`computeFollowThrough` nutzt `abs(close - levelCenter)` — richtungsagnostisch.
Ein sauberer Ausbruch durch ein Level wuerde Score=1.0 erzeugen (genauso wie ein Bounce).
Das ist ein bekannter False-Positive-Fall. Loesung via richtungsbasiertem Follow-Through
(SUPPORT: close > level gut, RESISTANCE: close < level gut) ist als ODIN-060 vorgemerkt.
Begruendung: Requires SrSide-Parameter und Interface-Aenderung in SrLevel/SrZone —
besser als separatem Story isoliert behandeln.

---

## ChatGPT-Sparring (tatsaechliche Session — 2026-02-27)

### Runde 1: Code-Review

**P0 (kritisch) — gefunden und behoben:**

1. **Age-Indexierung in falschen Indexraum (P0#1)**
   Urspruengliche Implementierung: `age = currentBarIndex - touch.barIndex()`.
   `touch.barIndex()` ist RTH-relativ, `currentBarIndex` absolut. Mischung der Indexraeume.
   Folge: Bei `rthStartBarIndex=10` und `touch.barIndex()=2` ergab sich age=12 (falsch) statt age=2.
   Fix: `absTouchIndex = touch.barIndex() + rthStartBarIndex`, dann `age = currentBarIndex - absTouchIndex`.

2. **volumeSignal kann negativ werden (P0#2)**
   `SrMath.satExp(volumeRatio - 1.0, k)` fuer `volumeRatio < 1.0` liefert negative Werte.
   Folge: Unterdurchschnittliches Volumen "bestrafte" den Score. Konzept sieht 0.0 als Floor vor.
   Fix: `double x = Math.max(0.0, volumeRatio - 1.0)` vor satExp.

3. **satExpRate JavaDoc falsch (P0#3)**
   JavaDoc behauptete "doppeltes Volumen → ~0.39" — das ist korrekt nur fuer `satExp`, nicht `satExpRate`.
   `satExpRate(1.0, 2.0) = 1 - exp(-2) ≈ 0.865` — komplett anderer Wert.
   Fix: JavaDoc korrigiert, Unterschied explizit dokumentiert.

4. **Fehlende null-Pruefung fuer touch (P0#4)**
   `computeDynamic(null, ...)` wuerde bei `touch.barIndex()` mit NPE scheitern.
   Fix: `if (touch == null) return ATR_UNAVAILABLE_FALLBACK;` am Methodenanfang.

**P1 (Design) — bewertet:**

5. **Direktionales Follow-Through (P1#5)**
   ChatGPT empfahl `SrSide`-Parameter fuer richtungsbasiertes Follow-Through.
   Bewertung: Valider Punkt, aber zu grosse Signaturveraenderung fuer ODIN-059.
   Entscheidung: Als ODIN-060 vorgemerkt.

6. **null-Bars bei age=0 (P1#7)**
   ChatGPT empfahl T=0-Score zurueckgeben statt Fallback 0.5 bei null-Bars.
   Bewertung: Korrekt — T=0 ist vollstaendig im TouchEvent. Umgesetzt (D4).

### Runde 2: Follow-up zu Design-Entscheidungen
ChatGPT bestaetigt: (a) Direktionale FT via `SrSide` ist der richtige Weg, minimaler Aufwand,
aber gehoert in separates Story. (b) null-Bars → T=0 only ist semantisch sauber und besser
als konstanter 0.5-Fallback.

---

## Gemini-Review (tatsaechliche Session — 2026-02-27)

### Dimension 1: Bugs / null-safety / Index-Safety

**Findings (berechtigte, eingearbeitet):**

1. **Pre-RTH IndexOutOfBoundsException-Risiko**
   Bei `rthStartBarIndex = -1` wuerde der Fallback `absTouchIndex = touch.barIndex()` (negativ bei Pre-RTH-Touch)
   dazu fuehren, dass `fromIdx = absTouchIndex + 1` negativ sein kann. `subList(neg, toIdx)` wirft Exception.
   Fix: `rthStartBarIndex < 0` → fruehzeitige Rueckgabe mit pure T=0-Score. Sicherer semantisch, weil
   RTH-Kontext fehlt und Follow-Through nicht berechnet werden kann.

2. **NaN-Propagation durch clamp01**
   `SrMath.clamp01(Double.NaN)` lieferte `NaN` (beide Vergleiche false).
   Wenn `priceVelocity` oder `volumeRatio` NaN sind (moglich bei Zero-Bar-Berechnung upstream),
   wird der gesamte Quality-Score zu NaN.
   Fix 1: `clamp01()` behandelt NaN als 0.0 (defensive Schicht).
   Fix 2: `computeVolumeSignal()` und `computeVelocitySignal()` haben explizite NaN-Guards → return 0.0.

3. **subList fromIdx Guard**
   Zusaetzlicher Guard `if (fromIdx >= 0 && fromIdx < toIdx)` vor subList-Aufruf, defensiv gegen
   potenzielle Randwertfehler.

**Korrekt erkannte Staerken:**
- age-Clamp auf 0 behandelt korrekt den "Zukunfts-Touch"-Fall
- `computeFollowThrough()` hat korrekte null/empty-Guards
- Candle-Range-Guard in `computeForSupport` (barRange <= 0) verhindert Division-by-Zero

### Dimension 2: Konzepttreue

**Vollstaendige Fidelitaet bestaetigt:**
- T=0-Score-Formel (0.50 / 0.30 / 0.20): korrekt
- Proximity-Formel mit WINDOW=2.0: korrekt
- VolumeSignal mit satExp(k=2.0) divisor-form: korrekt (inklusive max(0,...)-Clamp)
- VelocitySignal mit WINDOW=2.0: korrekt
- maturityBlend = min(1.0, age/3): korrekt
- Quality-Blend-Formel (0.60/0.40 bei voller Reife): korrekt
- Follow-Through mit WINDOW=3.0: korrekt

### Dimension 3: Praxis-Szenarien

**Scenario A (erster RTH-Bar, barIndex=0):** Korrekt — age=0, pure T=0. Bestaetigt.

**Scenario B (Pre-Market seeded Touch):** Identifizierter semantischer Bug.
Wenn Pivots aus Pre-Market als absolute Indizes gespeichert wuerden und dann RTH-Offset
addiert wird, entsteht ein falscher absTouchIndex.
Klarstellung: In ODIN werden Pivots nur waehrend RTH geseedet (SrLevelEngine laeuft nur RTH).
Pre-Market-Pivots sind explizit out-of-scope. Deshalb kein Code-Bug, aber Dokumentationserfordernis.
Mitigation: rthStartBarIndex < 0 Guard verhindert den Fehlerfall.

**Scenario C (Multi-Touch gleicher Bar):** Korrekt — beide Touches unabhaengig, subList ist View (read-only). Bestaetigt.

**Scenario D (quality bei exact touch bar):** Korrekt — age=0, maturityBlend=0, pure T=0. Bestaetigt.

**Scenario E (1000 Bars, alter Touch):** Korrekt — lookEnd begrenzt auf HORIZON=3 Bars nach Touch,
subList(11, 14) korrekt. Bestaetigt.

---

## Offene Punkte / Follow-ups

### ODIN-060: Direktionales Follow-Through
`computeFollowThrough` ist richtungsagnostisch. Ein Ausbruch durch ein Level wird genauso
bewertet wie ein Bounce. Loesung: `SrSide`-Parameter (SUPPORT/RESISTANCE/UNKNOWN) in
`computeFollowThrough`, damit nur richtungskonformes Follow-Through bewertet wird.
Blocked by: Braucht Signaturerweiterung in SrLevel.averageTouchQuality(), SrZone, SupportLevelConfidence.
Prioritaet: Medium (tritt nur auf bei echten Level-Ausbruechen, die ohnehin seltener sind).
