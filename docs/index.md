# ODIN — Orderflow Detection & Inference Node

Vollautomatischer Intraday-Trading-Agent fuer Aktien (US, Europa). Kombiniert LLM-Situationsanalyse, deterministische Regellogik und quantitative Validierung.

**Multi-Instrument (2-3 parallel) | Single-Day | Long-Only | EOD-Flat**

---

## Dokumentation

| Bereich | Inhalt |
|---------|--------|
| [Fachkonzept](concept/00-overview.md) | Was ODIN fachlich tut — Strategielogik, Entry/Exit, Regime, Risk |
| [Architektur](architecture/index.md) | Systemuebersicht (Kap 0) — uebergreifend |
| [Backend](backend/index.md) | Architektur (Kap 1–8, 10), Guardrails |
| [Frontend](frontend/index.md) | Architektur (Kap 9), Guardrails, Requirements |
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
