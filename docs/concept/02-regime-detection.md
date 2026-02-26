# 02 -- Regime-Erkennung, Subregimes, Regime-Fusion, FusionLab

> **Quellen:** Fachkonzept v1.5 (Kap. 13: Intraday Regime Detection), Master-Konzept v1.1 (KPI-primaere Regime-Bestimmung, Konservativitaets-Ranking), Build-Spec v3.0 (Kap. 9: Unified Hybrid, Reliability-weighted Fusion, Ground-Truth-Labeling, FusionLab), Strategie-Sparring (Cold-Start-Policy, Decision-Level-Updates, hierarchisches Pooling), Stakeholder-Feedback (Regime-Hysterese, Subregimes)

---

## 1. Regime-Modell (5 Klassen)

Das System verwendet ein fuenfstufiges Regime-Modell als gemeinsames Vokabular fuer QuantEngine, LLM und Arbiter. Jedes Regime wird als Label mit Confidence (0.0–1.0) gefuehrt.

| Regime | KPI-Kriterien (Decision-Bar-Frame: 3m oder 5m) | Taktik (Long-Only) |
|--------|----------------------------------|---------------------|
| **TREND_UP** | EMA(9) > EMA(21) AND Preis > VWAP AND ADX > 20 (konfigurierbar) | Trend-Following: Pullbacks zum VWAP kaufen, Trailing-Stop, Gewinne laufen lassen |
| **TREND_DOWN** | EMA(9) < EMA(21) AND Preis < VWAP AND ADX > 20 (konfigurierbar) | Default: Nicht handeln. Aggressive Mode (Config OFF): RSI < 25, Volume > 2x SMA, halbe Position |
| **RANGE_BOUND** | ADX < 20 AND Preis innerhalb VWAP +/- 1.0x ATR (konfigurierbar) | Mean-Reversion: Am unteren Range-Ende kaufen, am oberen verkaufen |
| **HIGH_VOLATILITY** | ATR(aktuell) > 1.5x ATR(20-Bar-SMA) (konfigurierbar) | Spike-Harvesting: Schnelle Trades, enge Stops, halbe Positionsgroesse |
| **UNCERTAIN** | Keines der obigen klar erfuellt | Nicht handeln. Beobachten und warten |

**Konservativitaets-Ranking (Long-Only-Kontext):** Bei KPI/LLM-Widerspruch gilt das konservativere Regime. Rangfolge (konservativstes zuerst):

```
UNCERTAIN > TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > TREND_UP
```

---

## 2. Subregimes (5 x 4 = 20 Zustaende)

Das fuenfstufige Regime-Modell wird um Subregimes erweitert, die eine feinere taktische Steuerung ermoeglichen. Subregimes beeinflussen Entry-Strenge, Exit-Defaults und Trail-Parameter.

| Regime | Subregimes |
|--------|-----------|
| TREND_UP | `EARLY`, `MATURE`, `LATE`, `EXHAUSTION` |
| TREND_DOWN | `RELIEF_RALLY`, `PULLBACK`, `ACCELERATION`, `CAPITULATION` |
| RANGE_BOUND | `MEAN_REVERT`, `RANGE_EXPANSION`, `BREAKOUT_ATTEMPT`, `CHOP` |
| HIGH_VOLATILITY | `NEWS_SPIKE`, `MOMENTUM_EXPANSION`, `AFTERSHOCK_CHOP`, `LIQUID_VOL` |
| UNCERTAIN | `TRANSITION`, `MIXED_SIGNALS`, `LOW_LIQUIDITY`, `DATA_QUALITY` |

**Bestimmung (Stakeholder-Entscheidung):** Die KPI-Engine bestimmt das Subregime primaer. Das LLM liefert einen sekundaeren Hint. Bei Widerspruch zwischen KPI und LLM waehlt der Subregime-Resolver das **konservativere** Subregime.

---

## 3. Regime-Hysterese (Anti-Flipping)

