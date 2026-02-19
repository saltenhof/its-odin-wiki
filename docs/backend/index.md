# Backend — Java 21 / Spring Boot

Das ODIN-Backend besteht aus 10 Maven-Modulen (odin-api bis odin-app), implementiert in Java 21 mit Spring Boot. Es umfasst die gesamte Trading-Logik von der Marktdatenverarbeitung ueber KPI-Berechnung und LLM-Analyse bis zur Orderausfuehrung.

## Architektur

9 Kapitel beschreiben die Backend-Architektur im Detail. Die querschnittliche Systemuebersicht (Kap 0) lebt auf der obersten Ebene.

| Kapitel | Thema | Version |
|---------|-------|---------|
| [Kap 1](architecture/01-module-architecture.md) | Modularchitektur & Package-Struktur | v0.5 |
| [Kap 2](architecture/02-realtime-pipeline.md) | Echtzeit-Datenpipeline | v0.4 |
| [Kap 3](architecture/03-broker-integration.md) | Broker-Anbindung (IB Gateway, TWS API 10.40) | v0.3 |
| [Kap 4](architecture/04-kpi-engine.md) | KPI-Engine (ta4j, Parallelisierung) | v0.3 |
| [Kap 5](architecture/05-llm-integration.md) | LLM-Integration (Claude SDK, OpenAI) | v0.3 |
| [Kap 6](architecture/06-rules-engine.md) | Rules Engine & Decision Loop | v0.3 |
| [Kap 7](architecture/07-oms.md) | Order Management System (OMS) | v0.3 |
| [Kap 8](architecture/08-data-model.md) | Datenmodell & Persistenz (PostgreSQL) | v0.6 |
| [Kap 10](architecture/10-deployment.md) | Deployment & Betrieb (Windows, On-Prem) | v0.6 |

## Guardrails

| Guardrail | Geltungsbereich | Version |
|-----------|----------------|---------|
| [Modulstruktur](guardrails/module-structure.md) | Java-Backend (8 Module: odin-api bis odin-app) | v1.2 |
| [Entwicklungsprozess](guardrails/development-process.md) | Java-Backend (Phasen, Checkpoints, Reviews) | v1.2 |
