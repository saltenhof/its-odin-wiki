# User-Story-Spezifikation

**Stand:** 2026-02-21
**Geltungsbereich:** Alle ODIN User Stories (Backend, Frontend, Infrastruktur)

Dieses Dokument definiert verbindlich, was eine User Story enthalten muss und welche Anforderungen im Rahmen der Umsetzung zu erfuellen sind. Jeder Umsetzer (Mensch oder Agent) MUSS diese Spezifikation einhalten.

---

## 1. Story-Struktur

Jede User Story wird als eigenes Verzeichnis mit einer `story.md`-Datei angelegt.

### Verzeichniskonvention

```
<story-root>/
├── ODIN-NNN_<short-title>/
│   ├── story.md        ← Story-Definition (wird VOR Umsetzung geschrieben)
│   └── protocol.md     ← Protokolldatei (wird WAEHREND Umsetzung geschrieben)
├── ODIN-NNN-PHASE2_<short-title>/
│   ├── story.md
│   └── protocol.md
```

- **Story-ID:** Fortlaufend, Format `ODIN-NNN`
- **Short-Title:** Kebab-case, sprechend (z.B. `brain-gate-cascade`, `api-subregime-model`)
- **Phase-2-Stories:** Erhalten `PHASE2` im Verzeichnisnamen fuer schnelle Filterung per `ls`/`find`

### Pflichtfelder in `story.md`

| Feld | Beschreibung |
|------|-------------|
| **Titel** | Kurzbezeichnung der Story |
| **Modul** | Betroffenes Maven-Modul (odin-api, odin-data, odin-brain, odin-execution, odin-core, odin-audit, odin-persistence, odin-app, odin-frontend) |
| **Phase** | `1` (V1-Scope) oder `2` (zurueckgestellt) |
| **Abhaengigkeiten** | Story-IDs, die VOR dieser Story abgeschlossen sein muessen. `Keine` wenn unabhaengig |
| **Geschaetzter Umfang** | `S` / `M` / `L` / `XL` (siehe Abschnitt 1.1) |
| **Kontext** | Warum diese Story existiert, fachliches Ziel (2-4 Saetze) |
| **Scope** | In Scope + Out of Scope (was explizit NICHT Teil der Story ist) |
| **Akzeptanzkriterien** | Konkrete, testbare Kriterien (Checkliste) |
| **Technische Details** | Klassen, Patterns, Konfiguration. KONKRET, nicht generisch |
| **Konzept-Referenzen** | Exakte Verweise: Datei + Abschnitt + ggf. Tabelle/Absatz |
| **Guardrail-Referenzen** | Exakte Verweise auf einzuhaltende Guardrails |
| **Definition of Done** | Vollstaendige DoD-Checkliste (siehe Abschnitt 2) |
| **Notizen fuer den Implementierer** | Fallstricke, Komplexitaeten, Hinweise |

### 1.1 Umfangs-Kategorien

| Groesse | Klassen | Typischer Aufwand | Beschreibung |
|---------|---------|-------------------|-------------|
| **S** (Small) | 1-3 | < 1 Tag | Enum, DTO, einfacher Service. Wenig Logik |
| **M** (Medium) | 3-8 | 1-2 Tage | Service mit Geschaeftslogik, Konfiguration, mehrere Klassen |
| **L** (Large) | 8-15 | 2-4 Tage | Komplexe Komponente mit vielen Interaktionen |
| **XL** (Extra Large) | >15 | >4 Tage | Moduluebergreifend oder fundamentaler Umbau. Sollte ggf. in Sub-Stories gesplittet werden |

**Zielverteilung:** Die meisten Stories sollten **M** sein. Wenige S (zu granular) und wenige L/XL (zu gross).

---

## 2. Definition of Done (DoD)

