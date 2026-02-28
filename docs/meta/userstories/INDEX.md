# ODIN User Story Index

**Stand:** 2026-02-28
**Kanonischer Pfad:** `its-odin-wiki/docs/meta/userstories/`

---

## Statistik

| Kategorie | Anzahl |
|-----------|--------|
| Gesamt | 84 |
| Phase 1 | 79 |
| Phase 2 | 5 |
| Abgeschlossen (mit Protokoll) | 59 |
| Offen (ohne Protokoll) | 25 |

---

## V1 Basis-Architektur (ODIN-001 bis ODIN-045)

| ID | Titel | Modul |
|----|-------|-------|
| ODIN-001 | API Subregime Model | odin-api |
| ODIN-002 | API LLM Tactical Schema | odin-api |
| ODIN-003 | API Gate Cascade Model | odin-api |
| ODIN-004 | API Degradation Mode Enum | odin-api |
| ODIN-005 | API Monitor Event Types | odin-api |
| ODIN-006 | API Cycle Tracking Model | odin-api |
| ODIN-007 | Data 1m Monitor Events | odin-data |
| ODIN-008 | Data Session Boundaries | odin-data |
| ODIN-009 | Data Pattern Feature Recognition | odin-data |
| ODIN-010 | Brain Subregime Resolver | odin-brain |
| ODIN-011 | Brain Gate Cascade | odin-brain |
| ODIN-012 | Brain Setup-A FSM | odin-brain |
| ODIN-013 | Brain Setup-D FSM | odin-brain |
| ODIN-014 | Brain R-Based Profit Protection | odin-brain |
| ODIN-015 | Brain Exhaustion Detection | odin-brain |
| ODIN-016 | Brain Tactical Exit | odin-brain |
| ODIN-017 | Brain LLM Tactical Parameter Controller | odin-brain |
| ODIN-018 | Brain Pre-Market Instrument Profiling | odin-brain |
| ODIN-019 | Brain Dual-Key Arbiter | odin-brain |
| ODIN-020 | Execution Repricing Policy | odin-execution |
| ODIN-021 | Execution Stop Trailing 1m | odin-execution |
| ODIN-022 | Execution Multi-Cycle OMS | odin-execution |
| ODIN-023 | Execution FX Conversion Sizing | odin-execution |
| ODIN-024 | Execution Trail-Only Override | odin-execution |
| ODIN-025 | Core Multi-Cycle Orchestration | odin-core |
| ODIN-026 | Core Degradation Mode FSM | odin-core |
| ODIN-027 | Core EOD Forced Close Timing | odin-core |
| ODIN-028 | Core Flash Crash Kill Switch | odin-core |
| ODIN-029 | Core Daily Performance Recording | odin-core |
| ODIN-030 | Core Session Boundary Lifecycle | odin-core |
| ODIN-031 | Audit Hash Chain Integrity | odin-audit |
| ODIN-032 | Audit Trade Intent HMAC | odin-audit |
| ODIN-033 | Backtest Walk-Forward Validation | odin-backtest |
| ODIN-034 | Backtest Challenger Suite | odin-backtest |
| ODIN-035 | Backtest Governance Variants | odin-backtest |
| ODIN-036 | Backtest Cost Model Enhancement | odin-backtest |
| ODIN-037 | App Structured SSE Events | odin-app |
| ODIN-038 | App Control Endpoints Enhancement | odin-app |
| ODIN-039 | App Daily Report Endpoint | odin-app |
| ODIN-040 | Frontend Live Trading Dashboard | odin-frontend |
| ODIN-041 | Frontend Pipeline Detail Realtime | odin-frontend |
| ODIN-042 | Frontend Kill Switch Integration | odin-frontend |
| ODIN-043 | Frontend Alert Routing | odin-frontend |
| ODIN-044 | Persistence Cycle Table | odin-persistence |
| ODIN-045 | Persistence Audit Hash Chain Schema | odin-persistence |

## Phase 2 (zurueckgestellt)

