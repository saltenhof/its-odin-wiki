# ODIN — User-Story-Spezifikation

**Stand:** 2026-03-01
**Geltungsbereich:** Alle ODIN User Stories (Backend, Frontend, Infrastruktur)
**Komplementaerdokumente:**

| Dokument | Pfad | Inhalt |
|----------|------|--------|
| Playbook | `docs/meta/playbook.md` | Execution-Zyklus, Rollen, Orchestrator-Checklisten |
| CLAUDE.md | `T:\codebase\its_odin\CLAUDE.md` | Coding-Regeln, Architektur, Guardrails |
| GitHub Project Setup | `docs/meta/github-project-setup.md` | Projekt-IDs, Field-IDs, CLI-Referenz |

---

## 1. Story-Management: GitHub Project + Repo-Dateien

ODIN verwendet einen **Hybrid-Ansatz**: GitHub Issues und GitHub Project fuer Planung und Tracking, Markdown-Dateien im Repo fuer Arbeitsdokumente der Agents.

### 1.1 Steuerungsebenen

| Ebene | Werkzeug | Was lebt dort |
|-------|----------|---------------|
| **Planung & Tracking** | GitHub Project "ODIN" (`users/saltenhof/projects/2`) | Status, Priorisierung, Custom Fields, Gesamtueberblick |
| **Story-Definition** | GitHub Issue (im jeweiligen Repo) | Pflichtfelder, Akzeptanzkriterien, DoD-Checkliste, Diskussion |
| **Arbeitsdokumente** | `_concept/_userstories/ODIN-<NR>_<title>/` im Repo | `protocol.md`, `qa-report-r<N>.md` |

### 1.2 Verknuepfte Repos

| Repo | GitHub | Inhalt |
|------|--------|--------|
| Backend | `saltenhof/its-odin-backend` | Java-Backend (Maven multi-module, 10 Module) |
| Frontend | `saltenhof/its-odin-ui` | React-Frontend (Dashboard, Monitoring, Controls) |
| Wiki | `saltenhof/its-odin-wiki` | Architektur-Wiki (MkDocs Material) |

Issues werden im **jeweiligen Repo** erstellt (Backend-Stories in `its-odin-backend`, Frontend-Stories in `its-odin-ui`) und dem **uebergreifenden Project "ODIN"** zugeordnet.

### 1.3 GitHub CLI Voraussetzung

Alle `gh`-Befehle erfordern:

```bash
export GH_CONFIG_DIR="/c/Users/Sir Freejack/AppData/Roaming/GitHub CLI"
```

---

## 2. Story-Erstellung: GitHub Issue

Jede User Story wird als **GitHub Issue** erstellt. Das Issue IST die Story-Definition — es gibt KEINE separate `story.md`-Datei.

### 2.1 Issue-Erstellung per CLI

```bash
export GH_CONFIG_DIR="/c/Users/Sir Freejack/AppData/Roaming/GitHub CLI"

gh issue create --repo saltenhof/<REPO> \
  --title "<Titel der Story>" \
  --label "<labels>" \
  --body "$(cat <<'EOF'
[Issue-Body mit allen Pflichtfeldern — siehe Abschnitt 2.2]
EOF
)"
```

Nach Erstellung: Issue dem Project zuordnen und Custom Fields setzen:

```bash
# Issue dem Project zuordnen
gh project item-add 2 --owner saltenhof --url <ISSUE-URL>
```

### 2.2 Pflichtfelder im Issue-Body

Der Issue-Body MUSS folgende Struktur haben. Alle Abschnitte sind Pflicht:

```markdown
## Kontext
[Warum diese Story existiert. Fachliches Ziel in 2-4 Saetzen.
Was ist der Business-Value? Welches Problem wird geloest?]

## Modul
[Betroffenes Maven-Modul oder Frontend-Bereich.
Beispiele: odin-api, odin-data, odin-brain, odin-execution, odin-core,
odin-audit, odin-persistence, odin-app, odin-frontend]

## Abhaengigkeiten
[GitHub Issue-Nummern die VOR dieser Story abgeschlossen sein muessen.
Format: #42, #43 — oder "Keine" wenn unabhaengig.
Cross-Repo: saltenhof/its-odin-backend#42]

## Scope

### In Scope
- [Konkret was umgesetzt wird — Punkt 1]
- [Konkret was umgesetzt wird — Punkt 2]

### Out of Scope
- [Was explizit NICHT Teil dieser Story ist]

## Akzeptanzkriterien
- [ ] [Konkretes, testbares Kriterium 1]
- [ ] [Konkretes, testbares Kriterium 2]
- [ ] [Konkretes, testbares Kriterium 3]

## Technische Details
[Klassen, Patterns, Konfiguration, API-Aenderungen.
KONKRET, nicht generisch. Ein Agent muss daraus implementieren koennen.]

## Konzept-Referenzen
[Exakte Verweise: Datei + Abschnitt + ggf. Tabelle/Absatz.
Beispiel: `docs/concept/03-strategy-logic.md` — Kap. 4.2: Gate-Kaskade]

## Guardrail-Referenzen
[Verweise auf einzuhaltende Regeln aus CLAUDE.md, Wiki oder Konzept-Dokumenten.]

## Notizen fuer den Implementierer
[Fallstricke, Komplexitaeten, bekannte Probleme, Hinweise.
Was wuerde ein erfahrener Entwickler dem Implementierer mitgeben?]

## Definition of Done
- [ ] Code kompiliert fehlerfrei
- [ ] Alle Akzeptanzkriterien erfuellt
- [ ] Unit-Tests (Surefire: `*Test`)
- [ ] Integrationstests (Failsafe: `*IntegrationTest`)
- [ ] DB-Tests falls zutreffend
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Gemini-Review 3 Dimensionen (Code, Konzepttreue, Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Protokolldatei (`protocol.md`) vollstaendig
- [ ] Commit & Push mit aussagekraeftiger Message
```

### 2.3 Labels

Folgende Labels muessen in jedem Repo angelegt sein und beim Issue-Erstellen verwendet werden:

| Label | Zweck |
|-------|-------|
| `story` | Standard-User-Story |
| `bug` | Bugfix-Story |
| `refactoring` | Refactoring ohne Feature-Aenderung |
| `infrastructure` | Build, CI/CD, Tooling |
| `blocked` | Story ist blockiert (mit Kommentar warum) |
| `escalated` | Nach 3x QS-FAIL eskaliert |

### 2.4 GitHub Project Custom Fields

Nach Erstellung des Issues muessen die Project-Felder gesetzt werden:

| Feld | Wann setzen | Wer setzt |
|------|-------------|-----------|
| **Size** | Bei Story-Erstellung | Story-Ersteller |
| **Module** | Bei Story-Erstellung | Story-Ersteller |
| **Epic** | Bei Story-Erstellung | Story-Ersteller |
| **QA Rounds** | Nach Story-Abschluss | Orchestrator |
| **Gemini Calls** | Nach Story-Abschluss | Orchestrator |
| **ChatGPT Calls** | Nach Story-Abschluss | Orchestrator |
| **Processing Time (min)** | Nach Story-Abschluss | Orchestrator |

### 2.5 GitHub Project IDs (ODIN-Instanz)

| Entitaet | ID |
|----------|----|
| Project "ODIN" | `PVT_kwHODdCDrM4BQeRU` |
| Project Number | `2` |
| Owner | `saltenhof` |
| URL | `https://github.com/users/saltenhof/projects/2` |

#### Custom Field IDs

| Feld | Field-ID | Typ |
|------|----------|-----|
| Status | `PVTSSF_lAHODdCDrM4BQeRUzg-lPVg` | SINGLE_SELECT |
| Size | `PVTSSF_lAHODdCDrM4BQeRUzg-lP0Q` | SINGLE_SELECT |
| Gemini Calls | `PVTF_lAHODdCDrM4BQeRUzg-lQV4` | NUMBER |
| ChatGPT Calls | `PVTF_lAHODdCDrM4BQeRUzg-lQV8` | NUMBER |
| Processing Time (min) | `PVTF_lAHODdCDrM4BQeRUzg-lQWA` | NUMBER |
| QA Rounds | `PVTF_lAHODdCDrM4BQeRUzg-lQWE` | NUMBER |
| Module | `PVTF_lAHODdCDrM4BQeRUzg-lQWw` | TEXT |
| Epic | `PVTF_lAHODdCDrM4BQeRUzg-lQW0` | TEXT |

#### Status-Optionen

