# 08 -- Symmetric Hybrid Protocol, Dual-Key Decisions, Arbiter, Regime-Fusion

> **Quellen:** Fachkonzept v1.5 (Kap. 8: Quantitative Validierungsschicht; Kap. 9: Entscheidungsschleife), Master-Konzept v1.1 (MC-Mode, Tactical Parameter Controller), Build-Spec v3.0 (Kap. 8: Symmetric Hybrid, Dual-Key, Arbiter; Kap. 9: Regime-Fusion; Appendix C: FusionLab), Strategie-Sparring (Reliability-weighted Fusion, Decision-Level-Updates, hierarchisches Pooling, Loss-Cost-Matrix), Stakeholder-Feedback (Konservativitaets-Ranking, Parameter-Fusion)

---

## 1. Designprinzip: Duo auf Augenhoehe

Das System ist ein **symmetrischer Hybrid**:

- **QuantEngine:** Deterministische KPIs, Signale, Gate-Kaskade (Ja/Nein-Schwellen), Vetos, Risk-Fakten
- **LLM:** Lagebeurteilung (Regime/Szenario), Pattern-Hypothesen, taktische Profile/Parameter (bounded), Kontext-Interpretation

**Gleichberechtigung bedeutet:**

- Beide liefern **Vote + Confidence** (Entry/Exit/No-Trade)
- Beide koennen **Soft-Vetos** setzen (z.B. "Unsicherheit", "Exhaustion Risk")
- Der Arbiter kombiniert beides **deterministisch**
- **Ausnahme:** Harte Risk-Exits (Stop, Forced Close, Kill-Switch) sind nicht verhandelbar

---

## 2. Gegenseitige Schutzwirkung (Wichtiges Designziel)

### 2.1 Quant schuetzt vor LLM-Fehlern

- Invalides Schema → LLM-Output wird verworfen
- LLM-Vote ohne Quant-Bestaetigung → kein Entry
- Numerische Grenzen: Keine Stop-/Size-Freiheit fuer das LLM
- Drift-Detection: Wenn LLM systematisch overconfident → automatischer Reset

### 2.2 LLM schuetzt vor Quant-Blindheit

- Erkennt "Plateau ist Distribution" vs. "Plateau ist Pause"
- Erkennt Wechsel in "HIGH_VOL/Chop" frueher als Quant-Filter
- Kann "No-Trade" empfehlen, obwohl Quant formal ok ist (z.B. unsaubere Struktur)
- Erkennt Erschoepfungsmuster, die deterministisch schwer abbildbar sind

---

## 3. Dual-Key Decision (Grundsatz)

Entscheidungen entstehen aus zwei Schluesseln:

| Key | Komponenten |
|-----|------------|
| **Quant-Key** | QuantVote + Confidence + HardVetoFlags + Gate-Kaskade (alle Gates bestanden?) |
| **LLM-Key** | LlmVote + Confidence + SafetyFlags + TacticalControls |

Der **Arbiter** kombiniert beides deterministisch. Kein Key allein kann eine Aktion erzwingen (ausser Hard-Risk).

---

## 4. Vote-Schemata (Bounded Enums)

### 4.1 QuantVote

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `vote` | Enum: `STRONG_ENTRY`, `ALLOW_ENTRY`, `NO_TRADE`, `HOLD`, `EXIT_SOON`, `EXIT_NOW` | Primaere Empfehlung |
| `confidence` | `numeric(6,5)` [0.0 -- 1.0] | Confidence des Quant-Modells |
| `gates_passed` | `boolean` | Alle Gates der Gate-Kaskade bestanden (Ja/Nein-Schwellen pro Indikator, Details in 03-strategy-logic.md) |
| `failed_gates` | Set von Enums | Liste der nicht bestandenen Gates (leer wenn `gates_passed = true`) |
| `hard_veto_flags` | Set von Enums | `WARMUP_INCOMPLETE`, `DQ_VIOLATION`, `OVERBOUGHT`, `SPREAD_TOO_WIDE`, `RR_TOO_LOW` |
| `regime` | Enum: Regime-Vokabular | KPI-basiertes Regime |
| `regime_confidence` | `numeric(6,5)` | Confidence des KPI-Regimes |