Die DoD gilt fuer JEDE Story und ist nicht verhandelbar. Alle Punkte muessen abgehakt sein, bevor eine Story als abgeschlossen gilt.

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl <modul>`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer alle neuen Klassen mit Geschaeftslogik
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces (keine Spring-Kontexte noetig fuer Pro-Pipeline-Komponenten)
- [ ] Bugfix → reproduzierender Test PFLICHT
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

Klassenebene-Tests allein sind NICHT ausreichend. Es muessen zusaetzlich Integrationstests existieren, die **mehrere reale Implementierungen zusammenschalten** und ganze Strecken End-to-End durchtesten.

- [ ] Integrationstests mit realen Klassen (nicht alles weggemockt)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Beispiel: Service-Methode End-to-End mit echtem Repository-Stub, echtem Validator, echtem Mapper
- [ ] Mindestens 1 Integrationstest pro Story, der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass die Klassen nicht nur isoliert, sondern auch im Zusammenspiel korrekt funktionieren.

### 2.4 Tests — Datenbank (falls zutreffend)

Gilt nur fuer Stories, die Datenbankzugriff haben (Repositories, Migrations, Queries).

- [ ] Embedded-Postgres-Tests mit **Zonky** (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migrationen werden im Test ausgefuehrt
- [ ] Repository-Methoden werden gegen echte DB getestet (nicht nur Mocks)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

**Referenz:** CLAUDE.md → Tests (`*IntegrationTest` fuer Failsafe)

### 2.5 Test-Sparring mit ChatGPT

Bevor die Testphase als abgeschlossen gilt, MUSS der Umsetzer sich von ChatGPT explizit **Grenzfaelle und zusaetzliche Testszenarien** als Sparring holen.

- [ ] ChatGPT-Session gestartet mit: Klasse(n), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

Jede fertig implementierte Komponente MUSS von Gemini reviewed werden. Das Review hat **drei explizite Dimensionen:**

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Race Conditions, Null-Safety, Ressource-Leaks, Performance-Probleme"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + referenzierte Konzeptdokumente an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht. Vergleiche Akzeptanzkriterien, Enum-Werte, Schwellenwerte, Verhalten bei Edge Cases"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Welche realen Szenarien koennten auftreten, die wir uebersehen haben?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtg loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

**Haltung gegenueber Review-Feedback:** Gemini ist ein gleichgestellter Review-Partner, kein Vorgesetzter. Feedback kritisch bewerten. Dinge duerfen verworfen werden, wenn fachlich begruendet. Iterativ arbeiten — eine zweite Review-Runde bringt oft noch gute Verbesserungen.

### 2.7 Protokolldatei (`protocol.md`)

Jede Story hat eine Protokolldatei, die **waehrend der Arbeit live aktualisiert** wird — nicht erst am Ende.

#### Pflichtabschnitte:

```markdown
# Protokoll: ODIN-NNN — [Titel]

