# ODIN-075: Volume Profile (VPOC/VAH/VAL)

**Modul:** odin-brain, odin-api
**Phase:** 1
**Abhaengigkeiten:** Keine
**Geschaetzter Umfang:** M

---

## Kontext

Die S/R-Engine nutzt aktuell Vortageshoch/-tief, Pre-Market-Level, VWAP, runde Zahlen und Intraday-Konsolidierungszonen als Level-Quellen. Volume Profile (VPOC, VAH, VAL) ist die institutionell am staerksten beachtete Volume-basierte Level-Quelle und fehlt bisher komplett. VPOC (Volume Point of Control) ist das Preislevel mit dem meisten kumulativen Volumen, VAH/VAL begrenzen die Zone mit 70% des Volumens. Diese Level sind informationstheoretisch unabhaengig von den bestehenden Quellen (Volumen-Konzentration statt Preis-Reaktion).

## Scope

**In Scope:**

- Neue Klasse `VolumeProfileCalculator` in `de.its.odin.brain.kpi` fuer VPOC/VAH/VAL-Berechnung aus 1m-Bars
- Neues Record `VolumeProfile` mit VPOC, VAH, VAL und Value-Area-Prozent
- Integration der Volume-Profile-Level in `SrLevelEngine` als zusaetzliche Level-Quelle (`SrLevelSide.VPOC`, `SrLevelSide.VAH`, `SrLevelSide.VAL`)
- Neues Enum-Wert `VPOC`, `VAH`, `VAL` in `SrLevelSide` (oder neues passendes Enum falls SrLevelSide nicht passend)
- Konfigurationsparameter in `BrainProperties` (Namespace `odin.brain.kpi.volume-profile.*`)
- Taeglicher Reset des Volume Profiles (analog VWAP)

**Out of Scope:**

- Volume-Profile-Visualisierung im Frontend (separate Story)
- Multi-Day Volume Profile (nur Intraday-Session)
- Microstructure-Analyse auf Tick-Level
- Integration in LLM-Prompt (kann spaeter ueber SrSnapshot automatisch erfolgen)

## Akzeptanzkriterien

- [ ] Neue Klasse `VolumeProfileCalculator` berechnet VPOC, VAH, VAL aus einer Liste von 1m-Bars
- [ ] VPOC: Preislevel mit hoechstem kumulativem Volumen (Bins von konfigurierbarer Groesse, Default: $0.05)
- [ ] Value Area: Zone um VPOC die 70% des Gesamtvolumens enthaelt (Prozentsatz konfigurierbar)
- [ ] VAH: Obere Grenze der Value Area, VAL: Untere Grenze der Value Area
- [ ] `VolumeProfile` Record enthaelt: `double vpoc, double vah, double val, double totalVolume, int barCount`
- [ ] Berechnung wird inkrementell aktualisiert bei jedem neuen 1m-Bar (nicht komplett neu berechnet)
- [ ] Unit-Test: 10 Bars mit bekanntem Volumenprofil → VPOC ist das Preislevel mit meistem Volumen
- [ ] Unit-Test: Value Area enthaelt exakt 70% des Gesamtvolumens (+-1 Bin Toleranz)
- [ ] Unit-Test: Bei gleichem Volumen auf allen Level → VPOC ist der median-Preis
- [ ] Unit-Test: Leere Bar-Liste → VolumeProfile mit NaN-Werten
- [ ] `SrLevelEngine` erzeugt S/R-Level aus VPOC (Support UND Resistance), VAH (Resistance), VAL (Support)
- [ ] VPOC-Level hat erhoehte initiale Confidence (VPOC ist institutionelles Haupt-Level)
- [ ] Integrationstest: SrLevelEngine mit VolumeProfileCalculator liefert VPOC/VAH/VAL als Teil des SrSnapshot

## Technische Details

### Bestehende Klassen (werden erweitert)

| Klasse | Pfad | Aenderung |
|--------|------|-----------|
| `SrLevelEngine` | `odin-brain/.../sr/SrLevelEngine.java` | Neuer Aufruf `addVolumeProfileLevels()` nach Bar-Update |
| `SrLevel` | `odin-brain/.../sr/SrLevel.java` | Neuer `SrLevelSource` fuer VPOC/VAH/VAL |
| `KpiEngine` | `odin-brain/.../kpi/KpiEngine.java` | Optional: VolumeProfileCalculator als Feld, Update bei jedem 1m-Bar |
| `BrainProperties` | `odin-brain/.../config/BrainProperties.java` | Neues `VolumeProfileProperties` in `KpiProperties` |

### Neue Klassen

| Klasse | Package | Beschreibung |
|--------|---------|-------------|
| `VolumeProfileCalculator` | `de.its.odin.brain.kpi` | Inkrementelle Berechnung von VPOC/VAH/VAL aus 1m-Bars |
| `VolumeProfile` | `de.its.odin.brain.kpi` | Immutable Record: VPOC, VAH, VAL, totalVolume, barCount |

### Algorithmus: VPOC/VAH/VAL

1. **Price-Binning:** Alle Bars in Preis-Bins aufteilen (Bin-Groesse konfigurierbar, Default $0.05). Fuer jeden Bar wird das Volumen gleichmaessig auf die Bins zwischen Low und High verteilt.
2. **VPOC:** Bin mit hoechstem kumulativem Volumen → Mittelpreis dieses Bins.
3. **Value Area:** Ausgehend vom VPOC-Bin abwechselnd den naechsten Bin oberhalb und unterhalb hinzufuegen (jeweils den mit mehr Volumen), bis 70% des Gesamtvolumens erreicht sind.
4. **VAH/VAL:** Obere/untere Grenze der resultierenden Zone.