Haeufige Regime-Wechsel ("Flipping") fuehren zu Overtrading und schlechten Entscheidungen. Die Hysterese stellt sicher, dass nur stabile Regime-Wechsel anerkannt werden.

**Regel (Normativ):** Ein Regime-Wechsel wird erst akzeptiert, wenn:

- **Normalfall:** 2 aufeinanderfolgende Decision-Bar-Closes mit konsistentem neuem Regime
- **Ausnahme:** Ein Crash-Event (`CRASH_DOWN`, `SPIKE_UP`) KANN sofort `HIGH_VOLATILITY` erzwingen, ohne die Zwei-Bar-Regel abzuwarten

**Anmerkung:** Die fruehere 10m-Confirmation-Loop ist entfallen (Stakeholder-Entscheidung 2026-02-26). Die Regime-Hysterese wird vollstaendig auf Decision-Bar-Ebene geloest.

**Ziel:** Nicht bei jedem VWAP-Cross oder kurzfristigen Noise flippen.

---

## 4. Drei Ansaetze zur Regime-Bestimmung

Es existieren drei konzeptionell verschiedene Ansaetze zur Regime-Bestimmung. Welcher Ansatz produktiv eingesetzt wird, MUSS empirisch durch die FusionLab-Evaluation (Abschnitt 6) entschieden werden.

### 4.1 Ansatz 1: KPI-primaer (MC-Ansatz)

- QuantEngine bestimmt das Regime anhand deterministischer KPI-Kriterien (Abschnitt 1)
- LLM liefert sekundaeren Regime-Hint
- Bei Widerspruch gilt das konservativere Regime
- Regime-Wechsel erfordert Zwei-Bar-Bestaetigung

**Vorteil:** Vollstaendig deterministisch und reproduzierbar. Kein LLM fuer Regime-Bestimmung erforderlich.

**Nachteil:** Reagiert langsamer auf Regime-Uebergaenge, da KPI-Indikatoren lagging sind.

### 4.2 Ansatz 2: Symmetrischer Hybrid (Unified-Ansatz)

- QuantEngine und LLM liefern beide ein Regime-Label mit Confidence
- Der Arbiter fusioniert deterministisch (Dual-Key)
- Decision-Bar-Hysterese wirkt als zusaetzliches Gate
- Bei Widerspruch: konservativeres Regime

**Vorteil:** LLM kann Regime-Uebergaenge frueher erkennen (z.B. "Plateau ist Distribution" vs. "Plateau ist Pause").

**Nachteil:** Abhaengigkeit vom LLM fuer Regime-Qualitaet. Bei LLM-Ausfall: Fallback auf KPI-only.

### 4.3 Ansatz 3: Reliability-weighted Fusion

- Beide Quellen liefern Regime-Label + Confidence
- Jede Quelle hat einen dynamischen Reliability-Score (`R_quant`, `R_llm`)
- Fusion-Formel: `conf_final = normalize(R_quant * conf_quant + R_llm * conf_llm)`
- Reliability-Scores werden EOD anhand von Ground-Truth aktualisiert (asymmetrisch: Fehler senkt staerker als Treffer hebt)

**Vorteil:** Adaptiv — die bessere Quelle wird automatisch staerker gewichtet.

**Nachteil:** Erfordert ausreichend Ground-Truth-Daten fuer Kalibrierung (mindestens 100 Handelstage).

### 4.4 Safety Override (alle Ansaetze)

Unabhaengig vom gewaehlten Fusionsansatz gelten folgende harte Regeln:

- `TREND_DOWN` bestaetigt (2x Decision-Bars) MUSS neue Entries blockieren
- DQ-Risk MUSS neue Entries blockieren
- Crash-Event KANN sofort `HIGH_VOLATILITY` erzwingen

---

## 5. Ground-Truth-Labeling fuer Regime-Kalibrierung

