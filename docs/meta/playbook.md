# ODIN — User-Story-Execution Playbook

**Geltungsbereich:** Alle ODIN User Stories (Backend, Frontend, Infrastruktur)
**Komplementaerdokumente:**

| Dokument | Pfad | Inhalt |
|----------|------|--------|
| User-Story-Spezifikation | `docs/meta/user-story-specification.md` | Story-Struktur, DoD, Checklisten |
| User-Story-Index | `docs/meta/userstories/INDEX.md` | Fortschritt, Waves, Status aller Stories |
| CLAUDE.md | `T:\codebase\its_odin\CLAUDE.md` | Coding-Regeln, Architektur, Guardrails |

---

## 1. Rollen

### Orchestrator (Primary Claude)

Der Orchestrator **implementiert nichts selbst** — keine Datei-Reads, kein Code, keine Analyse. Er:

- Plant und priorisiert Stories anhand des Index und der Abhaengigkeiten
- Startet Worker-Agents und QS-Agents mit klar definierten Auftraegen
- Empfaengt binaere Rueckmeldungen (PASS / FAIL / Eskalation)
- Verifiziert Ergebnisse anhand von Beweisen (Build-Logs, Test-Counts, Diffs)
- Eskaliert an den User bei Blockern oder nach 3 gescheiterten Runden
- Haelt seinen Kontext sauber — keine Implementierungsdetails

### Worker-Agent (Implementierung)

Ein dedizierter Sub-Agent pro Story. Er:

- Liest die Story-Definition (`story.md`) und alle referenzierten Dokumente
- Implementiert Code, Tests, ChatGPT-Sparring und Gemini-Review
- Fuehrt die komplette DoD ab (siehe `user-story-specification.md` Abschnitt 2)
- Schreibt das Protokoll (`protocol.md`) live waehrend der Arbeit
- Committed und pusht bei Abschluss
- Bricht sofort ab bei blockierten DoD-Schritten (Fail-Fast)

### QS-Agent (Qualitaetssicherung)

Ein dedizierter Sub-Agent pro Story, gestartet NACH dem Worker. Er:

- Prueft alle DoD-Kriterien gegen den tatsaechlichen Code und die Protokolldatei
- Fuehrt Build und Tests aus (`mvn clean install`)
- Vergleicht Implementierung gegen Konzeptdokumente (Konzepttreue)
- Schreibt Findings in `temp/userstories/<STORY-ID>/qa-report-r<N>.md`
- Meldet binaeres Signal an den Orchestrator: **PASS** oder **FAIL**
- Bei PASS: kein weiterer Handlungsbedarf
- Bei FAIL: Findings-Datei enthaelt strukturierte Maengelliste

---

## 2. Execution-Zyklus

```
fuer jede Story:
  runde = 1
  LOOP:
    starte Worker-Agent (Background, Opus)
    warte auf Abschluss → verifiziere Beweise
    starte QS-Agent (Background, Opus)
    warte auf binaeres Signal:
      PASS  → Story abgeschlossen → BREAK
      FAIL und runde < 3 → runde++ → LOOP (neuer Worker mit qa-report als Input)
      FAIL und runde == 3 → ESKALATION an User
```

### 2.1 Worker-Runde

1. Orchestrator startet Worker-Agent mit vollstaendigem Prompt (siehe Abschnitt 4)
2. Worker arbeitet Story ab: Code → Unit-Tests → ChatGPT-Sparring → Integrationstests → Gemini-Review (3 Dimensionen) → Findings einarbeiten → Protokoll → Commit + Push
3. Worker meldet Abschluss mit Beweisen (Build-Output, Test-Counts)
4. Orchestrator verifiziert: Build GREEN? Test-Counts plausibel? Commit vorhanden?

### 2.2 QS-Runde

1. Orchestrator startet QS-Agent mit: Story-Pfad, Spezifikation, Rundennummer
2. QS-Agent prueft alle DoD-Punkte systematisch
3. QS-Agent schreibt Findings: `temp/userstories/<STORY-ID>/qa-report-r<N>.md`
4. QS-Agent meldet PASS oder FAIL

### 2.3 Remediation (bei FAIL)

1. Orchestrator startet neuen Worker-Agent (frischer Kontext!)
2. Worker erhaelt zusaetzlich: Pfad zur `qa-report-r<N>.md` der vorherigen Runde
3. Worker behebt ALLE Findings vollstaendig
4. Danach erneut QS-Runde

