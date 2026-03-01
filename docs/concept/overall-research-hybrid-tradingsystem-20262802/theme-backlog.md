# ODIN -- Priorisierte Themenliste (Research-Synthese)

**Datum:** 2026-02-28
**Basis:** 5 Research-Berichte (3x eigene Agents, ChatGPT Deep Research, Gemini Deep Research)
**Zweck:** Entscheidungsvorlage fuer Stakeholder-Freigabe vor User-Story-Erstellung

## Uebersicht

| # | Thema | Prio | Aufwand | Quellen | Umzusetzten (ja/nein/vielleicht) | 
|---|-------|------|---------|---------|-----------------------| 
| 1 | Intraday-Seasonality-Normalisierung (Expected Curves) | P0 | M | ChatGPT, Gemini | ja | 
| 2 | Backtesting-Robustheit (PBO, DSR, Monte Carlo) | P0 | L | Alle 5 | ja | 
| 3 | Execution-Quality-Feedback (TCA / Implementation Shortfall) | P0 | L | ChatGPT, Gemini | ja | 
| 4 | HMM-basierte Regime-Erkennung (probabilistisch) | P1 | L | Alle 5 | ja | 
| 5 | News-Integration mit Catalyst-Impact-Classifier | P1 | L | Research-LLM, ChatGPT, Gemini | nein | 
| 6 | Order Flow Imbalance (OFI) als Microstructure-Gate | P1 | L | ChatGPT, Gemini | ja | 
| 7 | LLM-Confidence-Kalibrierung (Platt Scaling / Brier Score) | P1 | M | Alle 5 | ja | 
| 8 | Post-Trade Counterfactual-Analyse (Quant-Only vs. Hybrid) | P1 | M | Research-Hybrid, ChatGPT | ja | 
| 9 | Anthropic Prompt-Caching fuer Tier-1-System-Prompt | P1 | S | Research-LLM | nur wenn mit Agent-SDK umsetzbar | 
| 10 | Chain-of-Thought-Forcing im System-Prompt | P1 | S | Research-LLM, ChatGPT | ja | 
| 11 | Pre-Trade Hard Controls (Max Order Size, Price Collars, Throttle) | P1 | M | ChatGPT, Gemini | ja | 
| 12 | Regime-abhaengige Arbiter-Gewichtung | P2 | M | Research-Hybrid, ChatGPT | ja | 
| 13 | Volume Profile (VPOC/VAH/VAL) als S/R-Quelle | P2 | M | Research-Quant, ChatGPT | ja | 
| 14 | Korrelations-Check fuer Multi-Instrument-Risiko | P2 | S | Research-Quant, Research-Hybrid | ja | 
| 15 | Gradierte Fallback-Hierarchie (5-Stufen-Degradation) | P2 | S | Research-Hybrid, ChatGPT | ja | 
| 16 | Explicit Multi-Timeframe-Labeling im LLM-Prompt | P2 | S | Research-LLM | ja | 
| 17 | ATR-Periode Intraday-Optimierung (14 vs. 10 vs. 7) | P2 | S | Research-Quant, ChatGPT | ja | 
| 18 | Prompt-Injection-Haertung fuer News/RAG-Content | P2 | M | Research-LLM, ChatGPT, Gemini | nein | 
| 19 | Execution-Policy-Schicht (Order-Typ-Selektion) | P2 | L | ChatGPT, Gemini | ja | 
| 20 | VPIN (Volume-Synchronized Probability of Informed Trading) | P3 | XL | Gemini | ja | 
| 21 | Vision-Modul (Chart-Screenshot an LLM) | P3 | L | Research-LLM | nein | 
| 22 | Adaptive Gewichtung via Bandit-Algorithmen | P3 | XL | Research-Hybrid | ja | 

---

## Thema 1: Intraday-Seasonality-Normalisierung (Expected Curves)

**Prioritaet:** P0 | **Aufwand:** M

**Was ist das?**
Intraday-Maerkte zeigen robuste periodische Muster: Volumen und Volatilitaet folgen einer U-foermigen Kurve (hoch am Open/Close, tief am Mittag). Ohne Beruecksichtigung dieser Tagesstruktur erzeugen KPIs wie `volumeRatio` und `atrDecayRatio` systematische Fehlsignale -- z.B. wird Mittags-Ruhe als Anomalie interpretiert, waehrend Opening-Volume faelschlich als Spike gewertet wird.

**IST-Zustand in ODIN:**
`volumeRatio` wird gegen SMA-20 berechnet, `atrDecayRatio` vergleicht aktuelle ATR mit gleitendem Durchschnitt. Beide sind nicht um die Tageszeit normalisiert. Das Volume-Gate und ATR-Gates arbeiten mit absoluten Vergleichen ohne Saisonalitaetskorrrektur.

**SOLL-Zustand (Research-Empfehlung):**
"Expected Curves" pro Instrument oder Cluster schaetzen: `expectedVolume(t)`, `expectedVolatility(t)`, `expectedSpread(t)` je 5-Min-Slot (Rolling 20-60 Handelstage). KPIs werden dann als `volumeSurprise = volume / expectedVolume(slot)` berechnet. Das macht Gates stabil ueber den gesamten Handelstag.

