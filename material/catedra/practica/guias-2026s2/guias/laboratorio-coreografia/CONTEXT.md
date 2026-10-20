# Project Context: Choreography Saga Pattern

This document provides a high-level technical context for AI agents working on this repository.

## 🎯 Project Overview
This project is a laboratory implementation of the **Choreographed Saga Pattern** for distributed transactions. It simulates an e-commerce flow where multiple microservices coordinate via events to maintain eventual consistency without a central orchestrator.

- **Objective**: Demonstrate event-driven coordination and compensation logic.
- **Domain**: E-commerce (Sales, Inventory, Payments).
- **Language**: Spanish (contextual/comments), English (code/structure).

## 🛠 Tech Stack
- **Language**: Java 17
- **Framework**: Spring Boot 3.x
- **Messaging**: Apache Kafka (Confluent 7.5.0)
- **Database**: MySQL (Single instance, multiple schemas)
- **Containerization**: Docker & Docker Compose
- **Build Tool**: Maven (Multi-module)

## 🏗 Architecture
### Hexagonal Architecture (Ports & Adapters)
Each microservice follows a strict hexagonal structure to decouple business logic from infrastructure:
- `application.core`: Domain entities and Use Cases (Input Port implementations).
- `application.ports`: Interfaces for Input (driven) and Output (driving) operations.
- `adapters`: Implementation of ports (REST controllers, Kafka consumers/producers, JPA repositories).
- `configuration`: Spring wiring (explicit `@Bean` instantiation for use cases).

### Distributed Transaction (Saga)
- **Pattern**: Choreography (Decentralized).
- **Topic**: Single shared topic `tp-saga-sale`.
- **Event Flow**:
  1. `microservice-sale`: Receives REST request -> Publishes `CREATED_SALE`.
  2. `microservice-inventory`: Consumes `CREATED_SALE` -> Reserves stock -> Publishes `UPDATED_INVENTORY` (or `ROLLBACK_INVENTORY` if out of stock).
  3. `microservice-payment`: Consumes `UPDATED_INVENTORY` -> Deducts balance -> Publishes `VALIDATED_PAYMENT` (or `FAILED_PAYMENT` if insufficient funds).
  4. `microservice-sale`: Consumes `VALIDATED_PAYMENT` -> Finalizes sale.
- **Compensations**:
  - If payment fails, `microservice-inventory` consumes `FAILED_PAYMENT` to restock and publishes `ROLLBACK_INVENTORY`.
  - `microservice-sale` consumes `ROLLBACK_INVENTORY` to mark the sale as `CANCELED`.

## 📦 Components
| Service | Role | Data Store |
|---------|------|------------|
| [microservice-sale](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/microservice-sale) | Saga Initiator, REST Entry point | `saga_sale` |
| [microservice-inventory](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/microservice-inventory) | Stock management | `saga_inventory` |
| [microservice-payment](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/microservice-payment) | User balance & transaction history | `saga_payment` |

## 🚀 Getting Started
### Infrastructure
Start the environment (Kafka, MySQL, Services):
```bash
docker compose up --build
```

### Common Commands
- **Build all**: `mvn clean package -DskipTests`
- **Test Success**: `curl -X POST 'http://localhost:8081/api/sales' ...` (See [README.md](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/README.md) for details)
- **Monitoring**:
  - **Kafka UI**: [http://localhost:8090](http://localhost:8090)
  - **MySQL**: `localhost:3306` (root/example)

## 📖 Key Documentation
- [README.md](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/README.md): General instructions and pattern definition.
- [CLAUDE.md](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/CLAUDE.md): Detailed AI instructions, package structure, and event routing table.
- [INTELLIJ_SETUP.md](file:///home/comunicarlos/Documentos/itba/microservicios/repositorios/laboratorio-coreografia/INTELLIJ_SETUP.md): IDE configuration guide.
