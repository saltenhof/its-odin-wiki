# ODIN — Orderflow Detection & Inference Node

Vollautomatischer Intraday-Trading-Agent fuer Aktien (US, Europa). Kombiniert LLM-Situationsanalyse, deterministische Regellogik und quantitative Validierung.

**Multi-Instrument (2-3 parallel) | Single-Day | Long-Only | EOD-Flat**

---

## Dokumentation

| Bereich | Inhalt |
|---------|--------|
| [Fachkonzept](concept/intraday-agent-concept.md) | Was ODIN fachlich tut — Strategielogik, Entry/Exit, Regime, Risk |
| [Architektur](architecture/index.md) | Wie ODIN technisch gebaut ist — 11 Kapitel von Systemuebersicht bis Deployment |
| [Guardrails](guardrails/index.md) | Verbindliche Bauvorschriften fuer Backend, Frontend und Entwicklungsprozess |
| [Meta](meta/playbook.md) | Playbook, Working State, Konzept-Changelog |

## Tech-Stack

| Bereich | Technologie |
|---------|------------|
| Backend | Java 21, Spring Boot, Maven Multi-Modul (10 Module) |
| Broker | Interactive Brokers TWS API 10.40 |
| KPI | ta4j |
| Datenbank | PostgreSQL |
| Frontend | React 18+, TypeScript, Vite |
| LLM | Claude Agent SDK (primaer), OpenAI API (alternativ) |
| Betrieb | On-Prem, Windows |

## Repos

| Repo | Zweck |
|------|-------|
| `its-odin-backend` | Java-Backend (10 Maven-Module) |
| `its-odin-ui` | React-Frontend |
| `its-odin-wiki` | Dieses Wiki |
