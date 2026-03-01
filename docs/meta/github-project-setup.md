# GitHub Project Setup — Referenzdokumentation

**Stand:** 2026-03-01
**Referenzimplementierung:** GitHub Project "MOSES" (`users/saltenhof/projects/1`)
**Zweck:** Vollstaendige Anleitung zur Replikation des GitHub-Project-Setups in einem neuen Projekt

---

## 1. Ueberblick & Architekturentscheidung

### 1.1 Hybrid-Ansatz: GitHub Project + Markdown-Dateien

Das Story-Management nutzt einen Hybrid-Ansatz, weil GitHub Projects V2 allein nicht ausreicht, um den vollen Agent-Workflow abzubilden:

| Ebene | Werkzeug | Inhalt | Zugriff durch |
|-------|----------|--------|---------------|
| **Planung & Tracking** | GitHub Project (User-Level) | Status, Custom Fields, Board-Views, Metriken | Orchestrator, User |
| **Story-Definition** | GitHub Issue (im Repo) | Pflichtfelder, Akzeptanzkriterien, DoD-Checkliste | Worker-Agent, QS-Agent |
| **Arbeitsdokumente** | Markdown im Repo | `protocol.md`, `qa-report-r<N>.md` | Worker-Agent, QS-Agent |

**Warum Hybrid?**
- GitHub Issues koennen von Agents via `gh issue view` gelesen werden
- Arbeitsdokumente (Protokolle, QA-Reports) werden von Agents live geschrieben — das geht nur als Dateien im Repo
- GitHub Project Custom Fields erfassen Metriken (Dauer, Calls, QA-Runden) fuer uebergreifende Auswertung
- Ein User-Level-Project kann mehrere Repos (Backend, Frontend) ueberspannen

### 1.2 Voraussetzungen

| Voraussetzung | Beschreibung |
|---------------|-------------|
| GitHub Account | User-Account mit Projektberechtigung (nicht Organization, sondern `users/<login>`) |
| `gh` CLI | GitHub CLI installiert und authentifiziert |
| Auth-Scopes | `project`, `read:project` (zusaetzlich zu den Standard-Scopes) |
| Windows: Encoding | `chcp 65001`, `LANG=en_US.UTF-8`, `LC_ALL=en_US.UTF-8` (im Startskript) |
| Windows: GH Config | `GH_CONFIG_DIR` muss auf das Profil-Verzeichnis zeigen |

---

## 2. Datenmodell

### 2.1 GitHub Project V2 — Entitaeten

```
GitHub Project (User-Level)
├── Linked Repositories []          ← Repos die Issues liefern
├── Custom Fields []                ← Projekt-eigene Felder
├── Items []                        ← Referenzen auf Issues/PRs
│     ├── content: Issue | PR       ← Das eigentliche Issue
│     └── fieldValues []            ← Werte der Custom Fields fuer dieses Item
└── Views []                        ← Board, Table, Roadmap
```

### 2.2 Custom Fields — Datenmodell (MOSES-Referenz)

| Feld | Typ | Optionen / Format | Zweck | Gesetzt bei |
|------|-----|-------------------|-------|-------------|
| **Status** | `SINGLE_SELECT` | `Todo`, `In Progress`, `Done` | Workflow-Status | Story-Beginn/-Ende |
| **Size** | `SINGLE_SELECT` | `XS`, `S`, `M`, `L`, `XL`, `XXL` | T-Shirt-Umfangsschaetzung | Story-Erstellung |
| **Module** | `TEXT` | Freitext (Komma-getrennt) | Betroffene Module/Bereiche | Story-Erstellung |
| **Epic** | `TEXT` | Freitext | Thematische Klammer | Story-Erstellung |
| **QA Rounds** | `NUMBER` | Integer | Anzahl QS-Durchlaeufe bis PASS | Nach Abschluss |
| **Gemini Calls** | `NUMBER` | Integer | Anzahl Gemini-Aufrufe | Nach Abschluss |
| **ChatGPT Calls** | `NUMBER` | Integer | Anzahl ChatGPT-Aufrufe | Nach Abschluss |
| **Processing Time (min)** | `NUMBER` | Integer (Minuten) | Gesamtverarbeitungsdauer | Nach Abschluss |

