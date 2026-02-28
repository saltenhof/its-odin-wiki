# ODIN-032: Trade Intent HMAC Signing

**Modul:** odin-audit + odin-execution
**Phase:** 1
**Groesse:** M
**Abhaengigkeiten:** keine
**Geschaetzter Umfang:** M

---

## Kontext

Das Konzept fordert HMAC-Signierung jedes Trade-Intent: Nachweisbarkeit, dass kein Intent nachtraeglich manipuliert wurde. Die Execution-Schicht validiert die Signatur vor Order-Submission.

## Scope

**In Scope:**
- HMAC-Generierung in odin-brain (Arbiter, nach Intent-Erstellung)
- HMAC-Validierung in odin-execution (RiskGate, vor Order-Submission)
- Secret-Key aus Environment-Variable (nicht in Config-File)
- Payload: Instrument + Richtung + Stueckzahl + Limit-Preis + Timestamp
- `TradeIntent` Record erhaelt neues Feld `hmacSignature`

**Out of Scope:**
- Secret-Key-Rotation (V1 nicht noetig, aber API muss Key-Austausch ermoeglichen)
- HMAC-Signierung anderer DTOs (nur TradeIntent)
- Zertifikatsbasierte Signierung (nur HMAC)

## Akzeptanzkriterien

- [ ] TradeIntent enthaelt `String hmacSignature` (HmacSHA256)
- [ ] Signatur-Payload: `instrumentId|direction|intentType|entryPrice|stopLevel|marketTime`
- [ ] Secret-Key ausschliesslich aus ENV-Variable `ODIN_HMAC_SECRET`
- [ ] RiskGate validiert HMAC vor Order-Weiterleitung
- [ ] Bei ungueltiger Signatur: Order REJECT + ALERT
- [ ] Bei fehlendem Secret: Fallback auf unsigned (mit Warning, kein Exception-Crash)

## Technische Details

**Dateien:**
- `odin-api/src/main/java/de/its/odin/api/dto/TradeIntent.java` (neues Feld `hmacSignature`)
- `odin-brain/src/main/java/de/its/odin/brain/arbiter/IntentSigner.java` (neue Klasse)
- `odin-execution/src/main/java/de/its/odin/execution/risk/IntentVerifier.java` (neue Klasse)
- `odin-execution/src/main/java/de/its/odin/execution/risk/RiskGate.java` (Verifikations-Logik einbinden)

## Konzept-Referenzen

- `docs/concept/06-risk-management.md` — Abschnitt 7.4 "Trade-Intent-Signierung"
- `docs/concept/10-observability.md` — Abschnitt 4 "Audit Trail: HMAC"

## Guardrail-Referenzen

- `T:\codebase\its_odin\CLAUDE.md` — "Secrets nie in Properties-Dateien (nur ENV/JVM)"
- `docs/backend/guardrails/module-structure.md` — Abhaengigkeitsregeln (odin-api haengt von keinem Fachmodul ab)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht

---

## Definition of Done

### 2.1 Code-Qualitaet

- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-api,odin-brain,odin-execution`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten (z.B. `HMAC_ALGORITHM`, `ENV_VAR_NAME`, `PAYLOAD_DELIMITER`)
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen (kompakt aber vollstaendig)
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch (Code + JavaDoc). Kommentare: Englisch
- [ ] Namespace-Konvention: `odin.{modul}.{komponente}.{property}` fuer Konfiguration
- [ ] Port-Abstraktion: Gegen Interfaces aus `de.its.odin.api.port` programmieren

**Referenz:** CLAUDE.md → Coding-Regeln (R1-R13), Guardrail `module-structure.md`

### 2.2 Tests — Klassenebene (Unit-Tests)

- [ ] Unit-Tests fuer `IntentSigner`: HMAC-Generierung mit bekanntem Secret und Payload, Ergebnis deterministisch
- [ ] Unit-Tests fuer `IntentVerifier`: Gueltige Signatur akzeptiert, manipulierter Intent abgelehnt
- [ ] Unit-Tests: Fehlender Secret (`ODIN_HMAC_SECRET` nicht gesetzt) → Fallback auf unsigned mit Warning
- [ ] Unit-Tests: Leere oder null Felder im Payload korrekt behandelt
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

**Referenz:** CLAUDE.md → Tests

### 2.3 Tests — Komponentenebene (Integrationstests)

- [ ] Integrationstest: Vollstaendige Strecke `IntentSigner` → `TradeIntent` → `IntentVerifier` → Entscheidung (Accept/Reject) mit realen Implementierungen
- [ ] Integrationstest: RiskGate verwirft Order bei manipuliertem HMAC und erzeugt ALERT
- [ ] Integrationstest: Fallback-Modus (kein Secret) laeuft ohne Exception durch
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest der die Hauptfunktionalitaet End-to-End abdeckt

**Ziel:** Sicherstellen, dass IntentSigner und IntentVerifier korrekt zusammenarbeiten und RiskGate bei Manipulation blockiert.

### 2.4 Tests — Datenbank

Nicht zutreffend. Diese Story hat keinen Datenbankzugriff — HMAC-Signierung und Validierung sind reine In-Memory-Operationen.

### 2.5 Test-Sparring mit ChatGPT

- [ ] ChatGPT-Session gestartet mit: `IntentSigner.java`, `IntentVerifier.java`, `RiskGate.java` (Erweiterung), Akzeptanzkriterien, bereits geschriebene Tests
- [ ] ChatGPT nach Grenzfaellen gefragt: Timing-Angriffe auf HMAC-Vergleich? Replay-Attacken (gleicher Intent zweimal)? Payload-Encoding-Probleme (Sonderzeichen im instrumentId)?
- [ ] Relevante Vorschlaege bewertet und sinnvolle Tests umgesetzt
- [ ] Ergebnis im `protocol.md` dokumentiert (was vorgeschlagen wurde, was umgesetzt/verworfen wurde und warum)

**Protokoll:** `chatgpt-pool-review` Skill oder manuell via Pool-MCP. Slot IMMER releasen!

### 2.6 Review durch Gemini — Drei Dimensionen

#### Dimension 1: Code-Review (Bugs & Implementierungsfehler)
- [ ] Code an Gemini uebergeben mit Auftrag: "Pruefe auf Bugs, Implementierungsfehler, Timing-Angriffe bei HMAC-Vergleich (constant-time comparison), Null-Safety, Ressource-Leaks beim Mac-Objekt"
- [ ] Findings bewertet und berechtigte Findings behoben

#### Dimension 2: Konzepttreue-Review
- [ ] Code + `docs/concept/06-risk-management.md` Abschnitt 7.4 + `docs/concept/10-observability.md` Abschnitt 4 an Gemini uebergeben
- [ ] Auftrag: "Pruefe ob die HMAC-Implementierung dem Konzept entspricht. Stimmen Payload-Felder, Secret-Handling, Fallback-Verhalten und Reject-Logik mit den Akzeptanzkriterien ueberein?"
- [ ] Abweichungen bewertet: begruendet abweichen oder korrigieren

#### Dimension 3: Praxis-Review (unbehandelte Themen)
- [ ] Auftrag: "Gibt es bei HMAC-Signierung in Trading-Systemen typische Luecken, die hier nicht behandelt sind? Welche realen Szenarien (Replay-Angriff, Secret-Kompromittierung, Key-Rotation unter Last) koennten auftreten?"
- [ ] Neue Erkenntnisse im `protocol.md` unter "Offene Punkte" dokumentiert
- [ ] Kritische Findings an Stakeholder eskaliert (nicht eigenmaechtig loesen)

**Protokoll:** `gemini-pool-review` Skill. Slot IMMER releasen!

### 2.7 Protokolldatei (`protocol.md`)

- [ ] Datei `protocol.md` im Story-Verzeichnis angelegt
- [ ] Working State wird bei jedem Meilenstein aktualisiert
- [ ] Design-Entscheidungen dokumentiert (z.B. Payload-Delimiter-Wahl, Fallback-Strategie bei fehlendem Secret)
- [ ] Offene Punkte erfasst
- [ ] ChatGPT-Sparring-Ergebnisse dokumentiert
- [ ] Gemini-Review-Ergebnisse (alle drei Dimensionen) dokumentiert

### 2.8 Abschluss

- [ ] Commit mit aussagekraeftiger Message (beschreibt das "Warum", nicht das "Was")
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

---

## Notizen fuer den Implementierer

- HmacSHA256 aus JDK (`javax.crypto.Mac`) — keine externe Abhaengigkeit
- Constant-time Vergleich beim HMAC-Check verwenden (`MessageDigest.isEqual`) um Timing-Angriffe zu verhindern
- Secret-Key Rotation: Aktuell nicht noetig (V1), aber API so bauen dass Key austauschbar ist (kein Singleton-Secret, sondern Injection)
- Abhaengigkeitsregel beachten: odin-api darf keine Fachmodule importieren — `TradeIntent` bleibt ein einfaches Record ohne Signierlogik