| Status | Option-ID |
|--------|-----------|
| Todo | `f75ad846` |
| In Progress | `47fc9ee4` |
| Done | `98236657` |

#### Size-Optionen

| Size | Option-ID |
|------|-----------|
| XS | `7b09b9cd` |
| S | `44c290b6` |
| M | `c56e42b7` |
| L | `1cb0b07a` |
| XL | `8344ad05` |
| XXL | `d5590be9` |

---

## 3. Arbeitsdokumente im Repo

Waehrend der Implementierung werden Arbeitsdokumente im Repo angelegt. Diese Dateien werden von Worker- und QS-Agents gelesen und geschrieben.

### 3.1 Verzeichniskonvention

```
_concept/_userstories/
  ├── playbook.md                              ← Prozess-Definition
  ├── user-story-specification.md              ← Story-Spezifikation (dieses Dokument)
  └── ODIN-<NR>_<short-title>/                ← Pro Story ein Verzeichnis
        ├── protocol.md                        ← Worker schreibt live
        ├── qa-report-r1.md                    ← QS-Agent Runde 1
        ├── qa-report-r2.md                    ← QS-Agent Runde 2 (falls noetig)
        └── qa-report-r3.md                    ← QS-Agent Runde 3 (falls noetig)
```

- **NR:** GitHub Issue Nummer (z.B. `ODIN-42`)
- **Short-Title:** Kebab-case, sprechend (z.B. `brain-gate-cascade`, `api-subregime-model`)
- Das Verzeichnis wird vom **Worker-Agent** bei Arbeitsbeginn angelegt

### 3.2 Protokolldatei (`protocol.md`)

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

### 3.3 QA-Report (`qa-report-r<N>.md`)

Wird vom QS-Agent geschrieben. Struktur:

```markdown
# QA-Report: ODIN-<NR> — [Titel]
## Runde: <N>
## Ergebnis: PASS | FAIL

## Pruefprotokoll

### 5.1 Code-Qualitaet
- [x] / [ ] [Kriterium] — [Befund]

### 5.2 Unit-Tests
- [x] / [ ] [Kriterium] — [Befund]

### 5.3 Integrationstests
- [x] / [ ] [Kriterium] — [Befund]

### 5.4 DB-Tests
- [x] / [ ] [Kriterium] — [Befund] (oder "nicht zutreffend")

### 5.5 ChatGPT-Sparring
- [x] / [ ] [Kriterium] — [Befund]

### 5.6 Gemini-Review
- [x] / [ ] [Kriterium] — [Befund]

### 5.7 Protokolldatei
- [x] / [ ] [Kriterium] — [Befund]

### 5.8 Abschluss
- [x] / [ ] [Kriterium] — [Befund]

## Findings (nur bei FAIL)

| # | Schwere | Bereich | Finding | Erwartung |
|---|---------|---------|---------|-----------|
| 1 | CRITICAL/MAJOR/MINOR | DoD-Abschnitt | Was ist falsch | Was erwartet wird |
| 2 | ... | ... | ... | ... |

## Zusammenfassung
[1-3 Saetze: Gesamtbewertung, ggf. wiederkehrende Muster]
```

---

## 4. Umfangs-Kategorien (Size)

| Groesse | Klassen | Beschreibung |
|---------|---------|-------------|
| **XS** | 1 | Konfigaenderung, Bugfix, Enum-Erweiterung |
| **S** | 1-3 | Einfacher Service, DTO, wenig Logik |
| **M** | 3-8 | Service mit Geschaeftslogik, Konfiguration, mehrere Klassen |
| **L** | 8-15 | Komplexe Komponente mit vielen Interaktionen |
| **XL** | >15 | Moduluebergreifend oder fundamentaler Umbau |
| **XXL** | >25 | Epic-Level — MUSS in Sub-Stories gesplittet werden |

**Zielverteilung:** Die meisten Stories sollten **S bis M** sein. L ist akzeptabel fuer komplexe Fachlogik. XL/XXL werden immer aufgeteilt.

---

## 5. Definition of Done (DoD) — Vollstaendig

Die DoD gilt fuer JEDE Story und ist nicht verhandelbar. Alle Punkte muessen abgehakt sein, bevor eine Story als abgeschlossen gilt. Vollstaendige Beschreibung aller Punkte im Playbook Abschnitt 5.

### 5.1 Code-Qualitaet

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