### 2.3 Built-in Fields (von GitHub bereitgestellt)

Diese Felder existieren automatisch in jedem Project V2 und muessen NICHT erstellt werden:

| Feld | Typ | Beschreibung |
|------|-----|-------------|
| Title | `TITLE` | Issue-Titel (read-only, kommt vom Issue) |
| Assignees | `ASSIGNEES` | Zugewiesene Personen |
| Status | `SINGLE_SELECT` | Workflow-Status (Standard: Todo/In Progress/Done) |
| Labels | `LABELS` | Issue-Labels |
| Linked pull requests | `LINKED_PULL_REQUESTS` | Verlinkte PRs |
| Milestone | `MILESTONE` | Milestone |
| Repository | `REPOSITORY` | Quell-Repository |
| Reviewers | `REVIEWERS` | Reviewer |
| Parent issue | `PARENT_ISSUE` | Uebergeordnetes Issue |
| Sub-issues progress | `SUB_ISSUES_PROGRESS` | Fortschritt der Sub-Issues |

### 2.4 Field-Typ-Referenz

GitHub Projects V2 unterstuetzt folgende Custom-Field-Typen:

| Typ | GraphQL-Typ | CLI-Flag | Beschreibung |
|-----|-------------|----------|-------------|
| Text | `TEXT` | `--text "<wert>"` | Freitext |
| Number | `NUMBER` | `--number <wert>` | Numerisch (Integer/Float) |
| Date | `DATE` | `--date "YYYY-MM-DD"` | Datum |
| Single Select | `SINGLE_SELECT` | `--single-select-option-id "<id>"` | Dropdown mit definierten Optionen |
| Iteration | `ITERATION` | `--iteration-id "<id>"` | Sprint/Iteration-Feld |

### 2.5 Size-Kategorien (T-Shirt-Groessen)

| Groesse | Klassen | Beschreibung |
|---------|---------|-------------|
| **XS** | 1 | Konfigaenderung, Bugfix, Enum-Erweiterung |
| **S** | 1-3 | Einfacher Service, DTO, wenig Logik |
| **M** | 3-8 | Service mit Geschaeftslogik, Konfiguration |
| **L** | 8-15 | Komplexe Komponente mit vielen Interaktionen |
| **XL** | >15 | Moduluebergreifend oder fundamentaler Umbau |
| **XXL** | >25 | Epic-Level — MUSS in Sub-Stories gesplittet werden |

### 2.6 Issue-Body-Struktur

Jedes Issue hat einen standardisierten Body mit diesen Pflichtabschnitten:

```
## Kontext              ← Warum existiert die Story?
## Modul                ← Betroffenes Modul
## Abhaengigkeiten      ← Verlinkte Issues (#NR) oder "Keine"
## Scope
### In Scope            ← Was wird umgesetzt
### Out of Scope        ← Was wird NICHT umgesetzt
## Akzeptanzkriterien   ← Testbare Checkboxen
## Technische Details   ← Konkrete Klassen, Patterns, APIs
## Konzept-Referenzen   ← Exakte Datei+Abschnitt-Verweise
## Guardrail-Referenzen ← Einzuhaltende Regeln
## Notizen fuer den Implementierer
## Definition of Done   ← DoD-Checkliste
```

### 2.7 Verzeichnisstruktur im Repo

```
_concept/_userstories/
├── playbook.md                              ← Prozess-Definition
├── user-story-specification.md              ← Story-Struktur, DoD
├── github-project-setup.md                  ← DIESES Dokument
└── <PRAEFIX>-<NR>_<short-title>/           ← Pro Story ein Verzeichnis
      ├── protocol.md                        ← Worker schreibt live
      ├── qa-report-r1.md                    ← QS-Agent Runde 1
      ├── qa-report-r2.md                    ← QS-Agent Runde 2
      └── qa-report-r3.md                    ← QS-Agent Runde 3
```

---

## 3. IDs & Referenzen (MOSES-Instanz)

### 3.1 Projekt-IDs

