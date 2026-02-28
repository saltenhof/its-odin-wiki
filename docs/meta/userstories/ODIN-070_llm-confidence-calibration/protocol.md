# Protokoll: ODIN-070 -- LLM Confidence Calibration

## Working State
- [x] Initiale Implementierung
- [x] Unit-Tests geschrieben (60 Tests, alle gruen)
- [x] ChatGPT-Sparring fuer Edge-Cases und Architektur-Review
- [x] Integrationstests geschrieben (CalibrationEndToEndIntegrationTest)
- [x] Gemini-Review Dimension 1 (Code)
- [x] Gemini-Review Dimension 2 (Konzepttreue)
- [x] Gemini-Review Dimension 3 (Praxis)
- [x] Review-Findings eingearbeitet
- [x] Full test suite: 305 Tests, 0 Failures
- [x] Commit & Push

## Design-Entscheidungen

1. **Pre-Aggregation vor PAVA**: Datenpunkte mit gleichem rawConfidence werden vor dem PAVA-Algorithmus zu ihrem empirischen Mittel aggregiert. Ohne dies produziert PAVA fehlerhafte Support-Punkte bei vielen identischen x-Werten.

2. **Lineare Interpolation statt Step-Function**: Standard-PAVA erzeugt piecewise-constant Funktionen. Wir verwenden die PAVA-Support-Punkte mit linearer Interpolation fuer glattere Kalibrierungskurven -- ein gaengiger praktischer Variant fuer Confidence-Kalibrierung.

3. **Separates calibratedConfidence-Feld in LlmVote**: Raw Confidence bleibt erhalten (Audit-Trail), kalibrierte Confidence wird nur fuer Threshold-Entscheidungen verwendet. Nullable Double, damit null = nicht kalibriert.

4. **Fail-Open Design**: Wenn Modell nicht verfuegbar oder Input NaN, wird raw Confidence unveraendert verwendet. ChatGPT-Review empfiehlt alternativ Fail-Conservative (Cap bei 0.6) -- als Future Enhancement notiert.

5. **Feature Flag Default Off**: `odin.brain.calibration.enabled=false` -- keine Aenderung im Produktionsverhalten ohne explizite Aktivierung.

6. **Jackson @JsonProperty-Annotations**: Explizite Annotations auf CalibrationModelDto noetig, da Jackson `getXValues()` sonst als "xvalues" (alles lowercase) serialisiert.

## Offene Punkte (Future Enhancements)

- **Model Hot-Reloading**: Gemini empfiehlt Mechanismus zum Neuladen des Modells zur Laufzeit (Scheduled Task oder FileWatcher) ohne Neustart. Aktuell: Modell wird beim Pipeline-Start einmalig geladen.
- **Fail-Conservative Fallback**: ChatGPT empfiehlt `effectiveConfidence = min(raw, 0.6)` statt raw Pass-Through wenn Kalibrierung aktiviert aber Modell fehlt.
- **Confidence-Quantisierung**: Bei kontinuierlichen LLM-Outputs koennten Confidences vor Aggregation auf 0.01-Schritte gerundet werden.
- **Walk-Forward Validation**: Brier/ECE sollten auf einem separaten Time-Split berechnet werden, nicht auf Trainingsdaten.
- **Triple-Barrier Ground Truth**: ChatGPT schlaegt als Alternative zu "3 Forward Bars" eine Triple-Barrier-Methode vor (Profit-Target, Stop-Loss, Max-Hold).

## ChatGPT-Sparring

### Bewertung
ChatGPT lieferte umfassende, hochwertige Analyse mit konkreten Code-Findings und Verbesserungsvorschlaegen.

### Wesentliche Findings (eingearbeitet)
1. **IsotonicRegressionModel-Konstruktor**: Fehlende Validierung fuer sortierte x-Werte, monotone y-Werte und finite Werte. **Fix implementiert**: Validierung fuer non-decreasing x, non-decreasing y, und Double.isFinite() auf allen Werten.
2. **ECE Binning Safety**: Negative Confidences fuehrten zu negativem Bin-Index. **Fix implementiert**: `Math.max(0, ...)` Clamping und Ueberspringen von NaN/Infinity-Datenpunkten.
3. **Piecewise-Constant Dokumentation**: JavaDoc sagte "piecewise-constant", aber Implementierung ist piecewise-linear. **Fix implementiert**: JavaDoc korrigiert.
4. **ATOMIC_MOVE Fallback**: `ATOMIC_MOVE` kann auf manchen Dateisystemen scheitern. **Fix implementiert**: AtomicMoveNotSupportedException gefangen mit Fallback auf non-atomic Move.

### Notiert (nicht eingearbeitet, da Future Enhancement)
- Fail-Conservative statt Fail-Open bei fehlendem Modell
- Confidence-Quantisierung bei kontinuierlichen LLM-Outputs
- Triple-Barrier Ground Truth als Alternative
- Walk-Forward Validation fuer Metriken

## Gemini-Review

### Dimension 1: Code Review
**Bewertung: Sehr gut.** Gemini bewertet die Implementierung als "remarkably well-engineered, production-grade". Keine kritischen Issues gefunden. Einziges Minor Issue: ArrayList.remove() in PAVA-Loop ist O(N^2), aber durch Pre-Aggregation praktisch irrelevant (N = Anzahl distinct Confidence-Werte, typisch < 100).

### Dimension 2: Konzepttreue
**Bewertung: Vollstaendig umgesetzt.** Alle Anforderungen aus ODIN-070 implementiert:
- Isotonic Regression via PAVA
- Brier Score und ECE als Monitoring-KPIs
- Feature Flag Pattern (Default disabled)
- Kalibrierte Confidence nur fuer Threshold-Entscheidungen
- Raw Confidence preserved fuer Audit Trail
- Fail-Open Design
- Ground Truth abstrahiert via CalibrationDataPoint

### Dimension 3: Praxis
**Bewertung: Gut geeignet.** Gemini bestaetigt, dass Isotonic Regression besser als Platt Scaling fuer LLMs ist (unregulare Confidence-Verteilungen). Operationelle Empfehlungen:
- Hot-Reloading-Mechanismus fuer Modellupdates (notiert als Future Enhancement)
- ECE-Binning-Sparsity bei enger LLM-Confidence-Range beachten
- 3-Bar Ground Truth funktional fuer taktische Entries, aber rauschig fuer Regime-Korrektheit