**Warum umsetzen?**
Eliminiert systematische Fehlsignale durch Tageszeit-Effekte. Reduziert Overfiltering am Open und falsches Passing in illiquiden Mittagsphasen. Direkte Verbesserung der Gate-Kaskaden-Qualitaet ohne Aenderung der Architektur.

**Quellen:** ChatGPT (P0, "stabilisiert Gates massiv"), Gemini (Kap. 3, Andersen/Bollerslev-Referenz)

---

## Thema 2: Backtesting-Robustheit (PBO, DSR, Monte Carlo)

**Prioritaet:** P0 | **Aufwand:** L

**Was ist das?**
ODINs Regelwerk hat viele Freiheitsgrade (7 Gates, 4 Pattern-FSMs, 20 Subregimes, LLM-Tactical-Controls). Ohne formale Overfitting-Kontrolle ist die Wahrscheinlichkeit hoch, dass Backtest-Ergebnisse ueberschaetzt werden. Combinatorial Purged Cross-Validation (CPCV), Deflated Sharpe Ratio (DSR) und Monte Carlo sind die akademischen Standardmethoden dagegen.

**IST-Zustand in ODIN:**
Backtesting via `BacktestRunner` ist event-basiert und reproduzierbar. Kein CPCV, kein DSR, kein Monte Carlo, kein Neighbor-Test fuer Parameter-Stabilitaet. Ergebnisse werden als Sharpe/Return/Win-Rate berichtet ohne Korrektur fuer Multiple Testing.

**SOLL-Zustand (Research-Empfehlung):**
Post-Processing-Schritt nach jedem Backtest-Lauf: (1) Parameter-Grid ueber kritische Parameter, (2) Neighbor-Test (+-10%), (3) Monte Carlo (Trade-Order-Shuffle, Parameter-Jitter), (4) DSR-Berechnung, (5) PBO als Wahrscheinlichkeitsmass fuer Overfitting.

**Warum umsetzen?**
Ohne Overfitting-Kontrolle sind alle Backtest-Ergebnisse wertlos (>90% der Strategien scheitern live laut Literatur). DSR und PBO sind die einzigen Werkzeuge, die statistische Konfidenz in Backtest-Ergebnisse bringen. Essentiell bevor Parameter-Optimierungen als "validiert" gelten.

**Quellen:** Research-Quant (Empfehlung 1, ausfuehrlich), Research-Hybrid (Abschnitt 6), ChatGPT (P1 Forschungshygiene), Gemini (Kap. 7, "90% scheitern"), Research-LLM (FINSABER-Referenz)

---

## Thema 3: Execution-Quality-Feedback (TCA / Implementation Shortfall)

**Prioritaet:** P0 | **Aufwand:** L

**Was ist das?**
Transaction Cost Analysis (TCA) misst die Differenz zwischen dem theoretischen Entscheidungspreis und dem tatsaechlichen Fill-Preis -- inklusive Slippage, Spread und Market Impact. Implementation Shortfall (IS) ist die Leitmetrik. Ohne dieses Feedback weiss ODIN nicht, ob Signale nach Kosten profitabel sind.

**IST-Zustand in ODIN:**
OMS erzeugt Orders und verwaltet Fills. Slippage wird im Backtest nicht realistisch modelliert. Kein systematisches Tracking von Arrival-Price vs. Fill-Price. Kein Feedback von Execution-Qualitaet zurueck in Entscheidungslogik (RiskGate, Arbiter).

**SOLL-Zustand (Research-Empfehlung):**
Pro Order/Trade: Implementation Shortfall persistieren (Decision-Price vs. Fill-Price). Effective Spread und Slippage-Metriken erfassen. Realistisches Fill-Modell im Backtest (Spread-basiert, nicht Mid-Price). Langfristig: Execution-KPIs als Feedback in SizeModifier und Liquidity-Gate.

**Warum umsetzen?**
"Intraday-Edge stirbt in der Praxis fast immer an Execution, Slippage, Spread und Market Impact -- nicht an zu wenigen Indikatoren." (ChatGPT). Ohne TCA koennen Backtests systematisch zu optimistisch sein. Das ist die groesste einzelne Quelle fuer Diskrepanz zwischen Simulation und Live.

**Quellen:** ChatGPT (P0, "entscheidend, sonst sind Backtests wertlos"), Gemini (Kap. 4, Almgren-Chriss, IS als institutionelle Benchmark)

---

## Thema 4: HMM-basierte Regime-Erkennung (probabilistisch)

**Prioritaet:** P1 | **Aufwand:** L

**Was ist das?**
Hidden Markov Models (HMM) ersetzen oder ergaenzen die deterministische Regime-Klassifikation (EMA/ADX/VWAP-Schwellen) durch ein probabilistisches Modell. Statt "Regime = TREND_UP" liefert HMM "P(TREND_UP) = 0.73, P(RANGE) = 0.22, P(BEAR) = 0.05" -- direkt als kalibrierte `regimeConfidence` verwendbar.