| Entitaet | ID |
|----------|----|
| Project "MOSES" | `PVT_kwHODdCDrM4BQb2r` |
| Project Number | `1` |
| Owner | `saltenhof` |
| URL | `https://github.com/users/saltenhof/projects/1` |

### 3.2 Custom Field IDs

| Feld | Field-ID | Typ |
|------|----------|-----|
| Status | `PVTSSF_lAHODdCDrM4BQb2rzg-jjnA` | SINGLE_SELECT |
| Size | `PVTSSF_lAHODdCDrM4BQb2rzg-jkX0` | SINGLE_SELECT |
| Gemini Calls | `PVTF_lAHODdCDrM4BQb2rzg-jkYk` | NUMBER |
| ChatGPT Calls | `PVTF_lAHODdCDrM4BQb2rzg-jkZU` | NUMBER |
| Processing Time (min) | `PVTF_lAHODdCDrM4BQb2rzg-jkaA` | NUMBER |
| QA Rounds | `PVTF_lAHODdCDrM4BQb2rzg-jkaE` | NUMBER |
| Module | `PVTF_lAHODdCDrM4BQb2rzg-jkaI` | TEXT |
| Epic | `PVTF_lAHODdCDrM4BQb2rzg-jkbg` | TEXT |

### 3.3 Status-Optionen

| Status | Option-ID |
|--------|-----------|
| Todo | `f75ad846` |
| In Progress | `47fc9ee4` |
| Done | `98236657` |

### 3.4 Size-Optionen

| Size | Option-ID |
|------|-----------|
| XS | `149a7b60` |
| S | `ebff64ac` |
| M | `740aa107` |
| L | `26c77906` |
| XL | `da730ef8` |
| XXL | `70d194cc` |

### 3.5 Verknuepfte Repos

| Repo | GitHub-Name |
|------|-------------|
| Backend | `saltenhof/moses-backend` |
| Frontend | `saltenhof/moses-frontend` (geplant) |

---

## 4. Setup — Schritt fuer Schritt

### 4.1 Voraussetzungen pruefen

```bash
# GitHub CLI installiert?
gh --version

# Authentifiziert?
gh auth status

# Falls nicht: Login
gh auth login

# Projekt-Scopes hinzufuegen (einmalig)
gh auth refresh -s project,read:project
```

**Windows-spezifisch:**

```bash
# GH Config Directory setzen (MUSS vor jedem gh-Befehl stehen)
export GH_CONFIG_DIR="/c/Users/<USERNAME>/AppData/Roaming/GitHub CLI"
```

### 4.2 Projekt erstellen

```bash
# User-Level-Projekt (uebergreifend fuer mehrere Repos)
gh project create --owner <USER> --title "<PROJEKT-NAME>"
```

Ausgabe liefert die Projekt-URL, z.B.:
```
https://github.com/users/<USER>/projects/<NUMBER>
```

Die Projekt-Nummer (`<NUMBER>`) wird fuer alle weiteren Befehle benoetigt.

### 4.3 Repos verlinken

```bash
# Jedes Repo, das Issues liefern soll, verlinken
gh project link <NUMBER> --owner <USER> --repo <USER>/<REPO-NAME>
```

### 4.4 Custom Fields erstellen

```bash
PROJECT_NR=<NUMBER>
OWNER=<USER>

# Single-Select-Feld: Size (T-Shirt-Groessen)
gh project field-create $PROJECT_NR --owner $OWNER \
  --name "Size" --data-type "SINGLE_SELECT" \
  --single-select-options "XS,S,M,L,XL,XXL"

# Number-Felder
gh project field-create $PROJECT_NR --owner $OWNER \
  --name "Gemini Calls" --data-type "NUMBER"

gh project field-create $PROJECT_NR --owner $OWNER \
  --name "ChatGPT Calls" --data-type "NUMBER"

gh project field-create $PROJECT_NR --owner $OWNER \
  --name "Processing Time (min)" --data-type "NUMBER"

gh project field-create $PROJECT_NR --owner $OWNER \
  --name "QA Rounds" --data-type "NUMBER"

# Text-Felder
gh project field-create $PROJECT_NR --owner $OWNER \
  --name "Module" --data-type "TEXT"

gh project field-create $PROJECT_NR --owner $OWNER \
  --name "Epic" --data-type "TEXT"
```