### 2.4 Eskalation (nach 3x FAIL)

Nach 3 gescheiterten Runden unterbricht der Orchestrator den Zyklus und informiert den User mit:

- Story-ID und Titel
- Anzahl Runden (3)
- Pfad zur letzten Findings-Datei (`qa-report-r3.md`)
- Zusammenfassung der wiederkehrenden Probleme
- Empfehlung fuer naechste Schritte

Der User entscheidet, ob und wie der Zyklus fortgesetzt wird.

---

## 3. Kernregeln

### 3.1 Eine Story = Ein Agent (Kontexthygiene)

**IMMER einen frischen Agent pro Story.** Niemals mehrere Stories in einem Agent buendeln.

Begruendung: Context-Compaction loescht Agent-Instruktionen → DoD-Schritte werden uebersprungen. Empirisch bestaetigt: Agent mit 5 Stories hat systematisch Reviews und Tests ausgelassen.

### 3.2 Fail-Fast bei Blockern

Wenn ein Worker einen Pflichtschritt nicht ausfuehren kann (ChatGPT/Gemini nicht verfuegbar, Build-Fehler nicht loesbar, fehlende Dependency):

1. Sofort abbrechen — Schritt NICHT ueberspringen
2. Fehlermeldung: welcher Schritt blockiert, warum, was fehlt
3. Keine Partial-Completion — eine Story ist vollstaendig oder gar nicht fertig

### 3.3 Parallelisierung

| Regel | Beschreibung |
|-------|-------------|
| Max. Agents | 3 gleichzeitig (Worker + QS zaehlen zusammen) |
| Module | Stories in verschiedenen Modulen: parallel moeglich |
| Gleiches Modul | Sequenziell oder per Git-Worktree isolieren |
| ChatGPT-Pool | Max. 4 parallele Slots |
| Gemini-Pool | Max. 4 parallele Slots |
| Background-Mode | Alle Agents im Background (Orchestrator bleibt responsiv) |

### 3.4 Modell-Auswahl

| Agent-Typ | Modell | Begruendung |
|-----------|--------|-------------|
| Worker (Standard) | Sonnet | Effizient fuer klar definierte Implementierungen |
| Worker (komplex/konzeptionell) | Opus | Fuer Stories mit L/XL-Umfang oder architektonischen Entscheidungen |
| QS-Agent | Sonnet | Pruefung gegen klar definierte Checkliste |
| Exploration/Research | Sonnet oder Opus | Je nach Tiefe der Fragestellung |

### 3.5 Anti-Loop

Nach 2 gescheiterten Versuchen mit demselben Ansatz: **STOPP.** Der Ansatz ist falsch.

Wechsel zu Bottom-Up: Unit-Tests fuer einzelne Klassen entlang der Call-Chain, dann Komponententests. Schichtweise Korrektheit beweisen, BEVOR Full-Stack-Retry.

### 3.6 Beweispflicht

Kein Deliverable ohne Beweis. "Sieht korrekt aus" ist KEINE Verifikation.

| Beweistyp | Beispiel |
|-----------|---------|
| Build | `BUILD SUCCESS`, Test-Counts sichtbar |
| Tests | Anzahl UT + IT, alle GREEN |
| Commit | Commit-Hash, aussagekraeftige Message |
| Review | ChatGPT-Sparring und Gemini-Review im Protokoll dokumentiert |

---

## 4. Prompt-Struktur

### 4.1 Worker-Agent (Implementierung)

Jeder Worker-Prompt MUSS enthalten:

| # | Bestandteil | Inhalt |
|---|-------------|--------|
| 1 | Story-Datei | Pfad zur `story.md` — Agent liest sie selbst |
| 2 | User-Story-Spezifikation | `docs/meta/user-story-specification.md` — vollstaendige DoD |
| 3 | CLAUDE.md | `T:\codebase\its_odin\CLAUDE.md` — Coding-Regeln, Architektur |
| 4 | Konzeptdateien | Pfade zu den in der Story referenzierten Konzeptdokumenten |
| 5 | Guardrails | Pfade zu den relevanten Guardrail-Dokumenten |
| 6 | QA-Report | Bei Remediation: Pfad zur `qa-report-r<N>.md` der Vorrunde |
| 7 | Rundennummer | Explizit angeben (fuer QS-Agent-Dateinamen) |
| 8 | Ausfuehrungsanweisung | "Lies die User-Story-Spezifikation und befolge ALLE DoD-Punkte 2.1 bis 2.8. Ueberspringe KEINEN Schritt. Bei Blockern: abbrechen und Fehler melden." |