**IST-Zustand in ODIN:**
`RegimeResolver` bestimmt Regime deterministisch (EMA-Kreuzung + ADX + VWAP). Regime-Confidence wird heuristisch aus Signalstaerke abgeleitet, nicht empirisch kalibriert. Hysterese (2-Bar-Confirmation) verhindert Flapping.

**SOLL-Zustand (Research-Empfehlung):**
3-State-HMM (Bull/Neutral/Bear) auf 5-Min-Returns trainieren. Forward-Algorithmus fuer Regime-Wahrscheinlichkeiten. P(Regime|Observations) direkt als `regimeConfidence`. Literatur: Sharpe 1.9, Max-Drawdown von 56% auf 24% reduziert (QuantConnect). Statistical Jump Models als Alternative mit besserer Stabilitaet.

**Warum umsetzen?**
Eliminiert scharfe Regime-Grenzen (binaer statt graduell), liefert kalibrierte Unsicherheitsquantifizierung, und macht Hysterese als Heuristik obsolet. Der groesste identifizierte Gap zwischen ODINs deterministischem Ansatz und dem akademischen State of the Art.

**Quellen:** Research-Quant (Kap. 5.2, ausfuehrlich), Research-Hybrid (Kap. 2.3, Dynamic Factor Allocation), ChatGPT (P0 in Roadmap), Gemini (Kap. 6.1, "institutionelles Instrument der Wahl"), Research-LLM (Kap. 3.1)

---

## Thema 5: News-Integration mit Catalyst-Impact-Classifier

**Prioritaet:** P1 | **Aufwand:** L

**Was ist das?**
Real-Time-News-Feed (z.B. Benzinga, Tiingo) mit zweistufiger Verarbeitung: (1) FinBERT-Schnellfilter fuer Sentiment + Relevanz (<10ms, lokal), (2) starke Nachrichten als strukturierter Catalyst-Kontext (`catalyst_direction`, `catalyst_magnitude`) in Tier-2 des LLM-Prompts.

**IST-Zustand in ODIN:**
Code-Infrastruktur vorbereitet (`odin.llm.news.enabled=false`). Kein News-Feed aktiv. LLM-Analyst hat keinen Zugang zu aktuellen Nachrichten. Gap-Up-Situationen (z.B. IREN 23.02) werden ohne Catalyst-Kontext analysiert.

**SOLL-Zustand (Research-Empfehlung):**
News-Feed aktivieren. FinBERT als Vorfilter. Neues Enum-Feld `catalyst_context` im LlmTacticalOutput. Prompt-Injection-Schutz (Thema 18) muss VOR News-Aktivierung implementiert sein. Social-Media-Aggregation (Reddit/Twitter) als spaetere Erweiterung.

**Warum umsetzen?**
ECC-Analyzer-Studie (ICAIF 2024): Dreifache Erklaerungskraft fuer kurzfristige Renditen gegenueber Indikatoren allein. News-getriebene Gap-Ups und Earnings-Surprises sind die staerksten kurzfristigen Signale -- ohne News-Kontext ist der LLM-Analyst bei diesen Situationen blind.

**Quellen:** Research-LLM (Empfehlung 1, HOCH), ChatGPT (Gap-Analyse), Gemini (Kap. 2.3, Architektur-Layer)

---

## Thema 6: Order Flow Imbalance (OFI) als Microstructure-Gate

**Prioritaet:** P1 | **Aufwand:** L

**Was ist das?**
OFI quantifiziert das Ungleichgewicht zwischen Bid- und Ask-Seite im Orderbuch. Statt nur den Spread zu pruefen (ODINs aktuelles Spread-Gate), misst OFI den tatsaechlichen Kaufdruck/Verkaufsdruck auf Best-Bid/Best-Ask-Level. Damit wird das Gate-System "execution-aware" -- es erkennt Adverse-Selection-Situationen bevor sie im Preis sichtbar werden.

**IST-Zustand in ODIN:**
Spread-Gate prueft Bid-Ask-Spread als Hard-Veto. Volume-Gate prueft Volume-Ratio gegen SMA. Kein Zugriff auf Orderbuch-Tiefe oder Tick-Level-Daten. Gate-Kaskade arbeitet ausschliesslich mit Bar-basierten Indikatoren.

**SOLL-Zustand (Research-Empfehlung):**
OFI auf Best-Level als zusaetzlichen KPI (pro 1m-Bar oder Event-basiert). Z-Score-Normalisierung mit rollierendem Fenster. Integration als zusaetzliches Gate oder als `elevated`-Modifier in bestehenden Gates. Depth-at-Best als ergaenzendes Liquidity-Signal.

**Warum umsetzen?**
Cont/Kukanov/Stoikov zeigen lineare Beziehung zwischen OFI und Preisaenderungen. OFI erkennt "catching falling knives"-Situationen und schuetzt vor Entries in toxischen Orderflow. Die natuerliche Erweiterung des existierenden Spread-Gates zu einem echten Liquidity-Gate.