### 5.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer alle neuen Klassen mit Geschaeftslogik
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces (keine Spring-Kontexte noetig fuer Pipeline-Komponenten)
- [ ] Bugfix → reproduzierender Test PFLICHT
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 5.3 Tests — Komponentenebene (Integrationstests)

Klassenebene-Tests allein sind NICHT ausreichend. Es muessen zusaetzlich Integrationstests existieren, die **mehrere reale Implementierungen zusammenschalten** und ganze Strecken End-to-End durchtesten.

- [ ] Integrationstests mit realen Klassen (nicht alles weggemockt)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest pro Story, der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass die Klassen nicht nur isoliert, sondern auch im Zusammenspiel korrekt funktionieren.

### 5.4 Tests — Datenbank (falls zutreffend)

Gilt nur fuer Stories, die Datenbankzugriff haben (Repositories, Migrations, Queries).

- [ ] Embedded-Postgres-Tests mit **Zonky** (`io.zonky.test:embedded-postgres`)
- [ ] Flyway-Migrationen werden im Test ausgefuehrt
- [ ] Repository-Methoden werden gegen echte DB getestet (nicht nur Mocks)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)

**Referenz:** CLAUDE.md → Tests (`*IntegrationTest` fuer Failsafe)

### 5.5 Test-Sparring mit ChatGPT

Bevor die Testphase als abgeschlossen gilt, MUSS der Umsetzer sich von ChatGPT explizit **Grenzfaelle und zusaetzliche Testszenarien** als Sparring holen.

- [ ] ChatGPT-Session gestartet mit: Klasse(n), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 5.6 Review durch Gemini — Drei Dimensionen

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
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

**Haltung gegenueber Review-Feedback:** Gemini ist ein gleichgestellter Review-Partner, kein Vorgesetzter. Feedback kritisch bewerten. Dinge duerfen verworfen werden, wenn fachlich begruendet. Iterativ arbeiten — eine zweite Review-Runde bringt oft noch gute Verbesserungen.

### 5.7 Abschluss

**Durch den Worker-Agent:**

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `protocol.md` (+ ggf. `qa-report-r<N>.md`)

**Durch den Orchestrator (NICHT delegierbar — siehe Playbook Abschnitt 2.4):**

- [ ] Per-Story Telemetrie-Datei lesen (`_temp/story-telemetry/<STORY-ID>.jsonl`) und aggregieren: Processing Time, ChatGPT Calls, Gemini Calls
- [ ] GitHub Project Metriken setzen: QA Rounds, Gemini Calls, ChatGPT Calls, Processing Time (min)
- [ ] GitHub Issue geschlossen (`gh issue close <NR> --repo <REPO> --reason completed`)
- [ ] GitHub Project Status auf "Done" gesetzt
- [ ] Story ist ERST nach diesem Schritt abgeschlossen

---

## 6. Konzept- und Guardrail-Referenzen

### 6.1 Referenzen muessen KONKRET sein

**Nicht akzeptabel:**
> "Siehe Konzeptdokument"

**Akzeptabel:**
> "`docs/concept/03-strategy-logic.md` — Abschnitt 4.2: Gate-Kaskade (7 Hard Gates), Tabelle 'Gates mit Schwellen'"

Der implementierende Agent muss auf einen Blick wissen, **wo genau** er nachschauen muss.

### 6.2 Relevante Guardrails

Jede Story verweist auf die Guardrails, die fuer die Umsetzung relevant sind:

| Guardrail | Wiki-Pfad | Inhalt |
|-----------|-----------|--------|
| Modulstruktur | `docs/backend/guardrails/module-structure.md` | Packages, Namensregeln, Abhaengigkeiten |
| Entwicklungsprozess | `docs/backend/guardrails/development-process.md` | Phasen, Checkpoints, Review-Pflicht |
| Frontend | `docs/frontend/guardrails/frontend.md` | Feature-Struktur, TypeScript, State Management |
| CLAUDE.md | `T:\codebase\its_odin\CLAUDE.md` | Coding-Regeln, Tech-Stack, Konventionen |

---

## 7. Abhaengigkeiten

### 7.1 Abhaengigkeitsregeln