### 4.5 Field-IDs und Option-IDs abfragen

Nach dem Erstellen der Felder MUESSEN die generierten IDs abgefragt und dokumentiert werden — sie werden fuer alle weiteren API-Aufrufe benoetigt.

```bash
gh api graphql -f query='{
  user(login: "<USER>") {
    projectV2(number: <NUMBER>) {
      id
      fields(first: 20) {
        nodes {
          ... on ProjectV2SingleSelectField {
            id name dataType
            options { id name }
          }
          ... on ProjectV2Field {
            id name dataType
          }
        }
      }
    }
  }
}'
```

Die Ausgabe enthaelt fuer jedes Feld die `id` und fuer Single-Select-Felder zusaetzlich die `options` mit ihren `id`s. Diese IDs muessen in der Projektdokumentation festgehalten werden (siehe Abschnitt 3).

---

## 5. Operationen — CLI-Referenz

### 5.1 Issue erstellen und dem Project zuordnen

```bash
export GH_CONFIG_DIR="/c/Users/<USERNAME>/AppData/Roaming/GitHub CLI"

# 1. Issue erstellen
ISSUE_URL=$(gh issue create --repo <USER>/<REPO> \
  --title "<PRAEFIX>-<NR>: <Titel>" \
  --body "<BODY>" \
  --label "story")

# 2. Dem Project zuordnen
gh project item-add <PROJECT-NR> --owner <USER> --url "$ISSUE_URL"
```

### 5.2 Item-ID im Project ermitteln

Jedes Issue hat im Project eine eigene `Item-ID`, die fuer Feld-Updates benoetigt wird.

```bash
# Alle Items mit ihren IDs und Issue-Nummern abfragen
gh api graphql -f query='{
  user(login: "<USER>") {
    projectV2(number: <NUMBER>) {
      items(first: 100) {
        nodes {
          id
          content {
            ... on Issue { number title }
          }
        }
      }
    }
  }
}'
```

### 5.3 Custom Fields setzen — CLI (`gh project item-edit`)

```bash
PROJECT_ID="<PROJECT-NODE-ID>"  # z.B. PVT_kwH...
ITEM_ID="<ITEM-NODE-ID>"       # z.B. PVTI_lAH...

# Text-Feld setzen
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "<FIELD-ID>" --text "<Wert>"

# Number-Feld setzen
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "<FIELD-ID>" --number <WERT>

# Single-Select-Feld setzen
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "<FIELD-ID>" --single-select-option-id "<OPTION-ID>"
```

### 5.4 Custom Fields setzen — GraphQL (Batch / programmatisch)

Fuer Batch-Operationen ist die GraphQL-API effizienter, weil mehrere Felder in einem einzigen API-Call gesetzt werden koennen:

```graphql
mutation {
  field1: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID>"
    fieldId: "<FIELD-ID>"
    value: { singleSelectOptionId: "<OPTION-ID>" }
  }) { projectV2Item { id } }

  field2: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID>"
    fieldId: "<FIELD-ID>"
    value: { text: "<WERT>" }
  }) { projectV2Item { id } }

  field3: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID>"
    fieldId: "<FIELD-ID>"
    value: { number: 42 }
  }) { projectV2Item { id } }
}
```

**WICHTIG — Windows-Sonderzeichen-Problem:**

GraphQL-Queries mit `$`-Variablen (z.B. `query($proj: ID!)`) funktionieren auf Windows in Git Bash NICHT zuverlaessig. Der `$`-Zeichen wird korrumpiert, unabhaengig von Quoting (Single/Double Quotes).

**Loesung: Hardcoded Values + Datei-basiert**

1. GraphQL-Mutation als `.graphql`-Datei schreiben (mit hartkodierten IDs statt Variablen)
2. Datei via `@`-Syntax an `gh` uebergeben:

```bash
# Query in Datei schreiben (Write-Tool oder echo)
# WICHTIG: Datei muss auf einem Windows-sichtbaren Pfad liegen,
# NICHT unter /tmp (wird von gh nicht gefunden)

gh api graphql -F query=@/pfad/zur/mutation.graphql
```

