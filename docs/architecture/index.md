# Architektur

11 Kapitel beschreiben die technische Architektur von ODIN — vom Systemkontext bis zum Deployment.

Alle Kapitel sind review-komplett (2 Cross-Kapitel-Reviews mit ChatGPT, DDD-Modulschnitt).

| Kapitel | Thema | Version |
|---------|-------|---------|
| [Kap 0](00-system-overview.md) | Systemuebersicht & Architekturkontext | v0.7 |
| [Kap 1](01-module-architecture.md) | Modularchitektur & Package-Struktur | v0.5 |
| [Kap 2](02-realtime-pipeline.md) | Echtzeit-Datenpipeline | v0.4 |
| [Kap 3](03-broker-integration.md) | Broker-Anbindung (IB Gateway, TWS API 10.40) | v0.3 |
| [Kap 4](04-kpi-engine.md) | KPI-Engine (ta4j, Parallelisierung) | v0.3 |
| [Kap 5](05-llm-integration.md) | LLM-Integration (Claude SDK, OpenAI) | v0.3 |
| [Kap 6](06-rules-engine.md) | Rules Engine & Decision Loop | v0.3 |
| [Kap 7](07-oms.md) | Order Management System (OMS) | v0.3 |
| [Kap 8](08-data-model.md) | Datenmodell & Persistenz (PostgreSQL) | v0.6 |
| [Kap 9](09-frontend.md) | React-Frontend | v0.3 |
| [Kap 10](10-deployment.md) | Deployment & Betrieb (Windows, On-Prem) | v0.6 |