Um Regime-Diagnosen zu kalibrieren (sowohl fuer LLM-Confidence-Pruefung als auch fuer die Reliability-weighted Fusion), wird ex-post ein Ground-Truth-Label erzeugt. Das Labeling erfolgt automatisiert im EOD-Report, ohne manuelles Eingreifen.

### 5.1 Labeling-Logik (30-Minuten-Lookforward, ATR-basiert)

**Inputs pro Snapshot zum Zeitpunkt t:**

- `C_t` = Close (3m) zum Zeitpunkt t
- `ATR_ref` = ATR(3m, 14) zum Zeitpunkt t (nur Vergangenheit)
- Im Fenster [t, t+30m]: `C_fwd` (Close), `H_fwd_max` (Max High), `L_fwd_min` (Min Low)

**Regime-Labels (5 Klassen):**

| Regime-Label | Ground-Truth-Kriterium |
|--------------|----------------------|
| **TREND_UP** | `C_fwd >= C_t + 0.50 * ATR_ref` UND `drawdown_in_window = (H_fwd_max - C_fwd) <= 0.60 * ATR_ref` (nicht komplett zurueckgegeben) |
| **TREND_DOWN** | `C_fwd <= C_t - 0.50 * ATR_ref` |
| **HIGH_VOLATILITY** | `range_fwd = H_fwd_max - L_fwd_min >= 2.0 * ATR_ref` UND weder TREND_UP noch TREND_DOWN eindeutig (sonst hat Trend Prioritaet) |
| **RANGE_BOUND** | `abs(C_fwd - C_t) <= 0.25 * ATR_ref` UND `range_fwd <= 1.0 * ATR_ref` |
| **UNCERTAIN** | Alles andere (Uebergangszonen / Mischformen) |

**Tie-Break-Reihenfolge (deterministisch):**

```
TREND_UP / TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > UNCERTAIN
```

**Leakage-Vermeidung:** Die 30-Minuten-Fenster verwenden nur zukuenftigen Kontext (ex-post). Kein Vorwissen ueber den gesamten Resttag. Die Kalibrierungskurve wird nach **100 Handelstagen** erstmals berechnet und danach taeglich rollierend aktualisiert.

### 5.2 Ground-Truth Datenbank-Schema

```sql
create table if not exists bt_ground_truth (
  snapshot_hash text primary key,
  symbol text not null,
  decision_time timestamptz not null,
  lookforward_minutes int not null default 30,
  atr_ref numeric(20,8) not null,
  c_t numeric(20,8) not null,
  c_t_fwd numeric(20,8) not null,
  fwd_max_high numeric(20,8) not null,
  fwd_min_low numeric(20,8) not null,
  gt_regime text not null,               -- TREND_UP/TREND_DOWN/RANGE/HIGH_VOL/UNCERTAIN
  gt_conf numeric(6,5) not null,         -- optional, z.B. 1.0 wenn klar
  details jsonb not null default '{}'::jsonb
);

create index if not exists ix_bt_gt_time on bt_ground_truth(symbol, decision_time);
```

---

## 6. FusionLab: Isolierte Evaluation der Regime-Fusion

### 6.1 Motivation

Wenn die Reliability-weighted Fusion nur als Teil einer vollstaendigen Trading-Variante laeuft, koennen Verbesserungen wie "Decision-Level Updates" oder "hierarchisches Pooling" nicht sauber isoliert werden — Execution, Stops und Targets ueberdecken den Fusionseffekt.

FusionLab ist ein eigener Versuchsblock, unabhaengig vom Trading-P&L. Ziel ist die Messung der Regime-/Vote-Fusionsqualitaet isoliert.

### 6.2 FusionLab-Varianten

| Variante | Beschreibung |
|----------|-------------|
| **F0** | MC-Regel (KPI-first, konservativ bei Widerspruch) — sofort einsatzfaehig |
| **F1** | Reliability-weighted (EOD-Update der R-Scores) |
| **F2** | Reliability-weighted (Decision-Level-Update statt 1x/Tag) |
| **F3** | Reliability-weighted + hierarchisches Pooling (globaler Score + Regime-Offsets) |