- Eine Story darf nur gestartet werden, wenn **alle Abhaengigkeiten** (verlinkte Issues) Status "Done" haben
- Stories ohne Abhaengigkeiten koennen **parallel** bearbeitet werden
- Abhaengigkeiten werden im Issue-Body als `#<NR>` referenziert und im Abschnitt "Abhaengigkeiten" aufgelistet

### 7.2 Cross-Repo-Abhaengigkeiten

Wenn eine Frontend-Story von einer Backend-Story abhaengt:

```markdown
## Abhaengigkeiten
saltenhof/its-odin-backend#42 — REST-Endpoint muss existieren
```

Vollqualifizierte Referenz mit `owner/repo#nr`.

---

## 8. Epics

Ein **Epic** ist eine thematische Klammer ueber mehrere Stories. Epics sind KEINE Issues — sie werden ausschliesslich ueber das Custom Field `Epic` im GitHub Project abgebildet.

Beispiele fuer Epics:
- `Brain Pipeline` — Stories rund um Gate-Kaskade, Scoring, Signallogik
- `Data Infrastructure` — MarketData, DataProvider, Caching
- `Execution` — Order-Management, Broker-Integration, Kill-Switch
- `Monitoring & UI` — Dashboard, SSE-Streams, Controls

Alle Stories eines Epics haben denselben Wert im `Epic`-Feld. Filterung und Gruppierung erfolgt ueber Project-Views.

---

## 9. Agent-Execution-Regeln (verbindlich)

Die folgenden Regeln gelten fuer die Ausfuehrung von User Stories durch Claude-Sub-Agents. Sie sind das Ergebnis empirischer Erkenntnisse und NICHT verhandelbar.

### 9.1 Eine Story = Ein Agent (Kontexthygiene)

**IMMER einen neuen, dedizierten Agent pro User Story starten.** Niemals mehrere Stories in einem Agent buendeln.

**Begruendung:** Mit jeder Aktion waechst der Agent-Kontext. Ab einem Schwellenwert findet **Context-Compaction** statt — der Agent verliert Teile seiner Instruktionen. Wenn ein Agent mehrere Stories sequenziell abarbeitet, weiss er bei der dritten Story nur noch die Haelfte von dem, was er tun sollte. Die DoD-Compliance sinkt mit jeder weiteren Story drastisch.

**Konsequenz:**
- Der Orchestrator (Primary Claude) startet pro Story einen frischen Agent
- Jeder Agent erhaelt: die Issue-URL (der Agent liest das Issue via `gh issue view`), die User-Story-Spezifikation, CLAUDE.md, und die relevanten Konzeptdateien
- Jeder Agent arbeitet EINE Story vollstaendig ab (Code → Tests → ChatGPT-Sparring → Gemini-Review → Protokoll → Commit)
- Nach Abschluss meldet der Agent zurueck und wird beendet

### 9.2 Fail-Fast bei blockierten DoD-Schritten

Wenn ein Agent einen Pflichtschritt der DoD nicht ausfuehren kann (z.B. ChatGPT/Gemini nicht verfuegbar, Kompilierungsfehler nicht loesbar, Dependency nicht vorhanden), dann:

1. **Sofort abbrechen** — nicht den Schritt ueberspringen und weitermachen
2. **Fehlermeldung zurueckgeben** mit: welcher Schritt blockiert ist, warum, was fehlt
3. **Keine Partial-Completion** — eine Story ist entweder vollstaendig oder gar nicht abgeschlossen

### 9.3 QS-Gate nach jedem Agent

Nach Abschluss eines Implementierungs-Agents wird ein **dedizierter QS-Agent pro Story** gestartet, der prueft:

- Alle DoD-Checkboxen gegen den tatsaechlichen Code und die Protokolldatei
- Kompilierung und Testausfuehrung
- Konzepttreue (Vergleich Implementierung vs. Konzeptdokument)
- Vollstaendigkeit der Protokolldatei

Bei NICHT BESTANDEN geht die Story in die Nacharbeit (neuer Agent mit den QS-Findings als Input).

### 9.4 Agent-Prompt-Struktur (Pflichtbestandteile)

Jeder Implementierungs-Agent erhaelt im Prompt:

