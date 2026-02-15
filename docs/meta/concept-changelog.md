# Intraday Agent Konzept — Arbeitsstand

Stand: 2026-02-14

## Was ist passiert

### v1.0 → v1.1 Rewrite (Markdown-first)

ChatGPT hat v1.0 reviewed und substantielles Feedback geliefert. Dieses wurde in v1.1 eingearbeitet.

### v1.1 Nachbesserung (aus ChatGPT-Konversation)

Aus einer separaten ChatGPT-Konversation (`konversation.txt`) wurden praxisnahe Inhalte extrahiert und in das Konzept eingearbeitet. Quelle war eine Diskussion ueber konkretes Intraday-Trading eines V-Reversal-Charts mit ChatGPT als "virtueller Trader".

### v1.1 → v1.4 Iteratives Review mit ChatGPT (4 Runden)

Claude (Opus 4.6) hat das Konzept eigenstaendig ueberarbeitet, mit ChatGPT als ebenbuertigem Sparringspartner. 4 Review-Runden bis zur Konvergenz. ChatGPT bestaetigt: "Das Konzept ist rund."

**Datei:** `intraday-agent-concept.md` — **v1.4 FINAL** (~1100 Zeilen, 23 Kapitel)

Die HTML-Datei (`intraday-agent-concept.html`) ist NICHT aktualisiert — muss aus finalem MD neu generiert werden.

## Eingearbeitete Aenderungen

### Runde 1: v1.0 Review-Feedback (alle 20 Punkte)

#### A) Bug-Fixes
1. `quant_score` durchgaengig 0.0–1.0 normiert (nicht 0–100%)
2. "Kelly" → **Fixed Fractional Risk** mit Begruendung warum kein echtes Kelly
3. `stop_loss_suggestion` aus LLM-Schema entfernt (Stop = ausschliesslich ATR-basiert, Rules-Domaene)
4. R/R ohne Target: Expected Move = 2x ATR als synthetisches Target definiert

#### B) Architektur-Shift
5. LLM = Analyst, nicht Policy-Maker. Kein `action`/`action_confidence` mehr im Schema
6. Neue **Deterministic Strategy Rules Engine** (Kap. 7) mit expliziten Entry/Exit-Regeln pro Regime
7. **Vier-Schichten**-Architektur (Data → Brain[LLM+Rules+Quant] → Execution)
8. Decision Loop angepasst: LLM liefert Kontext → Rules pruefen → Quant validiert → Risk Gate
9. Neues LLM-Schema: `opportunity_zones`, `market_context_signals`, `urgency_level`

#### C) Neue Kapitel
10. Kap. 15: Execution Policy (Limit+Repricing, Pre-Market-Regeln, Slippage-Management)
11. Kap. 16: News Security & Input Validation (Whitelist-Felder, kein Rohtext, Prompt-Injection-Schutz)
12. Kap. 17: Backtesting & Shadow Mode Pipeline (Shadow → Paper → Klein-Live, Walk-Forward)
13. Kap. 18: Model Risk Management (Prompt-Versionierung, temp=0, Eval-Suites, Change-Mgmt)
14. Kap. 19: Security & Isolation (Secrets, Netzwerk-Policy, HMAC-signierte Trade-Intents, Hash-Chain-Logs)
15. Kap. 20: Compliance MiFID II (Art. 17 Mapping, Kill-Switch-Spezifikation)

#### D) Erweiterungen bestehender Kapitel
16. Kap. 12: Data Quality Gates (Stale Quote, Outlier, Time Sync, Bar Completeness, L2 Integrity)
17. Kap. 13: TREND_DOWN = Default nicht handeln, Aggressive Mode nur bei expliziter Aktivierung
18. Kap. 11: Confidence-Kalibrierung auf 100 Handelstage + Bootstrapping-Methode
19. Kap. 21: LULD ueber Close, "cannot liquidate", Pre-Trade Exposure, abnormale Fill-Rate, Quote Staleness
20. Kap. 22: Alert-Routing (4 Level), Eskalationspfade, SLOs, Market Surveillance

### Runde 2: Nachbesserung aus ChatGPT-Konversation (6 Ergaenzungen)

21–27: Trade-Pattern-State-Machines, Scaling-Out, Starter+Add, Pattern-Features, Failure Modes, Vision-Modul, Glossar

### Runde 3: v1.1 → v1.2 (ChatGPT Review Runde 1, 24 Punkte)

28. **PENDING_FILL State** in Zustandsmaschine (saubere Trennung Order gesendet vs. Position offen)
29. **Reject-Semantik** im Decision Loop (kein automatisches Retry, Intent wird final verworfen)
30. **LLM-Criticality-Klassifikation** (welche Aktionen LLM brauchen, welche nicht)
31. **Dynamisches Tages-Budget** (remaining_daily_budget statt pauschaler 3%)
32. **Crash-Detection VOR Outlier-Filter** (Signal-Interferenz-Schutz)
33. **ATR-relative Plausibilitaetsschwellen** (statt fixer Prozentwerte)
34. **Pre-Market als konfigurierbarer Schalter** (Default: off)
35. **Market-Order-Vereinheitlichung** (nur Forced Close + Kill-Switch)
36. **Stop-Market-Fallback-Kriterien** (zeitbasiert)
37. **Ground-Truth-Labeling** fuer Regime-Accuracy
38. **Kostenmodell fuer Backtesting** (IB Tiered, Spread, Slippage)
39. **LLM-Response-Caching** fuer Replay/Backtesting
40. **Repricing-Degradation** bei hoher Cancel/Replace-Rate
41. **Broker-State als Single Source of Truth** fuer Positionsdaten
42. **Timezone-Standardisierung** auf ET (mit DST-Hinweis)
43. **Urgency = Re-Evaluation-Trigger**, nicht Exit-Trigger
44. **Retention** jurisdiktionsabhaengig (Default 5 Jahre)
45. **Transaktionskosten und FX-Awareness**
46. **Schema-Mapping** entry_price_zone.ideal → Execution explizit