### 6.3 FusionLab-Evaluation Schema

```sql
create table if not exists bt_regime_eval (
  run_id bigint not null,
  variant_id bigint not null,
  fusion_id text not null,               -- F0/F1/F2/F3
  snapshot_hash text not null references bt_ground_truth(snapshot_hash),
  symbol text not null,
  decision_time timestamptz not null,

  quant_regime text not null,
  llm_regime text,
  fused_regime text not null,
  gt_regime text not null,

  correct boolean not null,
  loss_cost numeric(18,10) not null,     -- Kostenmatrix
  quant_conf numeric(6,5),
  llm_conf numeric(6,5),
  fused_conf numeric(6,5),

  r_quant numeric(6,5),                  -- Reliability zum Zeitpunkt t
  r_llm numeric(6,5),
  meta jsonb not null default '{}'::jsonb,

  primary key (fusion_id, snapshot_hash)
);

create index if not exists ix_bt_re_fusion_time on bt_regime_eval(fusion_id, decision_time);
```

### 6.4 Loss-Cost Matrix (asymmetrische Fehlerkosten)

Nicht alle Regime-Fehler sind gleich teuer. Im Long-Only-Kontext ist das Vorhersagen von TREND_UP bei tatsaechlichem TREND_DOWN besonders gefaehrlich.

**Default-Kostenmatrix (Startwerte):**

| Ground Truth | Vorhersage | Kosten | Begruendung |
|-------------|-----------|--------|-------------|
| TREND_DOWN | TREND_UP | **3.0** | Gefaehrlichster Fehler im Long-Only-Kontext |
| HIGH_VOL | TREND_UP | **2.0** | Hohe Verlustgefahr durch Volatilitaet |
| RANGE | TREND_UP | **1.0** | Opportunity-Kosten durch falsche Positionierung |
| TREND_UP | RANGE | **0.5** | Verpasste Opportunity |
| Korrekt | Korrekt | **0.0** | Kein Fehler |

Diese Cost-Matrix ist der Kern, um FusionLab als Proxy fuer geometrische Robustheit zu nutzen. Die Kostenwerte sind Startwerte und werden per Plateau-Test stabilisiert.

### 6.5 FusionLab-Vorteil: Viele Samples

FusionLab hat einen entscheidenden praktischen Vorteil: Pro Tag gibt es viele Entscheidungspunkte auf Decision-Bar-Ebene (typisch 80-130 pro Instrument bei 3m-Bars, 50-80 bei 5m-Bars). Damit stehen deutlich mehr Datenpunkte zur Verfuegung als bei der P&L-basierten Evaluation (1 Datenpunkt pro Tag). Fusion-Verbesserungen koennen statistisch schneller nachgewiesen werden, bevor sie in P&L "bewiesen" werden muessen.

---

## 7. Strategie-Policy-Map (Regime zu erlaubten Aktionen)

| Regime | Primaerziel | Entries? | Adds? | Scale-Out? | Re-Entry? |
|--------|------------|----------|-------|------------|-----------|
| TREND_UP | Trend reiten | Ja | Ja | Standard/Trail | Ja (nach Korrektur) |
| TREND_DOWN | Vermeiden (Long-only) | Nein (Default) | Nein | Exit/Flat | Nein |
| RANGE_BOUND | Kapital schuetzen | Selten | Nein | Klein/Defensiv | Selten |
| HIGH_VOLATILITY | Ueberleben | Sehr selektiv | Nein | Aggressiv (Risk) | Erst nach Beruhigung |
| UNCERTAIN | Warten | Nein | Nein | Nur Risk | Nein |

**TREND_DOWN Aggressive Mode** (Konfigurationsschalter, Default: OFF):