### 4.2 LlmVote

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `vote` | Enum: `STRONG_ENTRY`, `ALLOW_ENTRY`, `NO_TRADE`, `HOLD`, `EXIT_SOON`, `EXIT_NOW`, `ALLOW_REENTRY`, `BLOCK_REENTRY`, `SAFETY_VETO` | Primaere Empfehlung |
| `confidence` | `numeric(6,5)` [0.0 -- 1.0] | Confidence des LLM |
| `regime` | Enum: Regime-Vokabular | LLM-diagnostiziertes Regime |
| `regime_confidence` | `numeric(6,5)` | Regime-Confidence |
| `tactical_controls` | Nested Object | `exit_bias`, `trail_mode`, `profit_protection_profile`, `target_policy`, `exhaustion_signal`, `subregime`, `entry_timing_bias`, `scale_out_profile` |
| `pattern_candidates` | Array | `[{pattern: Enum, confidence: float, phase: Enum}]` |
| `safety_flags` | Set von Enums | `CHOP_RISK`, `FAKEOUT_RISK`, `EXHAUSTION_RISK`, `GAP_RISK`, `DATA_QUALITY_RISK` |

---

## 5. Arbiter-Logik (Normativ)

Der Arbiter entscheidet pro Decision Cycle genau **einen** Intent: `NO_ACTION`, `ENTER`, `ADD`, `SCALE_OUT`, `EXIT`.

### 5.1 Allowed Actions per State

| State | ENTER | ADD | SCALE_OUT | EXIT |
|-------|:-----:|:---:|:---------:|:----:|
| WARMUP | -- | -- | -- | -- |
| OBSERVING | Ja | -- | -- | -- |
| SEEKING_ENTRY | Ja | -- | -- | -- |
| PENDING_FILL | -- | -- | -- | Ja (Cancel/Abort) |
| MANAGING_TRADE | -- | Ja | Ja | Ja |
| FLAT_INTRADAY | Ja (Re-Entry) | -- | -- | -- |
| DAY_STOPPED | -- | -- | -- | Ja (Force close only) |
| FORCED_CLOSE | -- | -- | -- | Ja |
| EOD | -- | -- | -- | -- |

### 5.2 Entry-Entscheidung (Dual-Key)

Entry MUSS erfuellt sein:

1. **Keine** Quant Hard-Vetos aktiv
2. LLM nicht `SAFETY_VETO`
3. QuantVote in `{ALLOW_ENTRY, STRONG_ENTRY}`
4. LlmVote in `{ALLOW_ENTRY, STRONG_ENTRY}`
5. Confidences beider Seiten >= `min_conf_entry` (konfigurierbar)
6. Regime-Hysterese nicht gegensaetzlich (z.B. bestaetigt TREND_DOWN → kein Long-Entry)
7. **Gate-Kaskade bestanden** (`gates_passed = true`) — alle Indikator-Gates muessen ihre Ja/Nein-Schwelle erfuellen (Details in 03-strategy-logic.md)

**Dual-Key Matrix:**

| QuantVote | LLM Vote | Ergebnis |
|-----------|----------|----------|
| ENTER + Gates bestanden + keine Vetos | ALLOW_ENTRY / NEUTRAL | ENTER moeglich |
| ENTER + Gates bestanden | CAUTION | ENTER nur wenn Confirmation UP + Urgency HIGH (optional) |
| ENTER | VETO_ENTRY / SAFETY_VETO | **BLOCK** |
| NO_ACTION | ALLOW_ENTRY | **BLOCK** |
| EXIT (z.B. OVERBOUGHT) | ALLOW_ENTRY | **BLOCK** |

### 5.3 Entry mit Regime-Confidence-Schwellen

| Regime-Confidence | Gate-Kaskade-Verhalten |
|-------------------|----------------------|
| < 0.5 | Kein Entry (Regime = UNCERTAIN) — unabhaengig von Gates |
| 0.5 -- 0.7 | Gate-Kaskade MUSS bestanden sein UND Confidence beider Seiten >= erhoehte Schwelle |
| > 0.7 | Gate-Kaskade MUSS bestanden sein (Standard-Schwellen) |