Fuer Alias-basiertes Batching koennen bis zu ~50 Mutations in einer einzigen Query zusammengefasst werden (siehe Beispiel in Abschnitt 7.2).

### 5.5 Status-Workflow

```bash
# Status auf "In Progress" setzen
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "<STATUS-FIELD-ID>" \
  --single-select-option-id "<IN-PROGRESS-OPTION-ID>"

# Status auf "Done" setzen
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
  --field-id "<STATUS-FIELD-ID>" \
  --single-select-option-id "<DONE-OPTION-ID>"
```

### 5.6 Issue lesen (fuer Agents)

```bash
# Issue-Inhalt lesen (Body, Titel, Labels, Status)
gh issue view <NR> --repo <USER>/<REPO>

# Nur Body als Markdown
gh issue view <NR> --repo <USER>/<REPO> --json body --jq '.body'
```

### 5.7 Issue schliessen

```bash
# Als completed schliessen
gh issue close <NR> --repo <USER>/<REPO> --reason completed

# Kommentar hinterlassen
gh issue comment <NR> --repo <USER>/<REPO> --body "<TEXT>"
```

### 5.8 Project-Board abfragen

```bash
# Alle Items auflisten
gh project item-list <PROJECT-NR> --owner <USER> --format json

# Projekt-Uebersicht
gh project view <PROJECT-NR> --owner <USER>
```

---

## 6. Bekannte Probleme & Workarounds

### 6.1 Windows: `$`-Zeichen in GraphQL-Queries

**Problem:** Git Bash (MSYS2) auf Windows korrumpiert `$`-Zeichen in Strings, die an `gh api graphql` uebergeben werden. Auch Single-Quoted Strings (`'...'`) schuetzen nicht zuverlaessig.

**Fehlerbild:** `gh: Expected VAR_SIGN, actual: UNKNOWN_CHAR ("")`

**Workaround:**
- GraphQL-Queries OHNE Variablen schreiben (alle Werte hardcoded inline)
- Query als Datei speichern und via `gh api graphql -F query=@datei.graphql` uebergeben
- Datei MUSS auf einem Windows-nativen Pfad liegen (z.B. `T:/...`), NICHT unter `/tmp`

### 6.2 Windows: `grep -P` nicht verfuegbar

**Problem:** `grep -P` (Perl-Regex) funktioniert in Git Bash nicht mit der Standard-Locale.

**Workaround:** `sed` statt `grep -P` verwenden. Oder Encoding-Variablen setzen:
```bash
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

### 6.3 Windows: `jq` nicht vorinstalliert

**Problem:** `jq` ist auf Windows standardmaessig nicht verfuegbar.

**Workaround:** `gh`'s eingebaute `--jq`-Flag verwenden:
```bash
# Statt: gh api ... | jq '.data.field'
# Besser:
gh api ... --jq '.data.field'
```

### 6.4 Windows: `/tmp`-Pfad-Mapping

**Problem:** `/tmp` in Git Bash wird auf ein MSYS2-internes Temp-Verzeichnis gemappt, das `gh.exe` (natives Windows-Binary) nicht finden kann.

**Workaround:** Temp-Dateien immer auf Windows-nativen Pfaden ablegen:
```bash
# NICHT: /tmp/query.graphql
# SONDERN: T:/codebase/projekt/_temp_query.graphql
```

### 6.5 Heredoc-Probleme in Batch-Skripten

**Problem:** Heredocs mit `<<'EOF'` koennen in bestimmten Windows-Shell-Kontexten fehlschlagen.

**Workaround:** Inhalte per Write-Tool (Claude Code) als Datei schreiben, dann via Bash referenzieren. Oder `echo` mit korrektem Escaping verwenden.

### 6.6 Sub-Agents und Bash-Permissions

**Problem:** Claude Code Sub-Agents muessen fuer jeden `gh`-CLI-Aufruf eine Bash-Permission bestaetigen lassen — bei vielen Aufrufen unpraktisch.

**Workaround:** Alle `gh`-Operationen in einem einzigen Bash-Skript buendeln. Ein Skript = eine Permission-Bestaetigung.

---

## 7. Rezepte

### 7.1 Batch-Erstellung von Issues mit Custom Fields

Vollstaendiges Skript fuer die Erstellung mehrerer Issues mit allen Feldern:

```bash
#!/bin/bash
export GH_CONFIG_DIR="/c/Users/<USERNAME>/AppData/Roaming/GitHub CLI"

