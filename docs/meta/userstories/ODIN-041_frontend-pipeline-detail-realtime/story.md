# ODIN-041: Frontend Pipeline Detail Realtime View

**Modul:** odin-frontend (its-odin-ui)
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** ODIN-040 (Live Dashboard), ODIN-037 (SSE Events)

---

## Kontext

Die Pipeline-Detail-Seite existiert aber zeigt statische/Mock-Daten. Sie muss Echtzeit-Daten zeigen: Live-Chart mit Indikatoren, aktuelle Position mit Tranchen, Decision-Log, LLM-History.

## Scope

**In Scope:**
- Per-Pipeline SSE-Stream fuer Detail-View
- Live-Chart-Updates: Candlesticks + Indikatoren aktualisieren bei neuen Bars
- Position-Panel: Tranchen-Tabelle, Stop-Level, Trail-Level, R-Multiple
- Decision-Log: Letzte N Entscheidungen mit Arbiter-Reasoning
- LLM-History: Letzte N LLM-Analysen mit Action, Confidence, Reasoning

**Out of Scope:**
- Historische Chart-Daten (nur aktuelle Intraday-Session)
- Editierbarkeit von Position-Parametern (nur Anzeige)
- Mobile-Responsiveness (Desktop-only)

## Akzeptanzkriterien

- [ ] Chart aktualisiert sich bei INDICATOR_UPDATE Events
- [ ] Position-Panel zeigt: Qty, Avg Entry, Current Price, Unrealized P&L, R-Multiple, Trail-Level
- [ ] Tranchen-Tabelle: Tranche-Label, Qty, Target, Status (open/filled)
- [ ] Decision-Log: IntentType, Regime, Confidence, Reason, Timestamp
- [ ] LLM-History: Action, ExitBias, TrailMode, Confidence, Truncated Reasoning
- [ ] Alles aktualisiert sich in Echtzeit (kein manuelles Refresh)
- [ ] SSE-Verbindungsverlust wird dem User sichtbar angezeigt (DisconnectedBanner o.ae.)

## Technische Details

**Dateien:**
- `its-odin-ui/src/domains/trading-operations/pages/PipelineDetailPage.tsx` (Erweiterung)
- `its-odin-ui/src/domains/trading-operations/components/` (diverse Erweiterungen)
- `its-odin-ui/src/domains/trading-operations/hooks/usePipelineDetail.ts` (Erweiterung)

**Patterns:**
- SSE-Hook: `addEventListener` fuer named events (NICHT `onmessage`)
- TypeScript strict: Union-Types statt enum-Keyword, Discriminated Unions fuer SSE-Events
- CSS Modules fuer alle Styles, Dark Theme

## Konzept-Referenzen

- `docs/concept/10-observability.md` -- Abschnitt 12 "Dashboard-Komponenten"
- `docs/concept/10-observability.md` -- Abschnitt 5 "Snapshot und DecisionLog Schemas"

## Guardrail-Referenzen

- `docs/frontend/guardrails/frontend.md` -- "Feature-basierte DDD-Struktur"
- `docs/frontend/guardrails/frontend.md` -- "CSS Modules, Dark Theme"
- `docs/frontend/guardrails/frontend.md` -- "TypeScript strict, Union-Types"
- `T:/codebase/its_odin/CLAUDE.md` -- Frontend-Architektur, SSE-Konventionen

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Frontend baut fehlerfrei (`npm run build` bzw. Vite)
- [ ] TypeScript strict -- kein `any`, keine ungetypten Props
- [ ] Union-Types statt `enum`-Keyword fuer endliche Mengen
- [ ] Discriminated Unions fuer SSE-Event-Typen
- [ ] CSS Modules fuer alle Styles (kein Inline-Style, kein globales CSS)
- [ ] Dark Theme konsistent eingehalten
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + Kommentare)
- [ ] Komponenten-Benennung: PascalCase, Hooks: `use`-Prefix
- [ ] Feature→Feature-Imports verboten (nur via shared/)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Vitest-Unit-Tests fuer alle Hooks mit Geschaeftslogik (insb. `usePipelineDetail`)
- [ ] Unit-Tests fuer SSE-Event-Parser und Typ-Konvertierungen
- [ ] R-Multiple-Berechnung unit-getestet: (currentPrice - entryPrice) / (entryPrice - initialStop)
- [ ] Tests fuer Disconnect-Handling (SSE-Verbindungsverlust)
- [ ] Neue Logik → Unit-Test PFLICHT

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Komponenten zusammengeschaltet (nicht alles weggemockt)
- [ ] `PipelineDetailPage` Integration: SSE-Events triggern korrekte UI-Updates
- [ ] Position-Panel + Tranchen-Tabelle: Vollstaendige Render-Kette mit echten Datenstrukturen
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt (SSE-Event rein → UI-Update)

### 2.4 Tests -- Datenbank

Nicht zutreffend -- reine Frontend-Story, kein Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Komponenten-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen, Edge Cases und uebersehenen Szenarien gefragt (z.B. SSE-Reconnect waehrend offenem Trade, leere Position, fehlendes LLM-Feld)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Race Conditions in SSE-Handling, Memory Leaks bei Event-Listener-Registrierung, fehlende Cleanup-Aufrufe in useEffect, Null-Safety bei optionalen SSE-Feldern"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht -- insb. korrekte SSE-Event-Typen, Decision-Log-Felder, LLM-History-Felder, Position-Panel-Felder"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind, die aber in der praktischen Implementierung relevant werden? Z.B. Performance bei vielen Updates, Chart-Flackern, lange LLM-Reasoning-Texte?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State nach initialer Implementierung dokumentiert
- [ ] Design-Entscheidungen dokumentiert (z.B. wie SSE-Reconnect gehandhabt wird)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt nach Test-Sparring
- [ ] Gemini-Review-Abschnitt ausgefuellt nach Review
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- IntraDayChart existiert bereits als Shared-Component -- nutzen und erweitern
- R-Multiple berechnen: (currentPrice - entryPrice) / (entryPrice - initialStop)
- Named SSE events brauchen `addEventListener("event-type", ...)`, NICHT `onmessage` (siehe MEMORY.md: SSE-Bug geloest)
- React StrictMode: Deferred disconnect (100ms grace period) in SSE-Hooks beachten
- `hasEverConnectedRef` Guard gegen false-positive red banner (siehe MEMORY.md)
