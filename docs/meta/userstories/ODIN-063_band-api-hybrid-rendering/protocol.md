# Protokoll: ODIN-063 -- Band API + Hybrid UI Rendering

## Working State
- [ ] Initiale Implementierung (Backend: ZoneType Enum + SrZoneDto)
- [ ] Initiale Implementierung (Frontend: LEVEL-Rendering)
- [ ] Initiale Implementierung (Frontend: BAND-Rendering)
- [ ] Unit-Tests geschrieben (Backend)
- [ ] Unit-Tests geschrieben (Frontend)
- [ ] ChatGPT-Sparring fuer Test-Edge-Cases
- [ ] Integrationstests geschrieben (Backend)
- [ ] Integrationstests geschrieben (Frontend)
- [ ] Gemini-Review Dimension 1 (Code)
- [ ] Gemini-Review Dimension 2 (Konzepttreue)
- [ ] Gemini-Review Dimension 3 (Praxis)
- [ ] Review-Findings eingearbeitet
- [ ] Playwright E2E-Tests mit Screenshot-Beweisen
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

## Playwright E2E-Tests
[Screenshot-Referenzen mit Dateiname und Beschreibung.]

| # | Screenshot | Datei | Beschreibung |
|---|-----------|-------|-------------|
| 1 | BAND-Rendering | `e2e/screenshots/...` | Konsolidierungsband als halbtransparentes oranges Rechteck |
| 2 | LEVEL-Rendering | `e2e/screenshots/...` | Regulaere S/R-Level mit scharfer Linie + Halo |
| 3 | Farben | `e2e/screenshots/...` | Orange Band, gruener Support, roter Resistance |
| 4 | Band-Label | `e2e/screenshots/...` | "Consol. $X.XX--$Y.YY" Format |
| 5 | POC-Linie | `e2e/screenshots/...` | Gestrichelte POC-Linie innerhalb des Bands |
| 6 | Graceful Degradation | `e2e/screenshots/...` | Unbekannter zoneType als LEVEL gerendert |
