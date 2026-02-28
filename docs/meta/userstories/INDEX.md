# ODIN User Story Index

**Stand:** 2026-02-28
**Kanonischer Pfad:** `its-odin-wiki/docs/meta/userstories/`

---

## Statistik

| Kategorie | Anzahl |
|-----------|--------|
| Gesamt (inkl. Sub-Stories) | 90 |
| Abgeschlossen | 58 |
| Offen | 27 |
| Phase 2 (zurueckgestellt) | 5 |

---

## V1 Basis-Architektur (ODIN-001 bis ODIN-045) — ALLE ABGESCHLOSSEN

| ID | Titel | Modul | Status |
|----|-------|-------|--------|
| ODIN-001 | API Subregime Model | odin-api | Done |
| ODIN-002 | API LLM Tactical Schema | odin-api | Done |
| ODIN-003 | API Gate Cascade Model | odin-api | Done |
| ODIN-004 | API Degradation Mode Enum | odin-api | Done |
| ODIN-005 | API Monitor Event Types | odin-api | Done |
| ODIN-006 | API Cycle Tracking Model | odin-api | Done |
| ODIN-007 | Data 1m Monitor Events | odin-data | Done |
| ODIN-008 | Data Session Boundaries | odin-data | Done |
| ODIN-009 | Data Pattern Feature Recognition | odin-data | Done |
| ODIN-010 | Brain Subregime Resolver | odin-brain | Done |
| ODIN-011 | Brain Gate Cascade | odin-brain | Done |
| ODIN-012 | Brain Setup-A FSM | odin-brain | Done |
| ODIN-013 | Brain Setup-D FSM | odin-brain | Done |
| ODIN-014 | Brain R-Based Profit Protection | odin-brain | Done |
| ODIN-015 | Brain Exhaustion Detection | odin-brain | Done |
| ODIN-016 | Brain Tactical Exit | odin-brain | Done |
| ODIN-017 | Brain LLM Tactical Parameter Controller | odin-brain | Done |
| ODIN-018 | Brain Pre-Market Instrument Profiling | odin-brain | Done |
| ODIN-019 | Brain Dual-Key Arbiter | odin-brain | Done |
| ODIN-020 | Execution Repricing Policy | odin-execution | Done |
| ODIN-021 | Execution Stop Trailing 1m | odin-execution | Done |
| ODIN-022 | Execution Multi-Cycle OMS | odin-execution | Done |
| ODIN-023 | Execution FX Conversion Sizing | odin-execution | Done |
| ODIN-024 | Execution Trail-Only Override | odin-execution | Done |
| ODIN-025 | Core Multi-Cycle Orchestration | odin-core | Done |
| ODIN-026 | Core Degradation Mode FSM | odin-core | Done |
| ODIN-027 | Core EOD Forced Close Timing | odin-core | Done |
| ODIN-028 | Core Flash Crash Kill Switch | odin-core | Done |
| ODIN-029 | Core Daily Performance Recording | odin-core | Done |
| ODIN-030 | Core Session Boundary Lifecycle | odin-core | Done |
| ODIN-031 | Audit Hash Chain Integrity | odin-audit | Done |
| ODIN-032 | Audit Trade Intent HMAC | odin-audit | Done |
| ODIN-033 | Backtest Walk-Forward Validation | odin-backtest | Done |
| ODIN-034 | Backtest Challenger Suite | odin-backtest | Done |
| ODIN-035 | Backtest Governance Variants | odin-backtest | Done |
| ODIN-036 | Backtest Cost Model Enhancement | odin-backtest | Done |
| ODIN-037 | App Structured SSE Events | odin-app | Done |
| ODIN-038 | App Control Endpoints Enhancement | odin-app | Done |
| ODIN-039 | App Daily Report Endpoint | odin-app | Done |
| ODIN-040 | Frontend Live Trading Dashboard | odin-frontend | Done |
| ODIN-041 | Frontend Pipeline Detail Realtime | odin-frontend | Done |
| ODIN-042 | Frontend Kill Switch Integration | odin-frontend | Done |
| ODIN-043 | Frontend Alert Routing | odin-frontend | Done |
| ODIN-044 | Persistence Cycle Table | odin-persistence | Done |
| ODIN-045 | Persistence Audit Hash Chain Schema | odin-persistence | Done |

---

## Phase 2 (zurueckgestellt, nicht beauftragt)

| ID | Titel | Status |
|----|-------|--------|
| ODIN-046 | EU Market Session Boundaries | Phase 2 |
| ODIN-047 | L2 Depth Data Tier | Phase 2 |
| ODIN-048 | FusionLab Evaluation | Phase 2 |
| ODIN-049 | Universe Screening | Phase 2 |
| ODIN-050 | Advanced Crash Recovery | Phase 2 |

---

## Erweiterungen Wave 2 (ODIN-051 bis ODIN-057)

| ID | Titel | Modul | Status |
|----|-------|-------|--------|
| ODIN-051 | Frontend E2E Agent Diagnostics | odin-frontend | Done |
| ODIN-052 | Backtest E2E Pipeline Fix | odin-backtest | Done |
| ODIN-053 | Backtest Event Log UI | odin-frontend | Done |
| ODIN-054 | Chart Trade Annotations | odin-frontend | Done |
| ODIN-055 | LLM Sampling Policy | odin-brain | Done |
| ODIN-056 | Latency-Aware Replay | odin-core | Done |
| ODIN-057 | Agent Diagnostic Layer | odin-core | Done |