| Regel | Wert |
|-------|------|
| Max. Positionsgroesse | 25% Tageskapital (halbe Standardgroesse) |
| Mindest-RSI | < 25 (extrem ueberverkauft) |
| Mindest-Volumen | Volume-Spike > 2x Durchschnitt |
| Max. Haltedauer | 15 Decision-Bars (bei 3m = 45 Minuten, bei 5m = 75 Minuten) |
| Stops | Enger als Standard |

---

## 8. Day-Type-Flags (Decision-Bar-Ebene)

Neben den lokalen Regime-Labels fuehrt das System Day-Type-Flags, die aus der Decision-Bar-Historie und Volatilitaetsmassen erzeugt werden:

| Day-Type | Heuristik | Strategische Konsequenz |
|----------|-----------|------------------------|
| `TREND_DAY_UP` | Preis >= 3 aufeinanderfolgende Decision-Bars ueber VWAP, EMA(20) steigt, ADX(14) 5m > Schwelle | Setup A/C bevorzugt |
| `TREND_DAY_DOWN` | Analog invers | Kein Trading (Long-only) |
| `RANGE_DAY` | ADX niedrig, haeufige VWAP-Crossings | Setup B/D bevorzugt, Entry-Gates restriktiv |
| `HIGH_VOL_DAY` | Range/ATR auf 5m-Frame ueber Schwelle | Sizing reduziert, Profit Protection frueher, FAST Scale-Out |

### Setup x Regime Matrix

| Regime/DayType | Setup A | Setup B | Setup C | Setup D |
|---------------|---------|---------|---------|---------|
| TREND_DAY_UP | Bevorzugt | Erlaubt | Bevorzugt | Erlaubt |
| RANGE_DAY | Konservativ | Bevorzugt | Nur stark | Bevorzugt |
| HIGH_VOL_DAY | Klein | Klein | Klein | Klein |
| DOWN confirmed | Blockiert | Nur nach Reclaim | Blockiert | Nur nach Trendwechsel |

---

## 9. Regime-Confidence und Entry-Schwellen

Die Regime-Confidence des LLM bestimmt die Quant-Score-Schwelle fuer Entries:

| Regime-Confidence | Quant-Score-Schwelle | Begruendung |
|-------------------|---------------------|-------------|
| < 0.5 | Kein Entry (Regime = UNCERTAIN) | Zu unsicher fuer jede Handlung |
| 0.5 – 0.7 | >= 0.70 (erhoehte Schwelle) | LLM unsicher, Quant muss stark sein |
| > 0.7 | >= 0.50 (Standard-Schwelle) | LLM sicher, Standardbetrieb |

**Vorbedingung (Mandatory Gate):** Ein frischer LLM-Output mit `regimeConfidence > 0` MUSS vorliegen. Ohne frischen LLM-Output ist `regimeConfidence = 0.0` und Entries sind blockiert. Das LLM **triggert** keinen Entry — es liefert ein notwendiges Input-Feature.

---

## 10. Historische Confidence-Kalibrierung (Schicht 5)

Nach einer Einlaufphase von mindestens **100 Handelstagen** wird die Kalibrierung der LLM-Confidence geprueft:

- Ist `regime_confidence = 0.8` tatsaechlich in 80% der Faelle korrekt?
- **Bootstrapping-Methode:** Aus den gesammelten Daten werden per Resampling Konfidenzintervalle fuer die Kalibrierungskurve berechnet
- Systematische Ueber-/Unterschaetzung fuehrt zur Anpassung der Schwellenwerte in der Rules Engine
- Die Kalibrierungskurve wird nach 100 Handelstagen erstmals berechnet und danach taeglich rollierend aktualisiert

---

## 11. Cold-Start-Policy (Normativ)

Das System durchlaeuft nach Erstinbetriebnahme (oder nach einem Reset der Kalibrierungsdaten) eine **Cold-Start-Phase**, in der die Reliability-Kalibrierung noch nicht ausreichend belastbar ist. Waehrend dieser Phase gelten besondere Regeln.

