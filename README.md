# 🏦 Account Acquisition Service

> A production-grade microservice simulating a digital bank's customer acquisition pipeline — handling signup orchestration, KYC verification triggering, and onboarding flow management.

---

## Overview

This service forms the core of a customer acquisition domain in a modern fintech platform. It exposes a set of REST APIs that manage the full lifecycle of a new customer from initial registration through identity verification and account activation.

Designed with scalability and resilience in mind, the service follows a clean hexagonal architecture pattern and integrates with asynchronous messaging for downstream event propagation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer           │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│              Account Acquisition Service                 │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Signup API │  │  KYC Handler │  │Onboarding Flow│  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                │                   │          │
│  ┌──────▼────────────────▼───────────────────▼───────┐  │
│  │              Domain Service Layer                  │  │
│  └──────────────────────┬────────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐  │
│  │           Repository / Persistence Layer           │  │
│  └──────────────────────┬────────────────────────────┘  │
└─────────────────────────┼───────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
     ┌────▼────┐   ┌──────▼──────┐  ┌────▼────┐
     │Postgres │   │  AWS SQS    │  │  Redis  │
     │  (RDS)  │   │(Event Queue)│  │ (Cache) │
     └─────────┘   └─────────────┘  └─────────┘
```

---

## Features

- **Customer Registration** — Validates and persists new customer signup data with duplicate detection
- **KYC Orchestration** — Triggers identity verification workflows via async SQS events
- **Onboarding State Machine** — Tracks each customer through a defined set of onboarding stages
- **Idempotent API Design** — Safe to retry without duplicate side effects
- **Event Publishing** — Emits domain events on state transitions for downstream consumers
- **Rate Limiting** — Per-IP and per-email request throttling to prevent abuse
- **Audit Logging** — Full audit trail for every state change (compliance-ready)

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Kotlin 1.9 |
| Framework | Spring Boot 3.2 |
| Database | PostgreSQL 15 (via Spring Data JPA) |
| Messaging | AWS SQS (LocalStack for local dev) |
| Caching | Redis |
| Containerization | Docker + Docker Compose |
| Build Tool | Gradle (Kotlin DSL) |
| Testing | JUnit 5, MockK, Testcontainers |
| Documentation | SpringDoc OpenAPI 3 |

---

## Project Structure

```
n26-account-acquisition-service/
│
├── src/
│   ├── main/
│   │   ├── kotlin/com/n26/acquisition/
│   │   │   ├── AcquisitionServiceApplication.kt
│   │   │   ├── api/
│   │   │   │   ├── CustomerRegistrationController.kt
│   │   │   │   ├── OnboardingController.kt
│   │   │   │   └── dto/
│   │   │   │       ├── RegistrationRequest.kt
│   │   │   │       ├── RegistrationResponse.kt
│   │   │   │       └── OnboardingStatusResponse.kt
│   │   │   ├── domain/
│   │   │   │   ├── model/
│   │   │   │   │   ├── Customer.kt
│   │   │   │   │   ├── OnboardingStatus.kt
│   │   │   │   │   └── KycStatus.kt
│   │   │   │   ├── service/
│   │   │   │   │   ├── CustomerRegistrationService.kt
│   │   │   │   │   ├── KycOrchestrationService.kt
│   │   │   │   │   └── OnboardingStateMachine.kt
│   │   │   │   └── port/
│   │   │   │       ├── CustomerRepository.kt
│   │   │   │       └── EventPublisher.kt
│   │   │   ├── infrastructure/
│   │   │   │   ├── persistence/
│   │   │   │   │   ├── CustomerJpaRepository.kt
│   │   │   │   │   └── CustomerEntity.kt
│   │   │   │   ├── messaging/
│   │   │   │   │   └── SqsEventPublisher.kt
│   │   │   │   └── config/
│   │   │   │       ├── AwsConfig.kt
│   │   │   │       ├── RedisConfig.kt
│   │   │   │       └── SecurityConfig.kt
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-local.yml
│   │       └── db/migration/
│   │           ├── V1__create_customers_table.sql
│   │           └── V2__create_onboarding_events_table.sql
│   └── test/
│       └── kotlin/com/n26/acquisition/
│           ├── api/
│           │   └── CustomerRegistrationControllerTest.kt
│           ├── domain/
│           │   └── CustomerRegistrationServiceTest.kt
│           └── integration/
│               └── CustomerRegistrationIntegrationTest.kt
│
├── docker-compose.yml
├── build.gradle.kts
├── settings.gradle.kts
├── .gitignore
└── README.md
```

---

## API Endpoints

### POST `/api/v1/customers/register`
Register a new customer account.

**Request:**
```json
{
  "email": "jane.doe@example.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "phoneNumber": "+49123456789",
  "dateOfBirth": "1990-05-14",
  "nationality": "DE",
  "referralCode": "N26REF123"
}
```

**Response `201 Created`:**
```json
{
  "customerId": "cust_01HXYZ9ABC",
  "status": "PENDING_KYC",
  "kycRedirectUrl": "https://kyc.provider.com/verify?token=abc123",
  "createdAt": "2024-11-01T10:22:00Z"
}
```

---

### GET `/api/v1/customers/{customerId}/onboarding-status`
Retrieve current onboarding progress.

**Response `200 OK`:**
```json
{
  "customerId": "cust_01HXYZ9ABC",
  "currentStage": "KYC_IN_PROGRESS",
  "completedStages": ["REGISTRATION", "EMAIL_VERIFIED"],
  "pendingStages": ["KYC_APPROVED", "ACCOUNT_ACTIVATED"],
  "lastUpdated": "2024-11-01T10:45:00Z"
}
```

---

### POST `/api/v1/customers/{customerId}/kyc-webhook`
Webhook endpoint to receive KYC provider callbacks.

---

## Onboarding State Machine

```
REGISTERED → EMAIL_PENDING_VERIFICATION
     ↓