OWNER="<USER>"
REPO="<USER>/<REPO>"
PROJECT_NR=<NUMBER>

# === Issues erstellen ===

ISSUE_1=$(gh issue create --repo "$REPO" \
  --title "<PRAEFIX>-001: Erster Titel" \
  --body "$(cat <<'EOF'
## Kontext
...
## Modul
...
[restlicher Body]
EOF
)" | sed 's|.*/||')

ISSUE_2=$(gh issue create --repo "$REPO" \
  --title "<PRAEFIX>-002: Zweiter Titel" \
  --body "..." | sed 's|.*/||')

echo "Erstellt: #$ISSUE_1, #$ISSUE_2"

# === Dem Project zuordnen ===

for NR in $ISSUE_1 $ISSUE_2; do
  URL="https://github.com/$REPO/issues/$NR"
  gh project item-add $PROJECT_NR --owner $OWNER --url "$URL"
done

# === Custom Fields setzen (via GraphQL-Datei) ===
# Siehe Abschnitt 7.2 fuer die GraphQL-Datei-Methode
```

### 7.2 Batch-Update von Custom Fields via GraphQL-Datei

Fuer das Setzen von Custom Fields auf mehreren Issues gleichzeitig:

**Schritt 1:** Item-IDs abfragen (siehe 5.2)

**Schritt 2:** GraphQL-Mutation als Datei schreiben (z.B. `_temp_fields.graphql`):

```graphql
mutation {
  i1_size: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID-1>"
    fieldId: "<SIZE-FIELD-ID>"
    value: { singleSelectOptionId: "<M-OPTION-ID>" }
  }) { projectV2Item { id } }

  i1_module: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID-1>"
    fieldId: "<MODULE-FIELD-ID>"
    value: { text: "modul-name" }
  }) { projectV2Item { id } }

  i1_epic: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID-1>"
    fieldId: "<EPIC-FIELD-ID>"
    value: { text: "Epic-Name" }
  }) { projectV2Item { id } }

  i2_size: updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID-2>"
    fieldId: "<SIZE-FIELD-ID>"
    value: { singleSelectOptionId: "<S-OPTION-ID>" }
  }) { projectV2Item { id } }

  # ... weitere Felder ...
}
```

**Schritt 3:** Ausfuehren:

```bash
gh api graphql -F query=@<PFAD>/_temp_fields.graphql
rm <PFAD>/_temp_fields.graphql
```

**Vorteile:**
- Ein einziger API-Call statt N*3 Einzelaufrufe
- Kein `$`-Encoding-Problem (keine GraphQL-Variablen)
- Atomare Ausfuehrung (alle Felder werden zusammen gesetzt)
- Bis zu ~50 Mutations pro Query moeglich

### 7.3 Neues Projekt aufsetzen (Komplett-Checkliste)

```
1. [ ] gh auth login (falls noetig)
2. [ ] gh auth refresh -s project,read:project
3. [ ] gh project create --owner <USER> --title "<PROJEKT>"
4. [ ] Projekt-Nummer notieren
5. [ ] gh project link <NR> --owner <USER> --repo <USER>/<REPO>
6. [ ] Custom Fields erstellen (Abschnitt 4.4)
7. [ ] Field-IDs und Option-IDs abfragen und dokumentieren (Abschnitt 4.5)
8. [ ] IDs in die Projekt-Dokumentation eintragen (analog Abschnitt 3)
9. [ ] Verzeichnisstruktur im Repo anlegen:
       _concept/_userstories/
       ├── playbook.md
       ├── user-story-specification.md
       └── github-project-setup.md