### 11.1 Grundregel

Solange die Reliability-/Fusion-Kalibrierung nicht ausreichend ist, laeuft das System im **MC-Mode (KPI-primaer)** als Default. Das LLM wird als Tactical Parameter Controller (bounded Enums) verwendet, hat aber keinen Einfluss auf die Regime-Fusion. Regime-Bestimmung ist rein KPI-basiert mit konservativem Fallback bei Widerspruch.

### 11.2 Kalibrierungsschwellen (Uebergang Cold-Start zu Reliability-Fusion)

Der Uebergang vom Cold-Start-Modus (MC-Mode) zur Reliability-weighted Fusion DARF erst erfolgen, wenn **alle** folgenden Kriterien erfuellt sind:

| Kriterium | Schwelle | Begruendung |
|-----------|----------|-------------|
| **Mindest-Handelstage** | >= 100 Handelstage mit vollstaendigen Decision-Logs | 20 Tage (wie in frueheren Versionen) sind statistisch nicht belastbar. 100 Tage liefern ausreichend Samples fuer eine robuste Bootstrapping-Analyse |
| **Mindest-Decision-Samples (global)** | >= 5.000 Decision-Samples pro Modell (Quant und LLM) | Bei typisch 80--100 3m-Decisions pro Tag und Instrument sind 5.000 Samples nach ca. 50--65 Instrumenten-Tagen erreicht. Fuer statistische Signifikanz der Kalibrierungskurve erforderlich |
| **Mindest-Samples pro Regime** | >= 200 Samples pro Regime-Klasse (TREND_UP, TREND_DOWN, RANGE_BOUND, HIGH_VOLATILITY, UNCERTAIN) | Regime-spezifische Offsets (fuer F3: hierarchisches Pooling) benoetigen ausreichende Stichprobengroesse pro Klasse. Bei seltenen Regimes (z.B. HIGH_VOLATILITY) kann Shrinkage zum globalen Score helfen |
| **Konfidenzintervall-Breite** | CI-Breite der Kalibrierungskurve (Bootstrap, 95%) < 0.15 | Wenn das Konfidenzintervall der Kalibrierungskurve zu breit ist, ist die Schaetzung nicht praezise genug fuer verlaessliche Gewichtung |

### 11.3 Uebergangsprozess

1. **Phase 0 (Cold Start):** MC-Mode. KPI-primaer, LLM sekundaer. Kein Reliability-Scoring aktiv. Ground-Truth-Labels werden trotzdem erzeugt (fuer spaetere Kalibrierung).
2. **Phase 1 (Datensammlung):** Weiterhin MC-Mode. Reliability-Scores werden im Hintergrund berechnet, aber NICHT fuer Regime-Fusion verwendet. FusionLab laeuft im Shadow-Mode (nur Logging, keine Auswirkung auf Trading).
3. **Phase 2 (Kalibrierung validiert):** Wenn alle Schwellen aus Abschnitt 11.2 erfuellt sind, wird die Reliability-weighted Fusion per Konfigurationsschalter **manuell** aktiviert (kein automatischer Uebergang). Der Operator prueft die FusionLab-Ergebnisse und entscheidet.

### 11.4 Fallback bei Kalibrierungsverlust

Wenn nach Aktivierung der Reliability-weighted Fusion die Kalibrierungsqualitaet sinkt (z.B. durch Regime-Shift im Markt, der historische Daten entwertet), MUSS das System automatisch auf MC-Mode zurueckfallen:

- **Trigger:** CI-Breite der Kalibrierungskurve ueberschreitet 0.20 fuer > 5 konsekutive Handelstage
- **Aktion:** Automatischer Fallback auf MC-Mode + WARNING Alert
- **Recovery:** Manuelle Reaktivierung nach erneuter Validierung der Kalibrierungsqualitaet