1. **Issue-URL:** `gh issue view <NR> --repo saltenhof/<REPO>` (der Agent liest das Issue selbst)
2. **User-Story-Spezifikation:** `docs/meta/user-story-specification.md` (DoD-Referenz)
3. **CLAUDE.md:** `T:\codebase\its_odin\CLAUDE.md` (Coding-Regeln)
4. **Konzeptdateien:** Pfade zu den in der Story referenzierten Konzeptdokumenten
5. **Guardrails:** Pfade zu den relevanten Guardrail-Dokumenten
6. **Explizite Anweisung:** "Lies die User-Story-Spezifikation und befolge ALLE DoD-Punkte 5.1 bis 5.7. Ueberspringe KEINEN Schritt. Bei Blockern: abbrechen und Fehler melden."

### 9.5 Parallelisierung

- Stories OHNE Abhaengigkeiten koennen parallel durch separate Agents umgesetzt werden
- Jeder parallele Agent arbeitet auf einem **eigenen Modul** (kein Git-Konflikt)
- Bei Stories im gleichen Modul: sequenziell oder per Git-Worktree isolieren
- **ChatGPT-Pool:** Max. 4 parallele Slots — bei mehr als 4 Agents staggern
- **Gemini-Pool:** Max. 4 parallele Slots — analoge Regelung

---

## 10. Batch-Erstellung von Stories

Wenn mehrere Stories auf einmal erstellt werden (z.B. aus einem Konzept-Dokument), gilt:

### 10.1 Ablauf

1. Konzeptdokument analysieren und Stories identifizieren
2. Abhaengigkeiten zwischen den Stories bestimmen
3. Stories in Erstellungsreihenfolge bringen (Abhaengigkeiten zuerst)
4. Issues sequenziell erstellen (damit `#NR`-Referenzen stimmen)
5. Alle Issues dem Project zuordnen (`gh project item-add 2 --owner saltenhof --url <URL>`)
6. Custom Fields setzen (Size, Module, Epic)
7. Zusammenfassung an User: Tabelle mit Issue-NR, Titel, Size, Abhaengigkeiten

### 10.2 Ausgabe-Format nach Batch-Erstellung

```
| Issue | Titel | Size | Module | Epic | Abhaengigkeiten |
|-------|-------|------|--------|------|-----------------|
| #1    | ...   | M    | ...    | ...  | Keine           |
| #2    | ...   | S    | ...    | ...  | #1              |
| #3    | ...   | L    | ...    | ...  | #1, #2          |
```

---

## 11. Story-Groessen-Leitprinzip

> **So klein wie moeglich, so gross wie noetig.**

- Eine Story = ein in sich abgeschlossenes, testbares Feature
- Nicht zu gross: "Implementiere die halbe Pipeline" ist keine Story
- Nicht zu klein: "Erstelle einen Getter" ist keine Story
- Gute Granularitaet: Ein Service, eine Komponente, oder eine zusammenhaengende Gruppe von Klassen die nur zusammen Sinn ergeben
- XXL wird IMMER in Sub-Stories gesplittet
- Im Zweifel lieber zwei Stories als eine XL-Story

---

## 12. Checkliste fuer Story-Ersteller

Vor dem Veroeffentlichen einer Story pruefen:

### Inhaltlich
- [ ] Alle Pflichtfelder im Issue-Body ausgefuellt (Abschnitt 2.2)
- [ ] Akzeptanzkriterien sind **testbar** (nicht "soll gut funktionieren")
- [ ] Konzept-Referenzen sind **exakt** (Datei + Abschnitt + Punkt)
- [ ] Guardrail-Referenzen sind angegeben
- [ ] Abhaengigkeiten sind korrekt und vollstaendig
- [ ] Scope ist klar abgegrenzt (In Scope / Out of Scope)
- [ ] Technische Details sind konkret genug fuer einen unabhaengigen Implementierer
- [ ] DoD-Checkliste ist vollstaendig im Issue-Body
- [ ] Umfangsschaetzung (Size) ist realistisch

### GitHub-Integration
- [ ] Issue im richtigen Repo erstellt (`its-odin-backend`, `its-odin-ui` oder `its-odin-wiki`)
- [ ] Issue dem Project "ODIN" zugeordnet (`gh project item-add 2 --owner saltenhof`)
- [ ] Custom Fields gesetzt: Size, Module, Epic
- [ ] Labels gesetzt (mindestens `story`, `bug`, `refactoring` oder `infrastructure`)
- [ ] Abhaengigkeiten als Issue-Referenzen (`#NR` oder `owner/repo#NR`)