### 4.2 QS-Agent

Jeder QS-Prompt MUSS enthalten:

| # | Bestandteil | Inhalt |
|---|-------------|--------|
| 1 | Story-Datei | Pfad zur `story.md` |
| 2 | User-Story-Spezifikation | `docs/meta/user-story-specification.md` |
| 3 | CLAUDE.md | `T:\codebase\its_odin\CLAUDE.md` |
| 4 | Rundennummer | N (bestimmt Dateiname: `qa-report-r<N>.md`) |
| 5 | Pruefauftrag | "Pruefe ALLE DoD-Kriterien aus Abschnitt 2.1-2.8 gegen den tatsaechlichen Code und die Protokolldatei. Schreibe Findings in `temp/userstories/<STORY-ID>/qa-report-r<N>.md`. Melde PASS oder FAIL." |

### 4.3 Pflichtregeln im Prompt

Folgende Regeln muessen in JEDEM Agent-Prompt explizit stehen (werden bei Context-Compaction sonst vergessen):

- **DB-Zugriff:** `"T:/codebase/its_odin/tools/odin-db.sh" "SQL"` — kein direktes `psql`
- **Build vor Tests:** `mvn clean install -DskipTests` vor jedem Test-Run oder Spring-Boot-Start
- **Backend-Start:** Nur aus Hauptagent-Kontext mit `run_in_background`
- **Temp-Dateien:** Nach Verwendung sofort loeschen, niemals committen

---

## 5. DoD-Kurzreferenz

Vollstaendige DoD in `user-story-specification.md` Abschnitt 2. Hier die Schritte im Ueberblick:

```
1. Code-Implementierung (2.1: Qualitaet, Naming, JavaDoc, keine TODOs)
2. Unit-Tests (2.2: *Test, Mocks fuer Ports, Bugfix → reproduzierender Test)
3. Integrationstests (2.3: *IntegrationTest, reale Klassen, min. 1 E2E pro Story)
4. DB-Tests falls zutreffend (2.4: Zonky, Flyway, echte DB)
5. ChatGPT-Sparring (2.5: Edge Cases, Grenzfaelle → protocol.md)
6. Gemini-Review 3 Dimensionen (2.6: Code-Bugs, Konzepttreue, Praxis-Gaps)
7. Findings einarbeiten
8. Protokolldatei aktualisieren (2.7: protocol.md mit allen Pflichtabschnitten)
9. Commit + Push (2.8: aussagekraeftige Message)
```

---

## 6. Orchestrator-Checkliste pro Story

Vor dem Start:

- [ ] Story hat `story.md` mit allen Pflichtfeldern
- [ ] Alle Abhaengigkeiten sind PASS
- [ ] Kein anderer Agent arbeitet am gleichen Modul (oder Worktree-Isolation)
- [ ] Prompt enthaelt alle 8 Pflichtbestandteile (Abschnitt 4.1)

Nach Worker-Abschluss:

- [ ] Build GREEN (Beweis gesehen)
- [ ] Test-Counts plausibel (UT + IT)
- [ ] Commit vorhanden
- [ ] Protokolldatei existiert und ist ausgefuellt

Nach QS-Abschluss:

- [ ] Binaeres Signal empfangen (PASS oder FAIL)
- [ ] Bei FAIL: Findings-Datei existiert, Runde inkrementieren
- [ ] Bei PASS: Story im Index als Done markieren
- [ ] Bei 3x FAIL: Eskalation an User

---

## 7. Hinweise fuer Context-Recovery

Wenn der Orchestrator nach einem Context-Reset hier landet:

1. **Lies dieses Playbook** — Prozessregeln
2. **Lies den Index** (`docs/meta/userstories/INDEX.md`) — aktueller Fortschritt
3. **Pruefe laufende Agents** — gibt es offene Worker oder QS-Agents?
4. **Pruefe `temp/userstories/`** — gibt es offene QA-Reports ohne Abschluss?
5. **Weiter im Zyklus** — naechste Story oder Remediation-Runde starten
