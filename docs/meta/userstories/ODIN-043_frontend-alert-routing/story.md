# ODIN-043: Frontend Alert Routing und Escalation Display

**Modul:** odin-frontend (its-odin-ui)
**Phase:** 1
**Groesse:** S
**Abhaengigkeiten:** ODIN-037 (SSE Events)

---

## Kontext

Das Konzept definiert 4 Alert-Level (INFO, WARNING, CRITICAL, EMERGENCY) mit unterschiedlicher Darstellung und Eskalation. Das Frontend hat einen AlertFeed aber ohne Level-Differenzierung und ohne Eskalations-Logik.

## Scope

**In Scope:**
- Alert-Level-Differenzierung: Farbcodiert, Icon, Sound (optional fuer CRITICAL/EMERGENCY)
- Toaster-Notifications fuer CRITICAL und EMERGENCY
- Alert-History mit Filter nach Level
- Dismiss-Logik: INFO/WARNING koennen dismissed werden, CRITICAL/EMERGENCY bleiben persistent

**Out of Scope:**
- Backend-Alert-Generierung (ist in anderen Stories)
- Alert-Persistenz in Datenbank
- E-Mail oder Push-Notifications (nur UI)
- Mobile-Responsiveness (Desktop-only)

## Akzeptanzkriterien

- [ ] INFO: Blaues Badge, normaler Feed-Eintrag
- [ ] WARNING: Gelbes Badge, visuell hervorgehoben
- [ ] CRITICAL: Oranges Badge + Toaster-Notification, persistent (kein Auto-Dismiss)
- [ ] EMERGENCY: Rotes Badge + Toaster + visueller Full-Screen-Hint, persistent
- [ ] Alert-History: Filterbar nach Level, zeitlich sortiert (neueste zuerst)
- [ ] Dismiss: INFO/WARNING per Klick, CRITICAL/EMERGENCY nur nach explizitem Acknowledge
- [ ] SSE-Event ALERT wird korrekt empfangen, geparst und geroutet
- [ ] Bei SSE-Disconnect: Pending Alerts bleiben sichtbar (kein Verlust)

## Technische Details

**Dateien:**
- `its-odin-ui/src/domains/dashboard/components/AlertFeed.tsx` (Erweiterung)
- `its-odin-ui/src/domains/trading-operations/components/TradingAlertFeed.tsx` (Erweiterung)
- `its-odin-ui/src/shared/components/ToasterNotification.tsx` (neue Komponente)

**Patterns:**
- Union-Type fuer Alert-Level: `'INFO' | 'WARNING' | 'CRITICAL' | 'EMERGENCY'`
- Discriminated Union fuer SSE ALERT-Event
- CSS Modules + Dark Theme

## Konzept-Referenzen

- `docs/concept/10-observability.md` -- Abschnitt 8 "Alert-Routing und Eskalation"
- `docs/concept/10-observability.md` -- Abschnitt 8, Tabelle "4 Alert-Level"

## Guardrail-Referenzen

- `docs/frontend/guardrails/frontend.md` -- "CSS Modules, Dark Theme"
- `docs/frontend/guardrails/frontend.md` -- "TypeScript strict, Union-Types, Discriminated Unions fuer SSE-Events"
- `T:/codebase/its_odin/CLAUDE.md` -- Frontend-Architektur, SSE-Konventionen

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Frontend baut fehlerfrei (`npm run build` bzw. Vite)
- [ ] TypeScript strict -- kein `any`, keine ungetypten Props
- [ ] Union-Type fuer Alert-Level (`'INFO' | 'WARNING' | 'CRITICAL' | 'EMERGENCY'`) -- kein `enum`-Keyword
- [ ] Discriminated Union fuer ALERT SSE-Event-Typ
- [ ] CSS Modules fuer alle Styles (kein Inline-Style, kein globales CSS)
- [ ] Dark Theme konsistent eingehalten (Farbwerte gemaess Konzept/Chart-Farben-Vorgabe)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + Kommentare)
- [ ] Feature→Feature-Imports verboten (ToasterNotification als shared/ Komponente)

### 2.2 Tests -- Klassenebene (Unit-Tests)

- [ ] Vitest-Unit-Tests fuer Alert-Level-Routing-Logik (welches Level → welche Darstellung)
- [ ] Unit-Tests fuer Dismiss-Logik: INFO/WARNING dismissbar, CRITICAL/EMERGENCY persistent
- [ ] Unit-Tests fuer SSE ALERT-Event-Parser und Typ-Konvertierung
- [ ] Unit-Tests fuer Alert-History-Filter (Filterung nach Level)
- [ ] Neue Logik → Unit-Test PFLICHT

### 2.3 Tests -- Komponentenebene (Integrationstests)

- [ ] Integrationstests mit realen Komponenten zusammengeschaltet (nicht alles weggemockt)
- [ ] `AlertFeed` Integration: SSE ALERT-Event rein → korrektes Rendering je Alert-Level
- [ ] `ToasterNotification` Integration: CRITICAL/EMERGENCY → Toaster erscheint, bleibt bis Acknowledge
- [ ] Mindestens 1 Integrationstest der die EMERGENCY-Eskalation End-to-End abdeckt

### 2.4 Tests -- Datenbank

Nicht zutreffend -- reine Frontend-Story, kein Datenbankzugriff.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: Komponenten-Code, Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt (z.B. mehrere EMERGENCY-Alerts gleichzeitig, Alert waehrend SSE-Reconnect, sehr langer Alert-Text, Acknowledge bei mehreren persistenten Alerts)
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

### 2.6 Review durch Gemini -- Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Memory Leaks im Alert-State bei vielen eingehenden Alerts (ungebegrenztes Wachstum?), Race Conditions bei gleichzeitigen SSE-Events, fehlende Key-Props in Listen, Accessibility-Probleme"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/10-observability.md` Abschnitt 8 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die Implementierung dem Konzept entspricht -- insb. korrekte Alert-Level-Semantik, Persist-Verhalten je Level, Farb-Codierung, Full-Screen-Hint fuer EMERGENCY"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es fachlich oder technisch Themen, die im Konzept nicht behandelt sind? Z.B. maximale Anzahl Alerts im Feed (Performance), Sortierung bei gleichem Timestamp, Sound-Support Browser-Restrictions, Barrierefreiheit fuer EMERGENCY-Hinweis?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert

### 2.7 Protokolldatei (`protocol.md`)

- [ ] `protocol.md` angelegt im Story-Verzeichnis
- [ ] Working State nach initialer Implementierung dokumentiert
- [ ] Design-Entscheidungen dokumentiert (z.B. maximale Alert-History-Groesse, Sound-Implementation)
- [ ] ChatGPT-Sparring-Abschnitt ausgefuellt nach Test-Sparring
- [ ] Gemini-Review-Abschnitt ausgefuellt nach Review
- [ ] Offene Punkte dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- Named SSE events brauchen `addEventListener("alert", ...)`, NICHT `onmessage`
- Alert-State im Hook halten, nicht in globalem Store -- Scope ist pro SSE-Connection
- CRITICAL/EMERGENCY: Toaster darf sich NICHT selbst schliessen -- nur explizites Acknowledge
- Sound ist optional -- wenn implementiert, dann nur nach User-Interaktion (Browser-Policy)
- Alert-Feed-Groesse begrenzen (z.B. max 100 Eintraege) um Memory-Leaks zu vermeiden