### 5.4 Scale-In / Add

Add MUSS zusaetzlich:

- Struktur-Confirm (Higher-Low) vorhanden
- Kein Exhaustion-Risk aktiv
- `risk_budget_remaining` ausreichend
- Beide Keys stimmen zu

### 5.5 Scale-Out / Tactical Exit

- **Hard-Risk** (Stop, Kill, Forced Close): **sofort**, kein Arbiter-Konsens noetig
- Sonst gilt:
  - Wenn **eine** Seite `EXIT_NOW` → setze `PENDING_EXIT`
  - Exit wird ausgefuehrt, wenn innerhalb von N=1--2 3m-Bars **bestaetigt** ODER 1m-Event `EXHAUSTION_CLIMAX`/`STRUCTURE_BREAK_1M` eintritt
  - Wenn nicht bestaetigt: `EXIT_SOON`-Profile (Trail tighter, Scale-Out aggressiver)

### 5.6 Re-Entry (Multi-Cycle)

Re-Entry nur wenn:

- State = `FLAT_INTRADAY` oder Position stark reduziert
- `cycleCounter < maxCyclesPerInstrumentPerDay`
- Beide Keys stimmen zu (`ALLOW_REENTRY` bzw. `ALLOW_ENTRY`)
- Regime-Hysterese unterstuetzt (oder mindestens nicht widerspricht)
- Cooling-Off abgelaufen (min. 15 Minuten)
- Vorheriger Cycle profitabel (Profit-Gate)
- Verbleibendes Risk-Budget erlaubt minimale Position (Budget-Gate)

---

## 6. Parameter-Fusion (Quant Baseline + LLM Modifier)

Fuer risiko-relevante Parameter gilt **konservativ gewinnt**:

### 6.1 Allgemeine Fusionsregeln

| Parameter | Quant liefert | LLM liefert | Fusion |
|-----------|--------------|-------------|--------|
| Trail-Faktor | `base_trail_factor` | `trail_mode` (WIDE/NORMAL/TIGHT) → Multiplikator | `effective = min(base, base * multiplier)` wenn LLM tighter; clamp auf [min, max] |
| Profit-Protection | Standard-Profil | `profit_protection_profile` (OFF/STANDARD/AGGRESSIVE) | Engerer Wert gewinnt |
| Exit-Bias | Kein direkter Beitrag | `exit_bias` (HOLD/NEUTRAL/EXIT_SOON/EXIT_NOW) | Aktiviert TACTICAL_EXIT-Check bei KPI-Bestaetigung |
| Scale-Out | Standard-Profil | `scale_out_profile` (OFF/CONSERVATIVE/STANDARD/AGGRESSIVE) | Profil-Wahl, deterministische Umsetzung |
| Entry-Timing | Standard-Gate | `entry_timing_bias` (ALLOW_NOW/WAIT_PULLBACK/WAIT_CONFIRMATION/ALLOW_RE_ENTRY/SKIP) | LLM verschaerft oder lockert Gate |

### 6.2 Grundregel

- LLM darf nur **enger** machen als Safety-Floor
- Wenn LLM "weiter" will, darf es nur bis zu einem quant-definierten Maximum
- Konfigurierbar, ob LLM in bestimmten Parametern auch "weiter" als Quant-Baseline gehen darf

---

## 7. Regime-Fusion (Normativ)

### 7.1 Regime-Bestimmung: KPI primaer, LLM sekundaer

Die Regime-Bestimmung folgt dem Prinzip **KPI primaer, LLM sekundaer**:

| Regime | KPI-Kriterien |
|--------|--------------|
| TREND_UP | EMA(9) > EMA(21) AND Preis > VWAP AND ADX > 20 |
| TREND_DOWN | EMA(9) < EMA(21) AND Preis < VWAP AND ADX > 20 |
| RANGE_BOUND | ADX < 20 AND Preis innerhalb VWAP +/- 1.0x ATR |
| HIGH_VOLATILITY | ATR(aktuell) > 1.5 x ATR(20-Bar-SMA) |
| UNCERTAIN | Keines der obigen klar erfuellt |

### 7.2 Regime-Fusion bei Widerspruch