---

## S/R Engine v2 (ODIN-058 bis ODIN-064)

| ID | Titel | Modul | Status |
|----|-------|-------|--------|
| ODIN-058 | Spatial Touch Registry | odin-brain | Done |
| ODIN-059 | Dynamic Touch Quality | odin-brain | Done |
| ODIN-060 | Consolidation Band Detection | odin-brain | Done |
| ODIN-061 | OR Level Relevance Gate | odin-brain | Done |
| ODIN-062 | Output Quality Policy | odin-brain | Done |
| ODIN-063 | Band API + Hybrid Rendering | odin-app, odin-frontend | Done |
| ODIN-064 | Price Action Touch Map | odin-brain | Done |

---

## Bugfixes

| ID | Titel | Status |
|----|-------|--------|
| ODIN-FIX-001 | Backend Error Cleanup | Done |

---

## Research-Themen (ODIN-065 bis ODIN-083) — ALLE OFFEN

Basierend auf der Research-Synthese vom 2026-02-28 (5 Quellen: eigene Agents, ChatGPT Deep Research, Gemini Deep Research).
Themen 5 (News-Integration), 18 (Prompt-Injection) und 21 (Vision-Modul) wurden vom Stakeholder abgelehnt.

### P0 — Fundament

| ID | Titel | Umfang | Abhaengigkeiten | Status |
|----|-------|--------|-----------------|--------|
| ODIN-065 | Intraday Seasonality Normalization | M | Keine | Offen |
| ODIN-066 | Backtesting Robustness (PBO, DSR, Monte Carlo) | L | Keine | Offen |
| ODIN-067 | Execution Quality Feedback (TCA) | L | Keine | Offen |

### P1 — Kernverbesserungen

| ID | Titel | Umfang | Abhaengigkeiten | Status |
|----|-------|--------|-----------------|--------|
| ODIN-068 | HMM Regime Detection | L | Keine | Offen |
| ODIN-069 | Order Flow Imbalance (OFI) | L | IB TWS Top-of-Book pruefen | Offen |
| ODIN-070 | LLM Confidence Calibration | M | 100+ Backtests mit EventLog | Offen |
| ODIN-071 | Post-Trade Counterfactual Analysis | M | Keine | Offen |
| ODIN-072 | Chain-of-Thought Forcing | S | Keine | Offen |
| ODIN-073 | Pre-Trade Hard Controls | M | Keine | Offen |
| ODIN-083 | Prompt Caching (Agent SDK) | S | Agent-SDK-Machbarkeitspruefung | Offen (bedingt) |

### P2 — Verfeinerung

| ID | Titel | Umfang | Abhaengigkeiten | Status |
|----|-------|--------|-----------------|--------|
| ODIN-074 | Regime-Dependent Arbiter Weighting | M | ODIN-071 | Offen |
| ODIN-075 | Volume Profile (VPOC/VAH/VAL) | M | Keine | Offen |
| ODIN-076 | Correlation Check Multi-Instrument | S | Keine | Offen |
| ODIN-077 | Graded Fallback Hierarchy | S | Keine | Offen |
| ODIN-078 | Explicit Multi-Timeframe Labeling | S | Keine | Offen |
| ODIN-079 | ATR Period Intraday Optimization | S | Keine | Offen |
| ODIN-080 | Execution Policy Layer | L | ODIN-067 | Offen |

### P3 — Langfristig (aufgesplittet in Sub-Stories)

| ID | Titel | Umfang | Abhaengigkeiten | Status |
|----|-------|--------|-----------------|--------|
| ODIN-081 | ~~VPIN~~ (aufgesplittet) | ~~XL~~ | — | Aufgesplittet |
| ODIN-081a | VPIN Volume Bucketizer | M | Keine | Offen |
| ODIN-081b | VPIN Calculator | M | ODIN-081a | Offen |
| ODIN-081c | VPIN Gate Integration | S | ODIN-081b | Offen |
| ODIN-082 | ~~Adaptive Weighting Bandit~~ (aufgesplittet) | ~~XL~~ | — | Aufgesplittet |
| ODIN-082a | Bandit Core Algorithm | M | ODIN-071, ODIN-074 | Offen |
| ODIN-082b | Bandit Arbiter Integration | M | ODIN-082a | Offen |
| ODIN-082c | Bandit Persistence + Reward | M | ODIN-082b, ODIN-071 | Offen |

### Abhaengigkeitsgraph

```
ODIN-067 (TCA) ──────────────────────────────> ODIN-080 (Execution Policy)

ODIN-071 (Counterfactual) ──> ODIN-074 (Regime-Gewichtung) ──> ODIN-082a (Bandit Core)
                          └──────────────────────────────────> ODIN-082a (Bandit Core)
                          └──────────────────────────────────> ODIN-082c (Bandit Persistence)

ODIN-081a ──> ODIN-081b ──> ODIN-081c
ODIN-082a ──> ODIN-082b ──> ODIN-082c

Alle anderen: unabhaengig, parallel umsetzbar
```
