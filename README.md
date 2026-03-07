Inflowlytics – Import Service (Architecture & Specification)

Architecture document for the import-service microservice in the Inflowlytics project.
The goal of this project is to learn production-grade patterns: microservices, DDD, Hexagonal Architecture, security, containerization, and observability.

1. Project Context (Inflowlytics)

Inflowlytics is a microservices-based financial data processing platform built with:

Java 21

Spring Boot 3

Microservices + DDD

Hexagonal Architecture (Ports & Adapters)

API Gateway: Spring Cloud Gateway

Authentication: Keycloak (OAuth2 / OIDC / JWT)

Database per microservice: PostgreSQL

Asynchronous processing: Spring Batch

Containerization: Docker

Orchestration: Kubernetes (Helm)

Observability: Actuator + OpenTelemetry + Jaeger

CI/CD: GitHub Actions

Frontend: React (MVP may start simple)

This is an educational + production-like project focused on architecture quality.

2. Role of Import Service

The import-service is responsible for:

Receiving financial report files (CSV/XLSX)

Storing files

Managing import jobs

Running asynchronous parsing and persistence

Reporting import status

Important architectural decision:

Import-service is process-oriented.
It manages the import workflow, not the financial domain.

It does NOT own:

categorization logic

analytics logic

financial domain rules

3. Architectural Principles
   3.1 Hexagonal Architecture (Ports & Adapters)

The application follows strict Ports & Adapters principles:

The core (domain + application) does NOT depend on Spring.

All technical integrations are implemented as adapters.

Dependencies always point inward.

Structure:

domain → business model (pure Java)

application → use cases + ports

adapters → REST, JPA, Batch, Storage

config → explicit wiring (BeanConfiguration)

3.2 Spring Usage Rules

❌ No Spring annotations in domain
❌ No Spring annotations in application
✅ Spring allowed only in adapters and config
✅ No @Service in application layer
✅ Beans wired explicitly in BeanConfiguration

3.3 Use Case Rule

Strict rule:

1 Use Case = 1 Inbound Port = 1 Service

MVP Use Cases:

ImportFileUseCase → ImportFileService

GetImportStatusUseCase → GetImportStatusService

4. Domain Model (Import Context Only)

Import-service domain is intentionally minimal.

Aggregate Root
ImportJob

Represents the lifecycle of a file import.

Responsibilities:

Track import status

Maintain job identity

Represent process state

ImportStatus

Enum representing:

PENDING

PROCESSING

COMPLETED

FAILED

Important Clarification

Transaction is NOT part of the domain model in import-service.

Reason:

Import-service does not manage financial business logic.

Transactions are results of processing.

Financial domain logic will belong to other services (e.g., categorization-service, analytics-service).

Therefore:

Transaction exists only as a persistence model (TransactionEntity)

It does not live in domain/

5. Spring Batch Integration

Spring Batch is treated as an infrastructure adapter.

It is responsible for:

Reading CSV

Processing rows

Writing transactions to DB

Updating import status

Batch components:

Reader

Processor

Writer

Job configuration

JobExecutionListener (for status updates)

Batch status must be mapped to ImportStatus.
Domain must not depend on Batch APIs.

6. Target Project Structure (Clean Version)
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
   │   │
   │   │   ├── ImportServiceApplication.java
   │   │
   │   │   ├── adapters/
   │   │   │   ├── in/
   │   │   │   │   └── rest/
   │   │   │   │       └── ImportController.java
   │   │   │   │
   │   │   │   └── out/
   │   │   │       ├── persistence/
   │   │   │       │   ├── JpaImportJobRepository.java
   │   │   │       │   ├── JpaTransactionRepository.java
   │   │   │       │   └── entity/
   │   │   │       │       ├── ImportJobEntity.java
   │   │   │       │       └── TransactionEntity.java
   │   │   │       │
   │   │   │       ├── storage/
   │   │   │       │   └── LocalFileStorage.java
   │   │   │       │
   │   │   │       └── batch/
   │   │   │           ├── ImportJobConfig.java
   │   │   │           ├── CsvTransactionReader.java
   │   │   │           ├── TransactionProcessor.java
   │   │   │           ├── TransactionWriter.java
   │   │   │           └── ImportJobExecutionListener.java
   │   │   │
   │   │   ├── application/
   │   │   │   ├── ports/
   │   │   │   │   ├── in/
   │   │   │   │   │   ├── ImportFileUseCase.java
   │   │   │   │   │   └── GetImportStatusUseCase.java
   │   │   │   │   │
   │   │   │   │   └── out/
   │   │   │   │       ├── ImportJobRepository.java
   │   │   │   │       ├── TransactionRepository.java
   │   │   │   │       ├── FileStorage.java
   │   │   │   │       └── ImportJobStarter.java
   │   │   │   │
   │   │   │   └── service/
   │   │   │       ├── ImportFileService.java
   │   │   │       └── GetImportStatusService.java
   │   │   │
   │   │   ├── domain/
   │   │   │   └── model/
   │   │   │       ├── ImportJob.java
   │   │   │       └── ImportStatus.java
   │   │   │
   │   │   └── config/
   │   │       └── BeanConfiguration.java
   │   │
   │   └── resources/
   │       └── application.yml
   │
   └── test/
   └── java/com/inflowlytics/imports/
   ├── application/
   │   ├── ImportFileServiceTest.java
   │   └── GetImportStatusServiceTest.java
   │
   └── adapters/
   └── in/rest/
   └── ImportControllerTest.java
7. Processing Flow
   Upload

POST /imports

File stored via FileStorage

ImportJob created (PENDING)

Batch job started

Status returned with jobId

Batch Execution

beforeJob → status = PROCESSING

CSV parsed in chunks

Transactions written to DB

afterJob:

COMPLETED

or FAILED

8. Testing Strategy

Use case tests:

No Spring context

Outbound ports mocked

Pure unit tests

Adapter tests:

REST via MockMvc / WebTestClient

Batch tested separately

Domain tests:

Pure unit tests

No framework dependencies

9. Roadmap

Next steps after MVP:

Retry / Cancel import

Kafka event publishing

S3 storage

Multi-tenancy

OpenAPI contracts

Improved validation

Observability improvements

10. Design Rules (Must Follow)

Controllers contain no business logic

Core does not depend on Spring

Adapters communicate with core only via ports

Each use case has its own service

Import-service manages process, not financial domain