EMAIL_VERIFIED → KYC_PENDING
     ↓
KYC_IN_PROGRESS → KYC_APPROVED / KYC_REJECTED
     ↓
KYC_APPROVED → ACCOUNT_ACTIVATION_PENDING
     ↓
ACCOUNT_ACTIVATED ✅
```

---

## Getting Started

### Prerequisites
- JDK 17+
- Docker & Docker Compose
- AWS CLI (for LocalStack setup)

### Run locally

```bash
# Clone the repo
git clone https://github.com/emmaoluga-sketch/n26-account-acquisition-service.git
cd n26-account-acquisition-service

# Start infrastructure (Postgres, Redis, LocalStack/SQS)
docker-compose up -d

# Run the application
./gradlew bootRun --args='--spring.profiles.active=local'
```

### Run tests

```bash
# Unit tests
./gradlew test

# Integration tests (requires Docker)
./gradlew integrationTest
```

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_URL` | PostgreSQL JDBC URL | `jdbc:postgresql://localhost:5432/acquisition` |
| `DB_USERNAME` | Database username | `postgres` |
| `DB_PASSWORD` | Database password | — |
| `REDIS_HOST` | Redis host | `localhost` |
| `AWS_SQS_ENDPOINT` | SQS endpoint (LocalStack) | `http://localhost:4566` |
| `KYC_WEBHOOK_SECRET` | HMAC secret for webhook validation | — |

---

## Key Design Decisions

**Hexagonal Architecture** — The domain layer has zero dependency on infrastructure. Ports and adapters keep the core logic fully testable in isolation.

**Idempotency Keys** — Registration requests include idempotency keys to safely handle retries without creating duplicate accounts — critical in high-traffic signup flows.

**Async KYC Triggering** — KYC initiation is decoupled via SQS. If the KYC provider is unavailable, the message is retried automatically without blocking the signup response.

**Flyway Migrations** — All schema changes are version-controlled via Flyway, ensuring consistent database state across all environments.

---

## Future Improvements

- [ ] Add OpenTelemetry distributed tracing
- [ ] Implement A/B test flag support for signup flow variants
- [ ] Add dead-letter queue (DLQ) monitoring dashboard
- [ ] Expand KYC provider abstraction to support multiple vendors

---

## License

MIT