| ID | Titel |
|----|-------|
| ODIN-046 | EU Market Session Boundaries |
| ODIN-047 | L2 Depth Data Tier |
| ODIN-048 | FusionLab Evaluation |
| ODIN-049 | Universe Screening |
| ODIN-050 | Advanced Crash Recovery |

## Erweiterungen Wave 2 (ODIN-051 bis ODIN-057)

| ID | Titel | Modul |
|----|-------|-------|
| ODIN-051 | Frontend E2E Agent Diagnostics | odin-frontend |
| ODIN-052 | Backtest E2E Pipeline Fix | odin-backtest |
| ODIN-053 | Backtest Event Log UI | odin-frontend |
| ODIN-054 | Chart Trade Annotations | odin-frontend |
| ODIN-055 | LLM Sampling Policy | odin-brain |
| ODIN-056 | Latency-Aware Replay | odin-core |
| ODIN-057 | Agent Diagnostic Layer | odin-core |

## S/R Engine v2 (ODIN-058 bis ODIN-064)

| ID | Titel | Modul |
|----|-------|-------|
| ODIN-058 | Spatial Touch Registry | odin-brain |
| ODIN-059 | Dynamic Touch Quality | odin-brain |
| ODIN-060 | Consolidation Band Detection | odin-brain |
| ODIN-061 | OR Level Relevance Gate | odin-brain |
| ODIN-062 | Output Quality Policy | odin-brain |
| ODIN-063 | Band API + Hybrid Rendering | odin-app, odin-frontend |
| ODIN-064 | Price Action Touch Map | odin-brain |

## Research-Themen (ODIN-065 bis ODIN-083)

Basierend auf der Research-Synthese vom 2026-02-28 (5 Quellen: eigene Agents, ChatGPT Deep Research, Gemini Deep Research).

| ID | Thema # | Titel | Prio | Umfang |
|----|---------|-------|------|--------|
| ODIN-065 | 1 | Intraday Seasonality Normalization | P0 | M |
| ODIN-066 | 2 | Backtesting Robustness (PBO, DSR, Monte Carlo) | P0 | L |
| ODIN-067 | 3 | Execution Quality Feedback (TCA) | P0 | L |
| ODIN-068 | 4 | HMM Regime Detection | P1 | L |
| ODIN-069 | 6 | Order Flow Imbalance (OFI) | P1 | L |
| ODIN-070 | 7 | LLM Confidence Calibration | P1 | M |
| ODIN-071 | 8 | Post-Trade Counterfactual Analysis | P1 | M |
| ODIN-072 | 10 | Chain-of-Thought Forcing | P1 | S |
| ODIN-073 | 11 | Pre-Trade Hard Controls | P1 | M |
| ODIN-074 | 12 | Regime-Dependent Arbiter Weighting | P2 | M |
| ODIN-075 | 13 | Volume Profile (VPOC/VAH/VAL) | P2 | M |
| ODIN-076 | 14 | Correlation Check Multi-Instrument | P2 | S |
| ODIN-077 | 15 | Graded Fallback Hierarchy | P2 | S |
| ODIN-078 | 16 | Explicit Multi-Timeframe Labeling | P2 | S |
| ODIN-079 | 17 | ATR Period Intraday Optimization | P2 | S |
| ODIN-080 | 19 | Execution Policy Layer | P2 | L |
| ODIN-081 | 20 | VPIN | P3 | XL |
| ODIN-082 | 22 | Adaptive Weighting via Bandit | P3 | XL |
| ODIN-083 | 9 | Prompt Caching (Agent SDK) | P1 | S |

### Abhaengigkeiten (Research-Themen)

```
ODIN-071 (Counterfactual) ──> ODIN-074 (Regime-Gewichtung) ──> ODIN-082 (Bandit)
ODIN-067 (TCA) ──> ODIN-080 (Execution Policy)
ODIN-069 (OFI) ──> erfordert IB TWS Top-of-Book-Daten
ODIN-070 (LLM-Calibration) ──> erfordert 100+ Backtests mit EventLog
ODIN-083 (Prompt-Caching) ──> erfordert Agent-SDK-Machbarkeitspruefung
```

## Bugfixes

| ID | Titel |
|----|-------|
| ODIN-FIX-001 | Backend Error Cleanup |