**Quellen:** ChatGPT (P1, "Edge/Robustheit"), Gemini (Kap. 3.1, OFI/MLOFI ausfuehrlich, Cont et al.)

**Abhaengigkeit:** Erfordert Top-of-Book-Daten (Best Bid/Ask + Size) von IB TWS API. Pruefen ob `IbMarketDataFeed` diese Daten bereits liefert oder erweitert werden muss.

---

## Thema 7: LLM-Confidence-Kalibrierung (Platt Scaling / Brier Score)

**Prioritaet:** P1 | **Aufwand:** M

**Was ist das?**
LLMs sind systematisch ueberoptimistisch kalibriert -- ein `regime_confidence: 0.8` entspricht in der Realitaet oft nur 60-65% Korrektheit. Platt Scaling oder isotonische Regression auf historischen Backtests korrigiert diese Verzerrung. Brier Score und ECE (Expected Calibration Error) als Pflicht-KPIs.

**IST-Zustand in ODIN:**
Rohe LLM-Confidence-Werte werden direkt in Schwellenlogik verwendet (0.5 fuer Regime-Nutzung, 0.7 fuer elevated Gate-Modus). Kein Tracking ob `regime_confidence: 0.8` tatsaechlich 80% korrekt ist. Schwellen basieren auf Design-Annahmen, nicht auf Empirie.

**SOLL-Zustand (Research-Empfehlung):**
Nach 100+ Backtests: Pro Regime (reported_confidence, war_regime_korrekt_nach_3_Bars) als Trainingsdaten. Isotonische Regression -> Kalibrierungsfunktion. Kalibrierte Confidence ersetzt rohe im Arbiter. Brier Score und Reliability Diagrams als Monitoring-KPIs.

**Warum umsetzen?**
Ohne Kalibrierung sind alle Confidence-Schwellen im Arbiter bestenfalls zufaellig korrekt. ChatGPT: "Wenn 0.7 vertrauenswuerdig bedeutet, aber das Modell diese Zahl nicht zuverlaessig als Wahrscheinlichkeit meint, ist deine Gate-Strenge zufaellig."

**Quellen:** Research-Hybrid (Empfehlung 1, Kap. 4), Research-LLM (Empfehlung 4), Research-Quant (Kap. 2.2), ChatGPT (P2 LLM Industrialization), Gemini (Kap. 8, Brier/ECE)

**Abhaengigkeit:** Erfordert ausreichend Backtest-Daten mit vollstaendigem EventLog (>100 LLM-Analysen mit Outcome).

---

## Thema 8: Post-Trade Counterfactual-Analyse (Quant-Only vs. Hybrid)

**Prioritaet:** P1 | **Aufwand:** M

**Was ist das?**
Systematisches Messen, ob die LLM-Integration netto positiv wirkt. Pro Trade wird zusaetzlich geloggt: Was haette Quant allein entschieden? Was haette LLM allein entschieden? Welche Seite hat dominiert? Aggregierte Auswertung zeigt den echten LLM-Mehrwert nach Regime.

**IST-Zustand in ODIN:**
`DecisionEvent` im EventLog enthaelt `QuantVote`, `LlmVote`, `FusionResult` mit `winnerSide`. Alle notwendigen Daten sind grundsaetzlich vorhanden. Kein dediziertes Analyse-Tool fuer Counterfactual-Vergleiche. Kein systematisches Performance-Tracking nach Quant-Only vs. Hybrid.

**SOLL-Zustand (Research-Empfehlung):**
Im DecisionEvent zusaetzlich loggen: `quantOnlyDecision` (was haette Quant-Only entschieden?) und `llmOnlyDecision`. Analyse-Tool das pro Regime vergleicht: Quant-Only-Return vs. Hybrid-Return. Basis fuer Shapley-Value-Attribution und adaptive Gewichtung.

**Warum umsetzen?**
"Ablation als Pflicht" (ChatGPT). Ohne dieses Tracking ist unbekannt, ob der LLM echten Mehrwert liefert oder nur Komplexitaet hinzufuegt. Essentielle Grundlage fuer Thema 12 (Regime-abhaengige Gewichtung) und Thema 22 (Bandit-Algorithmen).

**Quellen:** Research-Hybrid (Empfehlung 4, ausfuehrlich), ChatGPT (LLM-Value-Proof)

---

## Thema 9: Anthropic Prompt-Caching fuer Tier-1-System-Prompt

**Prioritaet:** P1 | **Aufwand:** S

**Was ist das?**
Anthropic bietet natives Prompt-Caching. Der fixe System-Prompt (Tier 1, ~2.000 Tokens) wird gecacht -- nur dynamische Teile (Tier 2+3) werden pro Call neu berechnet. Cache lebt 5 Minuten und wird bei aktiven Calls automatisch refresht.

**IST-Zustand in ODIN:**
`ClaudeAnalystClient` sendet den vollstaendigen Prompt (Tier 1+2+3) bei jedem Call. Kein Prompt-Caching aktiviert.

