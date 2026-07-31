# java-spring-microservices
Java Microservices for Patient management system

A production-style microservices-based Patient Management System built with Java, Spring Boot, Docker, and AWS. This project follows a real-world enterprise architecture pattern — from local service development through containerization, event-driven communication, security, and cloud deployment.

## Architecture Overview
The system is composed of independently deployable Spring Boot microservices, each with its own database and responsibility, communicating via REST, gRPC, and Kafka, and fronted by a single API Gateway.



### Services

| Service            | Port | Responsibility                                              |
|---------------------|------|---------------------------------------------------------------|
| **API Gateway**      | 4004 | Single entry point; routes requests, validates JWTs via `auth-service`, strips path prefixes |
| **Auth Service**     | 4005 | User authentication, JWT issuance & validation (Spring Security) |
| **Patient Service**  | 4000 | Core patient CRUD operations; owns `patient-service-db` (PostgreSQL) |
| **Billing Service**  | 4001 (REST), 9001 (gRPC) | Billing account management; communicates with Patient Service via gRPC and Kafka |

---

## Tech Stack

- **Language / Framework:** Java 21, Spring Boot 4.x
- **API Gateway:** Spring Cloud Gateway (WebFlux)
- **Security:** Spring Security, JWT
- **Inter-service Communication:** REST, gRPC
- **Event Streaming:** Apache Kafka
- **Persistence:** PostgreSQL (per-service databases), Spring Data JPA / Hibernate
- **Containerization:** Docker
- **Cloud (local simulation):** LocalStack, AWS CloudFormation (IaC)
- **AWS Components Modeled:** VPC, RDS, ECS
- **Testing:** JUnit 5, REST-Assured (integration tests)

## Project Structure

```
patient-management/
├── api-gateway/           # Spring Cloud Gateway — routing & JWT validation
├── auth-service/          # Authentication & JWT issuance
├── patient-service/       # Core patient management service
├── billing-service/       # Billing service (REST + gRPC)
├── integration-tests/     # Cross-service REST-Assured integration tests
└── infrastructure/        # CloudFormation templates for AWS/LocalStack deployment
```

---

## Getting Started (Local Development)

### Prerequisites
- Java 21
- Maven
- Docker & Docker Desktop
- (Optional) AWS CLI + LocalStack, for the cloud deployment phase

Each service has its own Dockerfile and can be built and run independently. Example:

```bash
# Build a service
cd patient-service
mvn clean package
docker build -t patient-service .

# Run with a persistent volume for its database
docker network create pm-network

docker run -d --name patient-service-db \
  --network pm-network \
  -v patient-service-db-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=<db-name> \
  -e POSTGRES_USER=<user> \
  -e POSTGRES_PASSWORD=<password> \
  -p 5432:5432 \
  postgres:latest

docker run -d --name patient-service \
  --network pm-network \
  -p 4000:4000 \
  patient-service
```



> **Note on persistence:** Postgres containers must be run with a named volume (`-v <volume>:/var/lib/postgresql/data`), or all data is lost whenever the container is recreated.

> **Note on networking:** Services communicate using Docker container names (e.g. `http://auth-service:4005`). Containers must share a **user-defined Docker network** for name resolution to work — the default bridge network does not support this.

Repeat similarly for `auth-service`, `billing-service`, and `api-gateway`, ensuring all containers join the same Docker network.

### Environment Variables

| Variable            | Used By      | Example                          |
|----------------------|---------------|------------------------------------|
| `AUTH_SERVICE_URL`   | api-gateway   | `http://auth-service:4005`        |

### API Gateway Routes

| Path                     | Forwards To                          | Notes                          |
|----------------------------|----------------------------------------|----------------------------------|
| `/auth/**`                 | `auth-service:4005`                   | Prefix stripped                |
| `/api/patients/**`         | `patient-service:4000`                | Prefix stripped, JWT-protected |
| `/api-docs/patients`       | `patient-service:4000/v3/api-docs`    | Swagger/OpenAPI docs           |

Example request flow:

```http
POST http://localhost:4004/auth/login
Content-Type: application/json

{
  "email": "testuser@test.com",
  "password": "password123"
}
```

```http
GET http://localhost:4004/api/patients
Authorization: Bearer <token>
```

---

## Testing

Integration tests live in `integration-tests/` and use REST-Assured against the running Docker stack (via the API Gateway on port 4004).

```bash
cd integration-tests
mvn test
```

---

## Cloud Deployment (LocalStack + AWS)

The infrastructure is defined as code using **AWS CloudFormation**, provisioning:

- **VPC** — network isolation for all services
- **RDS** — managed PostgreSQL for `auth-service` and `patient-service`
- **ECS** — container orchestration for running all microservices

Deployment is simulated locally using **LocalStack** before targeting real AWS infrastructure.

```bash
# Example — adjust paths/stack names to match your CloudFormation templates
localstack start
aws --endpoint-url=http://localhost:4566 cloudformation deploy \
  --template-file infrastructure/<template>.yml \
  --stack-name patient-management-stack
```

---
