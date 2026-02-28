# ODIN-048 [PHASE2]: FusionLab Evaluation Framework

**Modul:** odin-backtest, odin-brain
**Phase:** 2 -- Nicht fuer V1 vorgesehen. Umsetzung erst nach explizitem Stakeholder-Beschluss.
**Groesse:** XL
**Abhaengigkeiten:** ODIN-035 (Governance Variants)

---

## Kontext

Das FusionLab ist ein isoliertes Evaluations-Framework das die Regime-Fusion empirisch optimiert. 4 Varianten (F0-F3) werden gegeneinander getestet: MC-Regel, Reliability EOD, Reliability Decision-Level, Reliability + Pooling. Ground-Truth kommt aus 30-Min-ex-post-Labeling.

## Scope

**In Scope:**
- FusionLab als eigenstaendiger Evaluations-Modus im Backtest
- 4 Fusion-Varianten (F0-F3) mit Loss-Cost-Matrix
- Ground-Truth-Labeling: 30-Min-Lookforward, ATR-basiert
- Datenbank-Schema fuer FusionLab-Ergebnisse
- Statistische Vergleichsmethodik (paired t-test, confidence intervals)

**Out of Scope:**
- Live-Einsatz der FusionLab-Ergebnisse (nur Evaluation)
- Automatische Varianten-Selektion (manueller Beschluss)
- Mehr als 4 Varianten in V2 (F0-F3 ist vollstaendiger Satz)

## Akzeptanzkriterien

- [ ] FusionLab-Modus aktivierbar ueber Konfiguration (nicht das Standard-Backtest-Verhalten aendern)
- [ ] Variante F0 (MC-Regel): Implementiert und ausfuehrbar
- [ ] Variante F1 (Reliability EOD): Implementiert und ausfuehrbar
- [ ] Variante F2 (Reliability Decision-Level): Implementiert und ausfuehrbar
- [ ] Variante F3 (Reliability + Pooling): Implementiert und ausfuehrbar
- [ ] Ground-Truth-Labeling: 30-Min-Lookahead, ATR-basierte Schwelle, korrekt implementiert
- [ ] Loss-Cost-Matrix: Konfigurierbar (FP-Kosten vs. FN-Kosten)
- [ ] Statistische Auswertung: Paired t-test + Confidence Intervals fuer Varianten-Vergleich
- [ ] Ergebnisse werden in Datenbank gespeichert (FusionLab-spezifische Tabellen)
- [ ] Ergebnis-Report ausgegeben (JSON oder CSV)

## Technische Details

**Betroffene Klassen (Schaetzung):**
- `odin-backtest` (neues Modul oder Unterpaket): FusionLabRunner, FusionVariant (F0-F3), GroundTruthLabeler
- `odin-brain`: Anbindung der bestehenden Regime-Fusion-Logik
- `odin-persistence`: Neue Flyway-Migration fuer FusionLab-Ergebnis-Tabellen
- Konfiguration: `odin.fusionlab.{property}`

## Konzept-Referenzen

- `docs/concept/02-regime-detection.md` -- Abschnitt 5 "FusionLab"
- `docs/concept/02-regime-detection.md` -- Abschnitt 5.1-5.4 "Varianten F0-F3"
- `docs/concept/09-backtesting-evaluation.md` -- Abschnitt 1 "FusionLab (F0-F3)"

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` -- Modulabhaengigkeiten, Package-Konventionen
- `docs/backend/guardrails/development-process.md` -- Phasen, Checkpoints
- `T:/codebase/its_odin/CLAUDE.md` -- Coding-Regeln, Architektur

## Definition of Done

> **Hinweis Phase 2:** Diese Story wird erst umgesetzt nach explizitem Stakeholder-Beschluss. Die nachfolgende DoD gilt dann vollstaendig und ohne Ausnahme.

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- Konstanten fuer ATR-Schwellen, Lookahead-Perioden, Loss-Cost-Defaults
- [ ] Records fuer DTOs (FusionLabResult, GroundTruthLabel)
- [ ] ENUM fuer FusionVariant (F0, F1, F2, F3)
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc)
- [ ] FusionLab-Modus isoliert -- kein Einfluss auf Standard-Backtest und Live-Betrieb
- [ ] Namespace: `odin.fusionlab.{property}`

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer GroundTruthLabeler (30-Min-Lookahead, ATR-Berechnung, korrekte Label)
- [ ] Unit-Tests fuer Loss-Cost-Matrix-Kalkulation
- [ ] Unit-Tests fuer jeden Fusion-Varianten-Algorithmus (F0-F3) mit synthetischen Daten
- [ ] Unit-Tests fuer Paired t-test-Implementierung (mit bekanntem Ergebnis verifiziert)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Klassen zusammengeschaltet
- [ ] FusionLabRunner-Integration: Vollstaendiger Run mit synthetischen historischen Daten, alle 4 Varianten
- [ ] Ergebnis-Persistenz-Integration: FusionLab-Ergebnisse werden korrekt gespeichert und gelesen
- [ ] Mindestens 1 Integrationstest der alle 4 Varianten vergleicht und statistische Auswertung prueft

### 2.4 Tests -- Datenbank

- [ ] Embedded-Postgres-Tests mit Zonky fuer FusionLab-Ergebnis-Tabellen
- [ ] Flyway-Migrationen werden im Test ausgefuehrt
- [ ] Repository-Methoden fuer FusionLab-Ergebnisse gegen echte Embedded-DB getestet
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Implementierungscode, Konzept-Referenzen, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. zu wenig Datenpunkte fuer statistisch signifikanten Test, alle Varianten liefern gleiche Ergebnisse, Lookahead-Bias in Ground-Truth-Labeling)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben: "Pruefe auf Bugs, Lookahead-Bias in Ground-Truth-Labeling, statistische Korrektheit des Paired t-test, Off-by-One-Fehler bei 30-Min-Lookahead"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + Konzept-Referenzen an Gemini
- [ ] Auftrag: "Pruefe ob Implementierung dem Konzept entspricht -- insb. F0-F3-Algorithmen korrekt implementiert, Loss-Cost-Matrix gemaess Konzept, Ground-Truth-Labeling gemaess Abschnitt 5"
- [ ] Abweichungen bewertet

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es Themen die im Konzept nicht behandelt sind? Z.B. Daten-Menge die fuer statistisch signifikante Ergebnisse noetig ist, Overfitting-Risiko bei Varianten-Selektion, wie Ergebnisse in Live-System uebertragen werden"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State live aktualisiert waehrend der Arbeit
- [ ] Design-Entscheidungen dokumentiert
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- XL-Story: Erwaegen ob Sub-Stories sinnvoll sind (z.B. ODIN-048a: GroundTruthLabeler, ODIN-048b: F0-F1-Varianten, ODIN-048c: F2-F3-Varianten, ODIN-048d: Statistik + Persistenz)
- Lookahead-Bias ist das groesste Risiko: Ground-Truth darf nur Daten verwenden die in der Realitaet 30 Minuten NACH dem Beobachtungszeitpunkt liegen
- Statistische Tests (paired t-test) korrekt implementieren -- bestehende Bibliothek verwenden (z.B. Apache Commons Math)
- FusionLab ist ein Evaluations-Tool, kein Teil des Live-Trading-Pfads -- Isolation sicherstellen