Bei KPI/LLM-Widerspruch gilt das **konservativere** Regime (im Long-Only-Kontext):

```
Konservativitaets-Ranking (von konservativ zu offensiv):
UNCERTAIN > TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > TREND_UP
```

### 7.3 Regime-Hysterese (Anti-Flipping)

- Regime-Wechsel erst nach **2 aufeinanderfolgenden** Decision-Bar-Closes mit konsistentem neuem Regime
- **Ausnahme:** Crash-Event kann sofort `HIGH_VOLATILITY` erzwingen (via 1m-Event-Detektor)
- Ziel: Nicht bei jedem VWAP-Cross flippen

### 7.4 Subregime-Fusion

Das LLM liefert `subregime` als sekundaeren Hint. Der Subregime-Resolver folgt KPI primaer:

- Bei Widerspruch zwischen KPI und LLM: **konservativeres Subregime** wird gewaehlt
- Beispiel: KPI sagt TREND_UP/MATURE, LLM sagt TREND_UP/LATE → LATE gewinnt (konservativer)

---

## 8. Reliability-weighted Fusion (FusionLab)

Fuer die Kalibrierung und Evaluation der Regime-Fusion werden vier Fusionsmechaniken im FusionLab verglichen:

### 8.1 FusionLab-Varianten

| Variante | Beschreibung |
|----------|-------------|
| **F0: MC-Regel** | KPI-first, konservativ bei Widerspruch. Sofort einsatzfaehig |
| **F1: Reliability-weighted (EOD)** | Gewichtung nach Zuverlaessigkeit, EOD-Update der Reliability-Scores |
| **F2: Reliability-weighted (Decision-Level)** | Update der Reliability-Scores bei jeder Decision statt 1x/Tag |
| **F3: Hierarchisches Pooling** | Globaler Reliability-Score + Regime-spezifische Offsets |

### 8.2 Ground-Truth-Labeling fuer Kalibrierung

Ground-Truth wird ex-post berechnet (30-Minuten-Lookforward, ATR-basiert):

| Ground-Truth-Label | Kriterium |
|--------------------|-----------|
| **TREND_UP** | `C_fwd >= C_t + 0.50 x ATR_ref` UND Drawdown im Fenster <= 0.60 x ATR_ref |
| **TREND_DOWN** | `C_fwd <= C_t - 0.50 x ATR_ref` |
| **HIGH_VOLATILITY** | `range_fwd = H_fwd_max - L_fwd_min >= 2.0 x ATR_ref` UND weder TREND_UP noch TREND_DOWN eindeutig |
| **RANGE_BOUND** | `abs(C_fwd - C_t) <= 0.25 x ATR_ref` UND `range_fwd <= 1.0 x ATR_ref` |
| **UNCERTAIN** | Alles andere (Uebergangszonen / Mischformen) |

**Tie-Break-Reihenfolge (deterministisch):** TREND_UP / TREND_DOWN > HIGH_VOLATILITY > RANGE_BOUND > UNCERTAIN

### 8.3 Loss-Cost-Matrix

Fehler werden asymmetrisch gewichtet:

| Ground-Truth | Prediction | Cost |
|-------------|------------|------|
| TREND_DOWN | TREND_UP | **3.0** (gefaehrlich im Long-only) |
| HIGH_VOL | TREND_UP | **2.0** |
| RANGE | TREND_UP | **1.0** |
| TREND_UP | RANGE | **0.5** (Opportunity-Kosten) |
| Korrekt | Korrekt | **0.0** |

### 8.4 FusionLab-Datenbank-Schema

```sql
-- Ground-Truth Labels pro Decision-Snapshot
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
  gt_regime text not null,
  gt_conf numeric(6,5) not null,
  details jsonb not null default '{}'::jsonb
);

-- FusionLab Evaluation pro Fusion-Mechanik
create table if not exists bt_regime_eval (
  run_id bigint not null,
  variant_id bigint not null,
  fusion_id text not null,                 -- F0/F1/F2/F3
  snapshot_hash text not null references bt_ground_truth(snapshot_hash),
  symbol text not null,
  decision_time timestamptz not null,
  quant_regime text not null,
  llm_regime text,
  fused_regime text not null,
  gt_regime text not null,
  correct boolean not null,
  loss_cost numeric(18,10) not null,
  quant_conf numeric(6,5),
  llm_conf numeric(6,5),
  fused_conf numeric(6,5),
  r_quant numeric(6,5),
  r_llm numeric(6,5),
  meta jsonb not null default '{}'::jsonb,
  primary key (fusion_id, snapshot_hash)
);
```