## Working State
[Aktueller Stand. Wird bei jedem Meilenstein aktualisiert.]
- [ ] Initiale Implementierung
- [ ] Unit-Tests geschrieben
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Integrationstests geschrieben
- [ ] Gemini-Review Dimension 1 (Code)
- [ ] Gemini-Review Dimension 2 (Konzepttreue)
- [ ] Gemini-Review Dimension 3 (Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Commit & Push

## Design-Entscheidungen
[Entscheidungen, die waehrend der Implementierung getroffen wurden.
Warum so und nicht anders.]

## Offene Punkte
[Fragen, Unklarheiten, Themen die an den Stakeholder eskaliert werden muessen.]

## ChatGPT-Sparring
[Zusammenfassung: Welche Testszenarien vorgeschlagen, welche umgesetzt/verworfen.]

## Gemini-Review
[Zusammenfassung der drei Review-Dimensionen. Findings + Bewertung.]
```

#### Aktualisierungszeitpunkte (Pflicht):

| Meilenstein | Was ins Protokoll |
|-------------|------------------|
| Nach initialer Implementierung | Working State, Design-Entscheidungen |
| Nach ChatGPT-Test-Sparring | ChatGPT-Sparring-Abschnitt |
| Nach Integrationstests | Working State aktualisieren |
| Nach Gemini-Review | Gemini-Review-Abschnitt, ggf. Offene Punkte |
| Nach Review-Einarbeitung | Working State, ggf. Design-Entscheidungen |
| Bei Abschluss | Finaler Working State |

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## 3. Konzept- und Guardrail-Referenzen

### Konzept-Referenzen muessen KONKRET sein

Nicht akzeptabel:
> "Siehe Konzeptdokument"

Akzeptabel:
> "`docs/concept/03-strategy-logic.md` — Abschnitt 4.2: Gate-Kaskade (7 Hard Gates), Tabelle 'Gates mit Schwellen'"

Der implementierende Agent muss auf einen Blick wissen, **wo genau** er nachschauen muss.

### Relevante Guardrails

Jede Story verweist auf die Guardrails, die fuer die Umsetzung relevant sind:

| Guardrail | Wiki-Pfad | Inhalt |
|-----------|-----------|--------|
| Modulstruktur | `docs/backend/guardrails/module-structure.md` | Packages, Namensregeln, Abhaengigkeiten |
| Entwicklungsprozess | `docs/backend/guardrails/development-process.md` | Phasen, Checkpoints, Review-Pflicht |
| Frontend | `docs/frontend/guardrails/frontend.md` | Feature-Struktur, TypeScript, State Management |
| CLAUDE.md | `T:\codebase\its_odin\CLAUDE.md` | Coding-Regeln, Tech-Stack, Konventionen |

---

## 4. Abhaengigkeiten und Parallelisierung

### Abhaengigkeitsregeln

- Eine Story darf nur gestartet werden, wenn **alle Abhaengigkeiten abgeschlossen** sind
- Stories ohne Abhaengigkeiten koennen **parallel** bearbeitet werden
- Der Execution Plan gruppiert Stories in **Waves** — innerhalb einer Wave ist Parallelisierung moeglich

### Execution Plan

Fuer jedes Story-Set wird ein `EXECUTION-PLAN.md` erstellt mit:

1. **Story-Inventar nach Modul** (Tabelle)
2. **Abhaengigkeitsgraph** (ASCII oder Mermaid)
3. **Execution Waves** (Gruppen parallel ausfuehrbarer Stories)
4. **Kritischer Pfad** (laengste Abhaengigkeitskette)
5. **Risiko-Assessment** (HIGH/MEDIUM/LOW pro Story)

---

## 5. Phase-2-Stories

Stories fuer spaetere Entwicklungsstufen werden miterfasst aber explizit als Phase 2 geflaggt:

- Verzeichnisname enthaelt `PHASE2`: `ODIN-NNN-PHASE2_<title>/`
- Feld `Phase: 2` in der story.md
- Keine Umsetzung bis expliziter Stakeholder-Beschluss
- Dienen als Platzhalter und Planungshilfe

### Typische Phase-2-Kriterien

- Explizit als "spaetere Entwicklungsstufe" oder "Zielbild" im Konzept markiert
- Features fuer Maerkte ausserhalb des V1-Scope (z.B. EU-Maerkte)
- Fortgeschrittene Varianten mit hoher Komplexitaet (z.B. FusionLab, L2-Depth)
- Nice-to-have-Features ohne direkten Trading-Impact

---

## 6. Story-Groessen-Leitprinzip

> **So klein wie moeglich, so gross wie noetig.**

- Eine Story = ein in sich abgeschlossenes, testbares Feature
- Nicht zu gross: "Implementiere die halbe Datenpipeline" ist keine Story
- Nicht zu klein: "Erstelle einen Getter" ist keine Story
- Gute Granularitaet: Ein Service, eine Komponente, oder eine zusammenhaengende Gruppe von Klassen die nur zusammen Sinn ergeben
- Im Zweifel lieber zwei Stories als eine XL-Story

---

## 7. Agent-Execution-Regeln (verbindlich)

Die folgenden Regeln gelten fuer die Ausfuehrung von User Stories durch Claude-Sub-Agents. Sie sind das Ergebnis empirischer Erkenntnisse aus der ersten Implementierungswelle und NICHT verhandelbar.

### 7.1 Eine Story = Ein Agent (Kontexthygiene)

**IMMER einen neuen, dedizierten Agent pro User Story starten.** Niemals mehrere Stories in einem Agent buendeln.

**Begruendung:** Mit jeder Aktion waechst der Agent-Kontext. Ab einem Schwellenwert findet **Context-Compaction** statt — der Agent verliert Teile seiner Instruktionen. Wenn ein Agent mehrere Stories sequenziell abarbeitet, weiss er bei der dritten Story nur noch die Haelfte von dem, was er tun sollte. Die DoD-Compliance sinkt mit jeder weiteren Story drastisch.

**Konsequenz:**
- Der Orchestrator (Primary Claude) startet pro Story einen frischen Agent
- Jeder Agent erhaelt: die story.md, die User-Story-Spezifikation, CLAUDE.md, und die relevanten Konzeptdateien
- Jeder Agent arbeitet EINE Story vollstaendig ab (Code → Tests → ChatGPT-Sparring → Gemini-Review → Protokoll → Commit)
- Nach Abschluss meldet der Agent zurueck und wird beendet

### 7.2 Fail-Fast bei blockierten DoD-Schritten

Wenn ein Agent einen Pflichtschritt der DoD nicht ausfuehren kann (z.B. ChatGPT/Gemini nicht verfuegbar, Kompilierungsfehler nicht loesbar, Dependency nicht vorhanden), dann:

1. **Sofort abbrechen** — nicht den Schritt ueberspringen und weitermachen
2. **Fehlermeldung zurueckgeben** mit: welcher Schritt blockiert ist, warum, was fehlt
3. **Keine Partial-Completion** — eine Story ist entweder vollstaendig oder gar nicht abgeschlossen

**Begruendung:** "Fast alles gemacht, nur ChatGPT hat nicht funktioniert" fuehrt zu systematischen Luecken die spaeter teuer nachgearbeitet werden muessen. Lieber frueher scheitern und den Orchestrator informieren.

### 7.3 QS-Gate nach jedem Agent

Nach Abschluss eines Implementierungs-Agents wird ein **dedizierter QS-Agent pro Story** gestartet, der prueft:

- Alle DoD-Checkboxen gegen den tatsaechlichen Code und die Protokolldatei
- Kompilierung und Testausfuehrung
- Konzepttreue (Vergleich Implementierung vs. Konzeptdokument)
- Vollstaendigkeit der Protokolldatei

Bei NICHT BESTANDEN geht die Story in die Nacharbeit (neuer Agent mit den QS-Findings als Input).

### 7.4 Agent-Prompt-Struktur (Pflichtbestandteile)

Jeder Implementierungs-Agent erhaelt im Prompt:

1. **Story-Datei:** Pfad zur `story.md` (der Agent liest sie selbst)
2. **User-Story-Spezifikation:** `docs/meta/user-story-specification.md` (DoD-Referenz)
3. **CLAUDE.md:** `T:\codebase\its_odin\CLAUDE.md` (Coding-Regeln)
4. **Konzeptdateien:** Pfade zu den in der Story referenzierten Konzeptdokumenten
5. **Guardrails:** Pfade zu den relevanten Guardrail-Dokumenten
6. **Explizite Anweisung:** "Lies die User-Story-Spezifikation und befolge ALLE DoD-Punkte 2.1 bis 2.8. Ueberspringe KEINEN Schritt. Bei Blockern: abbrechen und Fehler melden."

### 7.5 Parallelisierung

- Stories OHNE Abhaengigkeiten koennen parallel durch separate Agents umgesetzt werden
- Jeder parallele Agent arbeitet auf einem **eigenen Modul** (kein Git-Konflikt)
- Bei Stories im gleichen Modul: sequenziell oder per Git-Worktree isolieren
- **ChatGPT-Pool:** Max. 4 parallele Slots — bei mehr als 4 Agents staggern
- **Gemini-Pool:** Max. 4 parallele Slots — analoge Regelung

---

## 8. Checkliste fuer Story-Ersteller

Vor dem Veroeffentlichen einer Story pruefen (gilt auch fuer den Orchestrator-Prompt an den Agent):

- [ ] Alle Pflichtfelder ausgefuellt
- [ ] Akzeptanzkriterien sind **testbar** (nicht "soll gut funktionieren")
- [ ] Konzept-Referenzen sind **exakt** (Datei + Abschnitt + Punkt)
- [ ] Guardrail-Referenzen sind angegeben
- [ ] Abhaengigkeiten sind korrekt und vollstaendig
- [ ] Scope ist klar abgegrenzt (In Scope / Out of Scope)
- [ ] Technische Details sind konkret genug fuer einen unabhaengigen Implementierer
- [ ] DoD ist vollstaendig (aus dieser Spezifikation uebernommen)
- [ ] Umfangsschaetzung ist realistisch
