# Inflowlytics – Import Service (Architecture & Specification)

> Architecture document for the **import-service** microservice in the Inflowlytics project.  
> The goal of this project is to learn production-grade patterns: microservices, DDD, Hexagonal Architecture, security, containerization, observability.

---

## 1. Project Context (Inflowlytics)

**Inflowlytics** is a microservices-based project built with:

- Spring Boot 3
- Microservices architecture + DDD
- Hexagonal Architecture (Ports & Adapters)
- API Gateway: Spring Cloud Gateway
- Authentication/Authorization: Keycloak (OAuth2/OIDC, JWT)
- Inter-service communication: REST (Kafka planned for the future)
- Asynchronous long-running tasks: Spring Batch
- Database per microservice: PostgreSQL
- Containerization: Docker
- Orchestration: Kubernetes (Helm)
- Observability: Actuator + OpenTelemetry + Jaeger
- CI/CD: GitHub Actions
- Frontend: React (MVP may start with low-code)

This is an **educational + production-like project** aimed at learning real-world architectural patterns.

---

## 2. Architectural Principles (DDD + Hexagonal)

### 2.1. Hexagonal Architecture (Ports & Adapters)

- The **application core (domain + application)** must not depend on any framework (no Spring).
- The outside world communicates with the core through **ports (interfaces)**.
- Technical integrations are implemented as **adapters**.

### 2.2. Spring Usage Rules

- ❌ No Spring annotations in `domain`
- ❌ No Spring annotations in `application`
- ✅ Adapters (`adapters/in`, `adapters/out`) may use Spring annotations
- ✅ `config/BeanConfiguration` is responsible for wiring ports to adapters

### 2.3. Use Cases

- ✅ **1 use case = 1 inbound port = 1 service (implementation)**
- Each use case has:
  - its own inbound port (interface)
  - its own service class (implementation)
- Examples:
  - `ImportFileUseCase` → `ImportFileService`
  - `GetImportStatusUseCase` → `GetImportStatusService`
  - `RetryImportUseCase` → `RetryImportService` (future)
  - `CancelImportUseCase` → `CancelImportService` (future)

### 2.4. Controllers (Inbound Adapters)

- ❌ Do NOT create 1 controller per use case
- ✅ **1 controller per resource / bounded context**
- For import-service:
  - one `ImportController`
  - multiple endpoints
  - each endpoint calls a different inbound port

### 2.5. Naming Conventions

- Inbound ports: `*UseCase`  
  e.g. `ImportFileUseCase`, `GetImportStatusUseCase`
- Implementations: `*Service`  
  e.g. `ImportFileService`, `GetImportStatusService`
- Outbound adapters: `JpaXxxRepository`, `KafkaXxxPublisher`, `LocalFileStorage`
- No `@Service` in `application` – only `BeanConfiguration`

---

## 3. Import Service – MVP Scope

MVP features:

- POST `/imports`
  - upload a file (CSV/XLSX)
  - store file (FileStorage)
  - create `ImportJob`
  - start async processing (Spring Batch)
  - return `jobId`

- GET `/imports/{id}/status`
  - returns import status: `PENDING | PROCESSING | COMPLETED | FAILED`

MVP constraints:

- ❌ Kafka – not used yet
- ❌ Multi-tenancy – planned for later
- ❌ AI – separate microservice in the future

---

## 4. Target Project Structure (DDD + Hexagonal)

```text
inflowlytics-import-service/
├── .github/
│   └── workflows/
│       └── ci.yml
├── Dockerfile
├── docker-compose.local.yml
├── README.md
├── build.gradle
└── src/
    ├── main/
    │   ├── java/com/inflowlytics/imports/
    │   │   ├── ImportServiceApplication.java      # Spring Boot main
    │   │   │
    │   │   ├── adapters/                          # HEXAGON: ADAPTERS
    │   │   │   ├── in/
    │   │   │   │   └── rest/
    │   │   │   │       └── ImportController.java  # 1 controller = Import resource
    │   │   │   │
    │   │   │   └── out/
    │   │   │       ├── persistence/
    │   │   │       │   ├── JpaImportJobRepository.java
    │   │   │       │   ├── JpaTransactionRepository.java
    │   │   │       │   └── entity/
    │   │   │       │       ├── ImportJobEntity.java
    │   │   │       │       └── TransactionEntity.java
    │   │   │       │
    │   │   │       ├── messaging/
    │   │   │       │   └── KafkaEventPublisher.java
    │   │   │       │
    │   │   │       ├── storage/
    │   │   │       │   ├── LocalFileStorage.java
    │   │   │       │   └── S3FileStorage.java       # future
    │   │   │       │
    │   │   │       └── batch/
    │   │   │           ├── ImportJobConfig.java     # Spring Batch Job/Step
    │   │   │           ├── CsvTransactionReader.java
    │   │   │           ├── TransactionProcessor.java
    │   │   │           └── TransactionWriter.java
    │   │   │
    │   │   ├── application/                        # HEXAGON: CORE (Use Cases)
    │   │   │   ├── ports/
    │   │   │   │   ├── in/
    │   │   │   │   │   ├── ImportFileUseCase.java
    │   │   │   │   │   ├── GetImportStatusUseCase.java
    │   │   │   │   │   ├── RetryImportUseCase.java        # future
    │   │   │   │   │   └── CancelImportUseCase.java       # future
    │   │   │   │   └── out/
    │   │   │   │       ├── ImportJobRepository.java
    │   │   │   │       ├── TransactionRepository.java
    │   │   │   │       ├── FileStorage.java
    │   │   │   │       └── EventPublisher.java
    │   │   │   │
    │   │   │   └── service/
    │   │   │       ├── ImportFileService.java
    │   │   │       ├── GetImportStatusService.java
    │   │   │       ├── RetryImportService.java      # future
    │   │   │       └── CancelImportService.java     # future
    │   │   │
    │   │   ├── domain/                              # DDD CORE
    │   │   │   ├── model/
    │   │   │   │   ├── ImportJob.java               # Aggregate Root
    │   │   │   │   ├── ImportStatus.java
    │   │   │   │   └── Transaction.java
    │   │   │   │
    │   │   │   └── events/
    │   │   │       ├── FileUploadedEvent.java
    │   │   │       └── FileParsedEvent.java
    │   │   │
    │   │   └── config/
    │   │       └── BeanConfiguration.java           # wiring ports ↔ adapters
    │   │
    │   └── resources/
    │       └── application.yml
    │
    └── test/
        └── java/com/inflowlytics/imports/
            ├── application/
            │   ├── ImportFileServiceTest.java
            │   ├── GetImportStatusServiceTest.java
            │   └── RetryImportServiceTest.java      # future
            │
            └── adapters/
                └── in/rest/
                    └── ImportControllerTest.java
5. Testing Strategy

Use case tests:

no Spring context

outbound ports mocked

Adapter tests:

REST controllers tested with MockMvc / WebTestClient

Core logic tested in isolation from frameworks

6. Roadmap (Import Service)

MVP

file upload

import status endpoint

batch processing (CSV → DB)

Next steps

link import status with Spring Batch (JobExecutionListener)

Kafka events (FileUploadedEvent, FileParsedEvent)

retry / cancel import job

S3 as FileStorage

multi-tenancy

business validations

OpenAPI contracts

7. Design Rules (Must-follow)

Controllers contain no business logic

Core does not depend on Spring

Adapters communicate with core only via ports

Each use case has its own service class

API is grouped by resources, not by use cases