# ODIN-001: Subregime-Modell im API-Modul

**Modul:** odin-api
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** keine (Foundation Story)

---

## Context

Das Konzept definiert 20 Subregimes (5 Regime x 4 Subregimes pro Regime). Die aktuelle Codebase kennt nur das 5-Regime-Enum (`Regime.java`). Subregimes sind notwendig fuer differenzierte Entry/Exit-Regeln, LLM-Kontext und Profit-Protection-Profile.

## Scope

**In Scope:**
- Neues Enum `Subregime` in `de.its.odin.api.model` mit allen 20 Werten gemaess Konzept 02, Abschnitt 2
- Erweiterung von `IndicatorResult` um `Subregime subregime` Feld
- Erweiterung von `RulesResult` (odin-brain) um `Subregime subregime` -- aber das Feld wird im API-Modul als DTO-Feld definiert
- Kein Breaking Change: Bestehende `Regime`-Nutzung bleibt erhalten, `Subregime` ist additiv

**Out of Scope:**
- Subregime-Resolver-Logik (kommt in ODIN-010)
- Aenderungen an odin-brain oder anderen Fachmodulen
- Persistierung von Subregime-Daten

## Acceptance Criteria

- [ ] Enum `Subregime` existiert mit allen 20 Werten: TREND_UP_STRONG, TREND_UP_MODERATE, TREND_UP_EARLY, TREND_UP_LATE, TREND_DOWN_STRONG, TREND_DOWN_MODERATE, TREND_DOWN_EARLY, TREND_DOWN_LATE, RANGE_BOUND_NARROW, RANGE_BOUND_WIDE, RANGE_BOUND_UPPER, RANGE_BOUND_LOWER, HIGH_VOLATILITY_BREAKOUT, HIGH_VOLATILITY_EXPANSION, HIGH_VOLATILITY_SQUEEZE, HIGH_VOLATILITY_REVERSAL, UNCERTAIN_NO_SIGNAL, UNCERTAIN_CONFLICTING, UNCERTAIN_TRANSITIONAL, UNCERTAIN_LOW_VOLUME
- [ ] Jedes Subregime hat eine Methode `parentRegime()` die das zugehoerige `Regime` zurueckgibt
- [ ] `IndicatorResult` enthaelt optionales `Subregime` Feld (nullable bis Resolver implementiert)
- [ ] Bestehende Tests kompilieren und laufen

## Technical Details

**Datei:** `odin-api/src/main/java/de/its/odin/api/model/Subregime.java`

Das Enum kodiert die Parent-Zuordnung direkt:
```java
public enum Subregime {
    TREND_UP_STRONG(Regime.TREND_UP),
    TREND_UP_MODERATE(Regime.TREND_UP),
    // ... etc
    ;
    private final Regime parentRegime;
    Subregime(Regime parent) { this.parentRegime = parent; }
    public Regime parentRegime() { return parentRegime; }
}
```

## Concept References

- `docs/concept/02-regime-detection.md` -- Abschnitt 2 "Subregimes"
- `docs/concept/02-regime-detection.md` -- Abschnitt 2, Tabelle "20 Subregimes"

## Guardrail References

- `docs/backend/guardrails/module-structure.md` -- Abschnitt "odin-api: Packages"
- `docs/backend/guardrails/module-structure.md` -- "ENUM statt String fuer endliche Mengen"
- `docs/backend/guardrails/development-process.md` -- "Neue Geschaeftslogik -> Unit-Test PFLICHT"
- `T:\codebase\its_odin\CLAUDE.md` -- Coding-Regeln (R1-R13), insbesondere "Keine Abhaengigkeit in odin-api ausser JDK + Validation-API"

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api`)
- [ ] Kein `var` -- explizite Typen
- [ ] Keine Magic Numbers -- `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Keine Abhaengigkeit ausser JDK + Validation-API (odin-api Regel)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer alle 20 `parentRegime()`-Zuordnungen
- [ ] Test: Jeder Regime hat genau 4 zugehoerige Subregimes
- [ ] Test: Enum-Vollstaendigkeit (alle 20 Werte vorhanden)
- [ ] Testklassen-Namenskonvention: `SubregimeTest` (Surefire)
- [ ] Bestehende Tests kompilieren und laufen weiterhin

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstest: `IndicatorResult` mit gesetztem `Subregime`-Feld wird korrekt gebaut und zurueckgegeben
- [ ] Integrationstest: Vollstaendiger Durchlauf -- Subregime wird korrekt durch `parentRegime()` einem Regime zugeordnet
- [ ] Testklassen-Namenskonvention: `SubregimeIntegrationTest` (Failsafe)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- odin-api enthaelt keine Datenbankzugriffe.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `Subregime.java`, `IndicatorResult`-Erweiterung, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt (z.B. null-Subregime in IndicatorResult, fehlendes Regime fuer neues Subregime)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Null-Safety, fehlende Enum-Abdeckung, Ressource-Leaks, Performance-Probleme"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/02-regime-detection.md` Abschnitt 2 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob alle 20 Subregimes korrekt implementiert sind und die Parent-Zuordnung dem Konzept entspricht"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Serialisierung, Null-Handling, Backward-Compatibility bei kuenftigem Hinzufuegen von Subregimes?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` existiert im Story-Verzeichnis
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Nullable vs Optional fuer `Subregime` in `IndicatorResult`)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt
- [ ] Gemini-Review-Abschnitt ausgefuellt

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notes for Implementer

- Die 20 Subregime-Bezeichnungen muessen exakt dem Konzept entsprechen (Tabelle in 02-regime-detection.md, Abschnitt 2)
- `IndicatorResult` ist ein Record -- Erweiterung erfordert neuen Parameter am Ende (backward-compatible fuer Builder-Pattern, aber alle Aufrufer muessen angepasst werden)
- Pruefen ob `MarketSnapshot` ebenfalls ein Subregime-Feld braucht (nein -- Regime/Subregime wird vom Brain bestimmt, nicht von Data)
- Die `protocol.md` muss WAEHREND der Arbeit live aktualisiert werden -- nicht erst am Ende