**SOLL-Zustand (Research-Empfehlung):**
`cache_control: {type: "ephemeral"}` im API-Request-Body fuer den System-Prompt-Teil. Einzeiler-Aenderung im Request-Builder.

**Warum umsetzen?**
~30-40% Kostenreduktion bei Claude-API-Calls plus Latenzverbesserung. Bei 78+ Calls pro Instrument pro Tag summiert sich das. Minimaler Aufwand, sofortiger ROI.

**Quellen:** Research-LLM (Empfehlung 2)

---

## Thema 10: Chain-of-Thought-Forcing im System-Prompt

**Prioritaet:** P1 | **Aufwand:** S

**Was ist das?**
System-Prompt explizit anweisen: "Fuelle zuerst das reasoning-Feld mit schrittweiser Analyse (Kontext -> Indikatoren -> Muster -> Regime -> Entscheidung) bevor andere Felder befuellt werden." Die bestehenden `reasoning`- und `key_observations`-Felder werden aktiv als Denkprozess-Erzwinger genutzt.

**IST-Zustand in ODIN:**
`reasoning` und `key_observations` sind Logging-Only-Felder im LlmTacticalOutput. Kein explizites Gebot im System-Prompt, diese vor den Decision-Feldern zu befuellen.

**SOLL-Zustand (Research-Empfehlung):**
Prompt-Aenderung in `PromptBuilder.buildSystemPrompt()`. Keine Code-Strukturaenderungen. Prompt-Version-Increment triggert Cache-Invalidierung.

**Warum umsetzen?**
Studien zeigen +5 bis +17% Alpha durch CoT gegenueber direkten Vorhersagen. CoT reduziert inkonsistente Feld-Kombinationen im Output (z.B. TREND_DOWN + Entry-Zone). Minimaler Aufwand fuer messbaren Qualitaetsgewinn.

**Quellen:** Research-LLM (Empfehlung 3, Kim et al. 2024)

---

## Thema 11: Pre-Trade Hard Controls (Max Order Size, Price Collars, Throttle)

**Prioritaet:** P1 | **Aufwand:** M

**Was ist das?**
Systemische Pre-Trade-Kontrollen unabhaengig vom Strategy-Code: Maximale Ordergroesse (Fat-Finger-Schutz), Preis-Collars (keine Orders ausserhalb dynamischer Baender), Order-Rate-Limits (Schutz vor Schleifen/duplizierten Orders), maximale Intraday-Position.

**IST-Zustand in ODIN:**
Risk-Gate prueft Position Sizing, R/R und globale Account-Limits. Kill-Switch und DAY_STOPPED vorhanden. Keine expliziten Fat-Finger-Limits, keine Preis-Collars, keine Order-Rate-Limits.

**SOLL-Zustand (Research-Empfehlung):**
Im Risk-Gate (odin-execution): Max Order Size als Hard-Reject, dynamische Price Collars (z.B. nicht ausserhalb VWAP +/- 5*ATR), Order Rate Limit pro Pipeline (max N Orders pro Minute), Max Intraday Position.

**Warum umsetzen?**
Schutz vor systematischen Fehlern im Strategy-Code (der haeufigste Fehlerort laut FIA Best Practices). Verhindert Runaway-Algos, duplizierte Orders und Fat-Finger-Fehler. Geringe Komplexitaet, hoher Schutzwert.

**Quellen:** ChatGPT (P1 Operational), Gemini (Kap. 8, Risk Controls)

---

## Thema 12: Regime-abhaengige Arbiter-Gewichtung

**Prioritaet:** P2 | **Aufwand:** M

**Was ist das?**
Statt Quant und LLM in allen Regimes als gleichwertige Partner zu behandeln, werden die Gewichte regime-abhaengig: In klarem Trend dominiert Quant (messbare Trendmetriken), in unklaren/News-getriebenen Situationen hat der LLM-Kontext mehr Gewicht.

**IST-Zustand in ODIN:**
Konservativitaetsprinzip: "Konservativster Wert gewinnt" -- symmetrisch in allen Regimes. Dual-Key Entry und Parameter-Fusion behandeln beide Seiten gleichwertig.

**SOLL-Zustand (Research-Empfehlung):**
`RegimeWeightProfile`: TREND_UP (Quant 0.70, LLM 0.30), RANGE_BOUND (0.50/0.50), HIGH_VOL (0.40/0.60), UNCERTAIN (0.60/0.40). Gewichte beeinflussen Confidence-Fusion und Parameter-Selektion, NICHT die Dual-Key-Regel.

**Warum umsetzen?**
"Der wichtigste ungenutzte Hebel" laut Research-Hybrid. Statische Gewichte sind suboptimal -- aber erst sinnvoll umsetzbar NACH Thema 8 (Counterfactual-Analyse liefert die empirische Basis).

**Quellen:** Research-Hybrid (Empfehlung 2, "Prioritaet Hoch"), ChatGPT (implizit in Regime-Diskussion)

**Abhaengigkeit:** Erfordert empirische Validierung durch Thema 8 (Post-Trade Counterfactual).

---

## Thema 13: Volume Profile (VPOC/VAH/VAL) als S/R-Quelle