10. [ ] Startskript mit UTF-8-Encoding-Variablen anpassen (Abschnitt 4.1)
11. [ ] Labels im Repo erstellen: story, bug, refactoring, infrastructure, blocked, escalated
12. [ ] Erstes Test-Issue erstellen und Workflow verifizieren
```

### 7.4 Labels im Repo erstellen

```bash
export GH_CONFIG_DIR="/c/Users/<USERNAME>/AppData/Roaming/GitHub CLI"
REPO="<USER>/<REPO>"

gh label create "story"          --repo "$REPO" --color "0075ca" --description "Standard User Story"
gh label create "bug"            --repo "$REPO" --color "d73a4a" --description "Bugfix"
gh label create "refactoring"    --repo "$REPO" --color "a2eeef" --description "Refactoring ohne Feature"
gh label create "infrastructure" --repo "$REPO" --color "d4c5f9" --description "Build, CI/CD, Tooling"
gh label create "blocked"        --repo "$REPO" --color "e4e669" --description "Story ist blockiert"
gh label create "escalated"      --repo "$REPO" --color "b60205" --description "Nach 3x QS-FAIL eskaliert"
```

---

## 8. GraphQL API — Kurzreferenz

### 8.1 Projekt-Schema introspizieren

```graphql
{
  user(login: "<USER>") {
    projectV2(number: <NR>) {
      id
      title
      url
      fields(first: 30) {
        nodes {
          ... on ProjectV2SingleSelectField {
            id name dataType
            options { id name }
          }
          ... on ProjectV2Field {
            id name dataType
          }
          ... on ProjectV2IterationField {
            id name dataType
          }
        }
      }
    }
  }
}
```

### 8.2 Items mit Field-Values lesen

```graphql
{
  user(login: "<USER>") {
    projectV2(number: <NR>) {
      items(first: 100) {
        nodes {
          id
          content {
            ... on Issue {
              number
              title
              state
              repository { name }
            }
          }
          fieldValues(first: 20) {
            nodes {
              ... on ProjectV2ItemFieldTextValue {
                field { ... on ProjectV2Field { name } }
                text
              }
              ... on ProjectV2ItemFieldNumberValue {
                field { ... on ProjectV2Field { name } }
                number
              }
              ... on ProjectV2ItemFieldSingleSelectValue {
                field { ... on ProjectV2SingleSelectField { name } }
                name
              }
            }
          }
        }
      }
    }
  }
}
```

### 8.3 Field-Value setzen (Mutation)

```graphql
mutation {
  updateProjectV2ItemFieldValue(input: {
    projectId: "<PROJECT-ID>"
    itemId: "<ITEM-ID>"
    fieldId: "<FIELD-ID>"
    value: { text: "wert" }          # oder: number: 42
                                      # oder: singleSelectOptionId: "<ID>"
                                      # oder: date: "2026-03-01"
  }) {
    projectV2Item { id }
  }
}
```

### 8.4 Item dem Project hinzufuegen (Mutation)

```graphql
mutation {
  addProjectV2ItemById(input: {
    projectId: "<PROJECT-ID>"
    contentId: "<ISSUE-NODE-ID>"
  }) {
    item { id }
  }
}
```

---

## 9. Glossar

| Begriff | Bedeutung |
|---------|-----------|
| **Project V2** | GitHub's aktuelles Projektmanagement-System (ersetzt "classic" Projects) |
| **Item** | Ein Eintrag im Project — Referenz auf ein Issue oder einen PR |
| **Item-ID** | Eindeutige Node-ID des Items im Project (z.B. `PVTI_lAH...`) |
| **Field-ID** | Eindeutige Node-ID eines Custom Fields (z.B. `PVTF_lAH...` oder `PVTSSF_lAH...`) |
| **Option-ID** | ID einer Single-Select-Option (z.B. `ebff64ac`) |
| **Project-ID** | Node-ID des gesamten Projects (z.B. `PVT_kwH...`) |
| **Project Number** | Numerische Projekt-Nummer (z.B. `1`) — wird in CLI-Befehlen verwendet |
| **Owner** | GitHub-Benutzername bei User-Level-Projects |
| **Content** | Das eigentliche Issue oder PR hinter einem Item |
| **User-Level Project** | Project auf Benutzerebene (nicht Organization) — kann mehrere Repos umspannen |