### Konfiguration

```properties
odin.brain.kpi.volume-profile.enabled=true
odin.brain.kpi.volume-profile.bin-size=0.05
odin.brain.kpi.volume-profile.value-area-percent=70.0
odin.brain.kpi.volume-profile.vpoc-initial-confidence=0.70
```

## Konzept-Referenzen

- `theme-backlog.md` Thema 13: "Volume Profile (VPOC/VAH/VAL) als S/R-Quelle" — IST-Zustand, SOLL-Zustand, 80%-Regel
- `docs/backend/architecture/04-kpi-engine.md` — KPI-Engine Architektur, Multi-Timeframe
- `docs/backend/architecture/02-realtime-pipeline.md` — Datenpipeline, Bar-Updates
- MEMORY.md: "Support/Resistance Level Detection" — VPOC explizit als Quelle fuer S/R-Level genannt

## Guardrail-Referenzen

- `docs/backend/guardrails/module-structure.md` — Package-Konventionen (`de.its.odin.brain.kpi`)
- `docs/backend/guardrails/development-process.md` — Phasen, Checkpoints, Review-Pflicht
- `CLAUDE.md` — Coding-Regeln (Records fuer DTOs, keine Magic Numbers, explizite Typen)

## Definition of Done

### 2.1 Code-Qualitaet
- [ ] Implementierung vollstaendig gemaess Akzeptanzkriterien
- [ ] Code kompiliert fehlerfrei (`mvn compile -pl odin-brain`)
- [ ] Kein `var` — explizite Typen
- [ ] Keine Magic Numbers — `private static final` Konstanten
- [ ] Records fuer DTOs
- [ ] ENUM statt String fuer endliche Mengen
- [ ] JavaDoc auf allen public Klassen, Methoden und Attributen
- [ ] Keine TODO/FIXME-Kommentare verbleibend
- [ ] Code-Sprache: Englisch
- [ ] Namespace-Konvention: `odin.brain.kpi.volume-profile.*`
- [ ] Port-Abstraktion eingehalten

### 2.2 Tests — Klassenebene (Unit-Tests)
- [ ] Unit-Tests fuer `VolumeProfileCalculator` (VPOC, VAH, VAL, Edge Cases)
- [ ] Testklassen-Namenskonvention: `*Test` (Surefire)
- [ ] Mocks/Stubs fuer Port-Interfaces
- [ ] Neue Geschaeftslogik → Unit-Test PFLICHT

### 2.3 Tests — Komponentenebene (Integrationstests)
- [ ] Integrationstests mit realen Klassen (VolumeProfileCalculator + SrLevelEngine)
- [ ] Testklassen-Namenskonvention: `*IntegrationTest` (Failsafe)
- [ ] Mindestens 1 Integrationstest: Volume Profile fließt korrekt in SrSnapshot ein

### 2.5 Test-Sparring mit ChatGPT
- [ ] ChatGPT-Session mit Klassen, Akzeptanzkriterien, bereits geschriebenen Tests
- [ ] Edge Cases und Grenzfaelle abgefragt
- [ ] Ergebnis im `protocol.md` dokumentiert

### 2.6 Review durch Gemini — Drei Dimensionen
- [ ] Dimension 1: Code-Review (Bugs, Numerik-Fehler bei Bin-Berechnung)
- [ ] Dimension 2: Konzepttreue-Review (Vergleich mit theme-backlog.md Thema 13)
- [ ] Dimension 3: Praxis-Review (Illiquide Aktien, Gap-Bars, Volume-Spikes)
- [ ] Findings bewertet und berechtigte Findings behoben

### 2.7 Protokolldatei
- [ ] `protocol.md` im Story-Verzeichnis

### 2.8 Abschluss
- [ ] Commit mit aussagekraeftiger Message
- [ ] Push auf Remote
- [ ] Story-Verzeichnis enthaelt: `story.md` + `protocol.md`

## Notizen fuer den Implementierer

- **Inkrementelle Berechnung:** Die naive Implementierung (alle Bars neu durchgehen) ist O(n*bins) und bei 390 1m-Bars pro Tag akzeptabel. Trotzdem sollte die Berechnung inkrementell sein (neuen Bar in Bins einfuegen, VPOC/VA nur bei Aenderung neu bestimmen) — das ist sauberer und zukunftssicher.
- **Bin-Groesse:** $0.05 ist fuer Aktien im Bereich $10-$200 sinnvoll. Fuer Penny Stocks oder hochpreisige Aktien (>$500) waere eine dynamische Bin-Groesse (z.B. 0.1% des Preises) besser — das ist aber Out of Scope und kann spaeter als Konfigurationsoption ergaenzt werden.
- **SrLevelEngine-Integration:** VPOC/VAH/VAL-Level muessen wie andere statische Level (VWAP, Vortageshoch) in den Level-Pool eingefuegt werden. Sie verschieben sich mit jedem neuen Bar (im Gegensatz zu fixen Level wie Vortageshoch). Die TouchRegistry funktioniert automatisch mit diesen Level.
- **Volume-Verteilung auf Bins:** Pro Bar das Volumen gleichmaessig auf Bins zwischen Low und High verteilen ist eine Vereinfachung (reales Tick-Volumen ist nicht gleichmaessig). Fuer Intraday-Zwecke ist das ausreichend.