---

## 9. Strategie-Policy Map (Regime → erlaubte Aktionen)

| Regime | Primaerziel | Entries? | Adds? | Scale-Out? | Re-Entry? |
|--------|------------|:--------:|:-----:|:----------:|:---------:|
| OPENING_VOLATILITY | Diagnose | Nein | Nein | Nur Risk | Nein |
| TREND_UP | Trend reiten | Ja | Ja | Standard/Trail | Ja (nach Korrektur) |
| RANGE_BOUND | Kapital schuetzen | Selten | Nein | Klein/defensiv | Selten |
| HIGH_VOLATILITY | Ueberleben | Sehr selektiv | Nein | Aggressiv (Risk) | Erst nach Beruhigung |
| EXHAUSTION_RISK | Gewinne schuetzen | Nein (spaet) | Nein | Aggressiv | Ggf. spaeter |
| RECOVERY | Turn nutzen | Ja (konservativ) | Selektiv | Standard | Ja |
| TREND_DOWN | Long-only: vermeiden | Nein | Nein | Exit/flat | Nein |
| BREAKOUT_ATTEMPT | Breakout validieren | Ja (mit Confirm) | Nach HL | Standard | Ja |
| UNCERTAIN | Warten | Nein | Nein | Nur Risk | Nein |

---

## 10. Conflict Resolution: Zusammenfassung

| Konflikt-Typ | Loesung |
|-------------|---------|
| KPI vs. LLM Regime | Konservativeres Regime gewinnt |
| Quant-Vote vs. LLM-Vote (Entry) | Beide MUESSEN zustimmen (Dual-Key) |
| Trail-Faktor: Profit-Protection vs. LLM | Engerer Wert gewinnt: `min(PP, LLM)` |
| Exit: Hard-Risk vs. Soft-Exit | Hard-Risk immer zuerst, LLM-unabhaengig |
| Subregime: KPI vs. LLM | Konservativeres Subregime gewinnt |
| Entry bei niedrigem Confidence | Erhoehte Confidence-Schwellen, Gate-Kaskade obligatorisch |

---

## 11. JSON-Schema-Referenz (Beispiel-Payloads)

Die folgenden Beispiele zeigen die konzeptionellen JSON-Strukturen fuer die drei zentralen Nachrichten im Symmetric Hybrid Protocol. Die Feldnamen, Typen und Wertebereiche sind normativ; die konkreten Werte sind illustrativ.

### 11.1 QuantOutput (bounded)

```json
{
  "quant_regime": "TREND_UP",
  "quant_regime_confidence": 0.74,
  "quant_vote": "ALLOW_ENTRY",
  "quant_vote_confidence": 0.68,
  "gates_passed": true,
  "failed_gates": [],
  "hard_veto_flags": [],
  "features": {
    "ext_vwap_atr": 0.4,
    "ema9_gt_ema21": true,
    "rsi14": 61.2,
    "adx14": 23.8,
    "vol_ratio": 1.4,
    "atr_decay_ratio": 0.55
  }
}
```

**Felder:**

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `quant_regime` | Enum (Regime-Vokabular) | KPI-basiertes Regime-Label |
| `quant_regime_confidence` | Float [0.0--1.0] | Confidence des KPI-Regimes |
| `quant_vote` | Enum: `STRONG_ENTRY`, `ALLOW_ENTRY`, `NO_TRADE`, `HOLD`, `EXIT_SOON`, `EXIT_NOW` | Primaere Empfehlung |
| `quant_vote_confidence` | Float [0.0--1.0] | Confidence des Quant-Votes |
| `gates_passed` | Boolean | Alle Gates der Gate-Kaskade bestanden — Ja/Nein-Schwellen pro Indikator, alle muessen bestanden sein (Details in 03-strategy-logic.md) |
| `failed_gates` | Set von Enums | Liste der nicht bestandenen Gates (leer wenn `gates_passed = true`). Beispiele: `RSI_OVERBOUGHT`, `SPREAD_TOO_WIDE`, `VOLUME_INSUFFICIENT` |
| `hard_veto_flags` | Set von Enums | Harte Vetos (z.B. `WARMUP_INCOMPLETE`, `DQ_VIOLATION`, `OVERBOUGHT`) |
| `features` | Object | Berechnete KPI-Features fuer Logging und Nachvollziehbarkeit |