### Runde 4: v1.2 → v1.3 (ChatGPT Review Runde 2, 8 P0/P1 Punkte)

47. **Unified Timeout-Policy** (Per-Call 10s, Consecutive Failures 3x → Quant-Only, Prolonged 30min)
48. **State-Naming** POSITIONED → PENDING_FILL/MANAGING_TRADE durchgaengig
49. **OCA-Semantik ueberarbeitet** (Stop SEPARAT von Targets, kein OCA, atomares Fill-Event-Handling)
50. **Kommissionen konsistent** (IB Tiered 0.0035 USD/Share ueberall)
51. **Stress-Multiplier** in Sizing (effective_stop_dist = stop_dist x 1.3 bei HIGH_VOL)
52. **Entry-Rules ATR-relative** (VWAP + 0.5x ATR statt VWAP + 1%)
53. **Kill-Mechanismen harmonisiert** (Heartbeat + Stale-Quote als unabhaengige Trigger)
54. **Instrument-Freeze** bei 07:00 ET (kein Symbolwechsel, kein Ersatzsymbol)

### Runde 5: v1.3 → v1.4 (ChatGPT Review Runde 3+4, finale Praezisierungen)

55. **Entry-Context-Freshness** (max. 2 Min fuer Entries, 15 Min fuer Position-Management)
56. **Market-Order-Policy vollstaendig** (5 explizite Ausnahmefaelle)
57. **Pre-Market State-Klarstellung** (OBSERVING→SEEKING_ENTRY freigeschaltet bei Config-Flag)
58. **Quote-Staleness harmonisiert** (15s/30s/60s Eskalationskette)
59. **LLM-Intervall konsistent** (3/5/15 Min ueberall)
60. **pattern_candidates als Enum** im LLM-Schema (statt Freitext via market_context_signals)
61. **P&L Echtzeit** via IB PnL-Stream, lokale Berechnung als Primaerquelle
62. **Entry-Approaching Event-Trigger** (edge-triggered, 90s Debounce, Single-Flight, Freshness-Gate hat Vorrang)
63. **Repricing-Guard** (darf entry_price_zone.max nicht ueberschreiten)
64. **FX-Conversion** explizit in Sizing-Formel (EUR→USD via Spot)
65. **entry_price_zone vs opportunity_zones** Beziehung klargestellt
66. **hold_duration_bars** = 5-Min-Bars als Referenz-Timeframe
67. **Regime-Wechsel** Event-Trigger beschleunigen Bestaetigung bei langen Intervallen

### Runde 7: v1.4 → v1.5 (Benutzer-Input, 3 Punkte)

68. **MiFID II → Operationelle Sicherheit** Kap. 20 pragmatisiert: Kein regulatorisches Framing (Privatbetrieb), Kill-Switch/Audit bleiben als gutes Engineering
69. **Tranchen budgetabhaengig** Kap. 7+15: 3T (< 5K), 4T (5–15K), 5T (> 15K) — Schwellenwerte als Properties
70. **Shadow Mode entfernt** Kap. 17: Zweistufig (Paper → Klein-Live), Papertrading als Startpunkt

## Naechste Schritte

1. ~~**HTML-Konvertierung:**~~ ERLEDIGT (2026-02-14). Dark-Theme mit Mermaid, Callouts, Tags.
2. **HTML-Aktualisierung:** HTML muss aus v1.5 MD neu generiert werden.
3. **Architektur-Entscheidung:** Konzept dem Benutzer vorlegen, Freigabe fuer Implementierungsplanung einholen.

## Dateien

| Datei | Status |
|-------|--------|
| `intraday-agent-concept.md` | v1.5 — Benutzer-Feedback eingearbeitet (3 Punkte) |
| `intraday-agent-concept.html` | v1.4 VERALTET — muss aus v1.5 MD neu generiert werden |
| `konversation.txt` | ChatGPT-Konversation, Inhalte in v1.1 eingearbeitet |

## Review-Historie

| Runde | Von | An | Ergebnis |
|-------|-----|-----|---------|
| R1 (v1.0) | ChatGPT | Claude | 20 Punkte → alle eingearbeitet in v1.1 |
| R2 (v1.1) | ChatGPT-Konversation | Claude | 6 Ergaenzungen → eingearbeitet in v1.1 |
| R3 (v1.1) | ChatGPT Review | Claude | 24 Punkte → eingearbeitet in v1.2 |
| R4 (v1.2) | ChatGPT Review | Claude | 8 P0/P1 → eingearbeitet in v1.3 |
| R5 (v1.3) | ChatGPT Review | Claude | 7 Punkte → eingearbeitet in v1.4 |
| R6 (v1.4) | ChatGPT Review | Claude | 3 Praezisierungen → eingearbeitet. **"Konzept ist rund"** |
| R7 (v1.4) | Benutzer | Claude | 3 Punkte (MiFID pragmatisiert, Tranchen flexibel, Shadow→Paper) → v1.5 |