**Prioritaet:** P2 | **Aufwand:** M

**Was ist das?**
Volume Point of Control (VPOC) ist das Preislevel mit dem meisten kumulativen Volumen in der Session. Value Area High (VAH) und Value Area Low (VAL) begrenzen die Zone, in der 70% des Volumens gehandelt wurde. VPOC fungiert als starkes S/R-Level.

**IST-Zustand in ODIN:**
S/R-Engine nutzt Vortageshoch/-tief, Pre-Market, VWAP, runde Zahlen, Intraday-Konsolidierungszonen. Kein Volume Profile, kein VPOC.

**SOLL-Zustand (Research-Empfehlung):**
VPOC aus Minuten-Bars berechnen (kumulatives Volumen pro Preislevel). Als S/R-Level in SrEngine einspeisen. Taeglicher Reset wie VWAP. "80%-Regel": Preis der aus der Value Area austritt, rotiert mit 80% Wahrscheinlichkeit zur anderen Seite.

**Warum umsetzen?**
VPOC ist das institutionell am staerksten beachtete Volume-basierte Level. Ergaenzt die bestehende S/R-Engine um eine informationstheoretisch unabhaengige Quelle (Volumen-Konzentration statt Preis-Reaktion).

**Quellen:** Research-Quant (Empfehlung 3), ChatGPT (implizit in Microstructure)

---

## Thema 14: Korrelations-Check fuer Multi-Instrument-Risiko

**Prioritaet:** P2 | **Aufwand:** S

**Was ist das?**
Beim Start einer neuen Position pruefen, ob bestehende offene Positionen hochkorreliert sind (z.B. beide Tech-Aktien). Bei Korrelation > Schwelle (z.B. 0.7) wird neuer Entry blockiert oder SizeModifier reduziert.

**IST-Zustand in ODIN:**
`GlobalRiskManager` verwaltet Account-Limits ueber alle Pipelines. Keine explizite Korrelationsberechnung zwischen simultanen Positionen.

**SOLL-Zustand (Research-Empfehlung):**
Rolling-Korrelation der letzten N 5-Min-Returns zwischen allen offenen Instrumenten. Blockiere oder reduziere Entry bei hoher Korrelation. Implementierung im GlobalRiskManager oder RiskGate.

**Warum umsetzen?**
Bei 2-3 parallelen Positionen in hochkorrelierten Aktien kann ein sektorweiter Einbruch alle Positionen gleichzeitig stoppen. Einfach umzusetzen, adressiert ein reales Risiko.

**Quellen:** Research-Quant (Empfehlung 5), Research-Hybrid (Kap. 4.4, Korrelationsrisiko)

---

## Thema 15: Gradierte Fallback-Hierarchie (5-Stufen-Degradation)

**Prioritaet:** P2 | **Aufwand:** S

**Was ist das?**
Statt binaer (LLM verfuegbar / Quant-Only) eine 5-stufige Degradation: (1) Vollbetrieb, (2) LLM-Output aelter als TTL aber vorhanden (reduzierte Confidence), (3) Quant-Only mit erhoehten Schwellen, (4) Exit-Only-Modus, (5) Safe-Mode/Trade-Halt.

**IST-Zustand in ODIN:**
Effektiv 3 Stufen: Normal, Quant-Only (Circuit Breaker), Trade-Halt (>30 Min). TTL-Gate ist binaer (frisch oder nicht). Kein Exit-Only-Modus als explizite Zwischenstufe.

**SOLL-Zustand (Research-Empfehlung):**
Freshness-Confidence-Decay statt binaeres TTL-Gate: <120s = 1.0, 120-300s = 0.7, 300-600s = 0.4, >600s = 0.0. Exit-Only als expliziter Modus bei 3+ Timeouts (bestehende Positionen schliessen, keine neuen).

**Warum umsetzen?**
Verhindert abruptes Verhalten beim TTL-Ablauf. Inkrementelle Verbesserung, geringe Komplexitaet. Macht das System robuster bei intermittierenden LLM-Latenzen.

**Quellen:** Research-Hybrid (Empfehlung 3), ChatGPT (implizit in Degradation)

---

## Thema 16: Explicit Multi-Timeframe-Labeling im LLM-Prompt

**Prioritaet:** P2 | **Aufwand:** S

**Was ist das?**
OHLCV-Bars in Tier-3-Kontext klar mit Timeframe-Labels versehen: "=== MICRO CONTEXT (1m) ===", "=== DECISION CONTEXT (5m) ===", "=== DAILY BIAS ===". Prompt-Engineering-Forschung zeigt messbare Verbesserung durch explizite Strukturierung.

**IST-Zustand in ODIN:**
Bars werden im Prompt uebergeben, aber ohne explizite Hierarchie-Beschriftung die den Timeframe-Zweck kommuniziert.

**SOLL-Zustand (Research-Empfehlung):**
Kosmetische Aenderung in `PromptBuilder.buildTier3Context()`. Prompt-Version-Increment notwendig.