### 11.2 LlmOutput (bounded)

```json
{
  "llm_regime": "TREND_UP",
  "llm_regime_confidence": 0.71,
  "llm_vote": "ALLOW_ENTRY",
  "llm_vote_confidence": 0.64,
  "pattern_candidates": [
    {"pattern": "SETUP_A_OPENING_CONSOLIDATION", "confidence": 0.66, "phase": "MATURE"}
  ],
  "tactical_controls": {
    "risk_mode": "NORMAL",
    "trail_mode": "NORMAL",
    "profit_protection_profile": "STANDARD",
    "scale_out_profile": "STANDARD",
    "entry_timing_bias": "WAIT_FOR_PULLBACK",
    "exit_bias": "HOLD"
  },
  "safety_flags": [],
  "logging_only": {
    "key_observations": [],
    "risk_factors": []
  }
}
```

**Felder:**

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `llm_regime` | Enum (Regime-Vokabular) | LLM-diagnostiziertes Regime-Label |
| `llm_regime_confidence` | Float [0.0--1.0] | Confidence des LLM-Regimes |
| `llm_vote` | Enum (siehe Abschnitt 4.2) | Primaere Empfehlung inkl. Re-Entry/Safety |
| `llm_vote_confidence` | Float [0.0--1.0] | Confidence des LLM-Votes |
| `pattern_candidates` | Array (max. 2) | Erkannte Patterns mit Enum + Confidence + Phase |
| `tactical_controls` | Object (bounded Enums) | Taktische Steuerungsparameter (alle als Enums) |
| `safety_flags` | Set von Enums | Safety-Flags (z.B. `CHOP_RISK`, `EXHAUSTION_RISK`) |
| `logging_only` | Object | Nur fuer Logging, NIE in Entscheidungslogik |

### 11.3 TradeIntent

```json
{
  "intent": "ENTRY",
  "symbol": "XYZ",
  "time": "2026-02-21T14:33:00Z",
  "size_profile": "STARTER",
  "tactic": "WAIT_PULLBACK",
  "reason_codes": ["SETUP_A_PULLBACK", "TREND_UP_CONFIRMED"],
  "constraints": {"max_chase_r": 0.2}
}
```

**Felder:**

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| `intent` | Enum: `NO_ACTION`, `ENTER`, `ADD`, `SCALE_OUT`, `EXIT` | Art der Aktion |
| `symbol` | String | Instrument-Identifier |
| `time` | ISO-8601 Timestamp | MarketClock-Zeitstempel der Entscheidung |
| `size_profile` | Enum: `STARTER`, `ADD`, `FULL` | Positionsgroessen-Profil |
| `tactic` | Enum: `WAIT_PULLBACK`, `BREAKOUT_FOLLOW`, `NO_TRADE` | LLM-Taktik — OMS bestimmt den konkreten Entry-Preis deterministisch aus Struktur-Levels (siehe 07-oms-execution.md, Abschnitt 3.2) |
| `reason_codes` | String[] | Maschinenlesbare Begruendungen (ReasonCode-Taxonomie) |
| `constraints` | Object | Optionale Einschraenkungen (z.B. max. Chase-Distanz in R) |

> **Hinweis:** Das LLM liefert keine konkreten Preisfelder (`entry_price_zone`, `target_price`). Der Entry-Preis wird vom OMS deterministisch aus Struktur-Levels bestimmt (Pre-Market High/Low, Prior Day Close/High/Low, erste 1m-Bar, Gap-Fill-Level, runde Zahlen). Die `tactic` des LLM impliziert, welches Struktur-Level relevant ist.