**Warum umsetzen?**
Studien zeigen: Explizite Strukturierung verbessert LLM-Verstaendnis messbar. Minimaler Aufwand, kein Risiko.

**Quellen:** Research-LLM (Empfehlung 5)

---

## Thema 17: ATR-Periode Intraday-Optimierung (14 vs. 10 vs. 7)

**Prioritaet:** P2 | **Aufwand:** S

**Was ist das?**
ATR(14) auf 5-Min-Bars entspricht 70 Minuten Look-Back -- moeglicherweise zu traege fuer Intraday. ATR(10) = 50 Min, ATR(7) = 35 Min. Analog: ADX(14) vs. ADX(7-10), RSI(14) vs. RSI(9-10).

**IST-Zustand in ODIN:**
ATR(14), ADX(14), RSI(14) auf 5-Min-Bars. Standard-Perioden, nicht intraday-optimiert.

**SOLL-Zustand (Research-Empfehlung):**
Backtest-Vergleich: ATR(14) vs. ATR(10) vs. ATR(7) auf historischen Daten. Neighbor-Test beider Werte. Ggf. Anpassung von ADX und RSI-Perioden parallel.

**Warum umsetzen?**
Hoehere Responsivitaet auf aktuelle Volatilitaet, bessere Anpassung an Intraday-Zeitfenster. Einfach per Konfiguration testbar (BrainProperties). Aber: Hoehere Responsivitaet != besser -- daher Backtest-Pflicht.

**Quellen:** Research-Quant (Kap. 7.2, Kap. 1.1)

---

## Thema 18: Prompt-Injection-Haertung fuer News/RAG-Content

**Prioritaet:** P2 | **Aufwand:** M

**Was ist das?**
Wenn News-Content in den LLM-Prompt fliesst (Thema 5), muss dieser gegen Prompt-Injection geschuetzt werden. Klare Separator-Token zwischen System-Kontext (TRUSTED) und externem Content (UNTRUSTED), Content-Sanitizing, Attack-Simulation als Test.

**IST-Zustand in ODIN:**
News-Feature noch deaktiviert. Kein externer Content im Prompt. Safety Contract im System-Prompt vorhanden. Kein Prompt-Injection-Schutz implementiert.

**SOLL-Zustand (Research-Empfehlung):**
Input-Isolation: "[EXTERNAL DATA -- UNTRUSTED]"-Marker. Content-Sanitizing fuer News-Artikel. Prompt-Injection-Testcases als Unit-Tests. LLM bleibt ausserhalb der Trust Boundary (bounded Enums, kein Tool-Zugriff).

**Warum umsetzen?**
OWASP und NCSC warnen: Prompt-Injection ist moeglicherweise nie perfekt mitigierbar. Systeme muessen so gebaut sein, dass kompromittierte Outputs moeglichst wenig Schaden anrichten. MUSS VOR Thema 5 (News) implementiert sein.

**Quellen:** Research-LLM (Kap. 5.1), ChatGPT (Security), Gemini (Kap. 8, OWASP)

**Abhaengigkeit:** Blocker fuer Thema 5 (News-Integration).

---

## Thema 19: Execution-Policy-Schicht (Order-Typ-Selektion)

**Prioritaet:** P2 | **Aufwand:** L

**Was ist das?**
Deterministische Schicht die je nach Liquidity-Regime (Spread, Depth, OFI, Volume-Curve) zwischen Order-Typen entscheidet: Marketable Limit, Passive Limit, Aggressive Limit, Market -- plus Slicing/Retry-Logik.

**IST-Zustand in ODIN:**
OMS erzeugt Bracket-Orders (Entry + Stop + Target). Keine adaptive Order-Typ-Selektion basierend auf Liquiditaetsbedingungen.

**SOLL-Zustand (Research-Empfehlung):**
Execution-Policy als determinstische Schicht zwischen Arbiter und OMS. Input: Spread, Depth, Volume-Regime, Urgency. Output: Order-Typ + Preis-Offset + Max-Slippage. Vereinfachte Almgren-Chriss-Prinzipien (Slicing bei groesseren Orders).

**Warum umsetzen?**
Reduziert systematisch Slippage. Aber: Fuer ODINs Groessenordnung (2-3 Aktien, moderate Positionsgroessen) ist der Mehrwert gegenueber einfachen Limit-Orders begrenzt. Erst relevant wenn TCA (Thema 3) zeigt, dass Execution ein Problem ist.

**Quellen:** ChatGPT (Execution-Policy), Gemini (Kap. 4, Almgren-Chriss)

**Abhaengigkeit:** Erfordert Daten aus Thema 3 (TCA) um Notwendigkeit zu begruenden.

---

## Thema 20: VPIN (Volume-Synchronized Probability of Informed Trading)

**Prioritaet:** P3 | **Aufwand:** XL

**Was ist das?**
VPIN quantifiziert Orderflow-Toxizitaet: Wie wahrscheinlich ist es, dass informierte institutionelle Akteure den Orderflow dominieren? Nutzt Volume-Buckets statt Zeitintervalle. Steigender VPIN = toxischer Orderflow = Stop-Adjustierung oder Trade-Halt.

**IST-Zustand in ODIN:**
Nicht implementiert. Kein Konzept fuer Orderflow-Toxizitaet.

**SOLL-Zustand (Research-Empfehlung):**
VPIN als Volume-Bucket-basierter KPI. Circuit-Breaker-aehnliche Logik: VPIN ueber Schwelle -> keine neuen Entries, Stop-Toleranzen anpassen.

**Warum umsetzen?**
Akademisch hochrelevant (Flash Crash 2010). Aber: Erfordert Tick-Level-Daten und Volume-Bucket-Segmentierung -- aufwaendig zu implementieren. Fuer ODINs Groessenordnung (2-3 Aktien Long-Only) ist der Mehrwert fragwuerdig.

**Quellen:** Gemini (Kap. 3.2, ausfuehrlich)

---

## Thema 21: Vision-Modul (Chart-Screenshot an LLM)

**Prioritaet:** P3 | **Aufwand:** L

**Was ist das?**
Chart-Screenshot (PNG) an multimodales LLM senden fuer visuelle S/R-Level-Bestaetigung oder Pattern-Erkennung. Nicht als primaeres Signal, sondern als optionale Validierungsschicht.

**IST-Zustand in ODIN:**
Fuer spaetere Version geplant, nicht implementiert. Code-Infrastruktur nicht vorbereitet.

**SOLL-Zustand (Research-Empfehlung):**
Chart-Screenshot (800x600px, letzte 60 Bars) + strukturierter Text-Prompt fuer S/R-Bestaetigung. Erst nach Text-basierter News-Integration (hoeheres Alpha-Potenzial) und S/R-Engine-Optimierung.

**Warum umsetzen?**
Vielversprechend aber noch nicht Production-Ready. Keine kontrollierten Studien belegen konsistente Ueberlegenheit gegenueber Text-basierten Indikatoren. Richtig eingestuft als "spaetere Version".

**Quellen:** Research-LLM (Kap. 6)

---

## Thema 22: Adaptive Gewichtung via Bandit-Algorithmen (Thompson Sampling)

**Prioritaet:** P3 | **Aufwand:** XL

**Was ist das?**
Statt statischer Regime-Gewichte (Thema 12): Thompson Sampling fuer adaptive Signal-Gewichtung. Pro Signalquelle eine Beta-Prior-Verteilung, Bayesian Update nach jedem Trade-Outcome. Implizite Exploration-Exploitation-Balance.

**IST-Zustand in ODIN:**
Statische Fusionsregeln im Arbiter. Kein Performance-basiertes Feedback-Signal fuer Gewichtsanpassung.

**SOLL-Zustand (Research-Empfehlung):**
Beta(alpha, beta)-Verteilungen pro Signalquelle pro Regime (5 Regimes x 2 Quellen = 10 Verteilungen). Reward-Signal: Trade-Profitabilitaet nach N Bars. Regime-spezifische Gewichtung adaptiert sich automatisch.

**Warum umsetzen?**
Theoretisch der eleganteste Ansatz fuer adaptive Fusion. Aber: Erfordert (a) klare Reward-Definition, (b) ausreichend Daten (hunderte Trades), (c) Thema 8 + Thema 12 als Vorstufen. Langfristige Perspektive.

**Quellen:** Research-Hybrid (Empfehlung 5, Thompson Sampling)

**Abhaengigkeit:** Erfordert Thema 8 (Counterfactual) + Thema 12 (Regime-Gewichtung) als Vorstufen.

---

## Zusammenfassung: Empfohlene Umsetzungsreihenfolge

### Phase 1 -- Fundament (P0)
1. **Intraday-Seasonality-Normalisierung** -- stabilisiert alle Gates sofort
2. **Backtesting-Robustheit** -- macht alle weiteren Optimierungen erst sinnvoll
3. **TCA / Implementation Shortfall** -- zeigt ob Signale nach Kosten funktionieren

### Phase 2 -- Kernverbesserungen (P1)
4. **HMM Regime-Erkennung** -- groesster einzelner Gap zum State of the Art
5. **Prompt-Caching + CoT-Forcing** -- Quick Wins, sofortiger ROI
6. **LLM-Confidence-Kalibrierung** -- macht Arbiter-Schwellen empirisch fundiert
7. **News-Integration** -- groesste fehlende Informationsquelle fuer den LLM
8. **Post-Trade Counterfactual** -- Grundlage fuer alle weiteren Optimierungen
9. **Pre-Trade Hard Controls** -- Operational Safety
10. **OFI als Microstructure-Gate** -- macht Gate-System execution-aware

### Phase 3 -- Verfeinerung (P2/P3)
11-22. Regime-Gewichtung, VPOC, Korrelation, Fallback, Vision, Bandits etc.

**Hinweis:** Die Phasen sind nicht streng sequentiell. Innerhalb einer Phase koennen Themen parallel bearbeitet werden. Abhaengigkeiten sind bei den einzelnen Themen vermerkt (z.B. Thema 18 muss VOR Thema 5).
