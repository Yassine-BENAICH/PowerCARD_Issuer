# PowerCARD Issuer Platform

> Enterprise-grade card issuing and management platform built with Java 21, Spring Boot 3, JPOS ISO 8583, and a Vite/React dashboard.

---

## Architecture

```
PowerCARD_Issuer/
├── api-gateway/             ← Spring Cloud Gateway  :8080
├── card-service/            ← Card lifecycle        :8081
├── account-service/         ← Cardholder KYC        :8082
├── transaction-service/     ← Transaction ledger    :8083
├── authorization-service/   ← ISO 8583 via JPOS     :8084
├── notification-service/    ← Email/SMS alerts      :8085
├── common/                  ← Shared DTOs & utils
├── frontend/                ← Vite + React          :5173
└── docker-compose.yml
```

### Microservice Topology

```
Vite Frontend (5173)  →  API Gateway (8080)
                             ├── Card Service       (8081) ← PostgreSQL
                             ├── Account Service    (8082) ← PostgreSQL
                             ├── Transaction Service(8083) ← PostgreSQL
                             ├── Authorization Svc  (8084) ← PostgreSQL
                             │        ↕ ISO 8583 (JPOS)
                             │    Card Switch / Network
                             └── Notification Svc   (8085)
```

---

## Tech Stack

| Layer      | Technology                     |
|------------|--------------------------------|
| Language   | Java 21 LTS                    |
| Framework  | Spring Boot 3.2                |
| Gateway    | Spring Cloud Gateway           |
| ISO 8583   | JPOS 2.1.9                     |
| Database   | PostgreSQL 16                  |
| Migrations | Flyway 10                      |
| ORM        | Spring Data JPA + Hibernate    |
| Mapping    | MapStruct 1.5                  |
| Build      | Maven (multi-module)           |
| Frontend   | Vite 5 + React 18 + TypeScript |
| Charts     | Recharts                       |
| Container  | Docker + Docker Compose        |

---

## Design Patterns

| Pattern           | Location                                               |
|-------------------|--------------------------------------------------------|
| **API Gateway**   | `api-gateway/` — single entry point                    |
| **Repository**    | `*Repository.java` — data access abstraction           |
| **Service Layer** | `*Service.java` — business logic separation            |
| **Factory**       | `CardFactory.java` — card creation with Luhn           |
| **Strategy**      | `ResponseCodeMapper.java` — ISO 8583 response routing  |
| **Observer**      | `NotificationEventListener.java` — event-driven alerts |
| **DTO**           | `common/dto/` — clean API boundaries                   |
| **Builder**       | `@Builder` on all entities and responses               |

---

## Quick Start

### Prerequisites
- Java 21+
- Maven 3.9+
- Docker Desktop
- Node.js 20+

### 1. Start Databases

```bash
docker-compose up postgres-cards postgres-accounts postgres-transactions postgres-authorization postgres-notifications -d
```

### 2. Build Java Services

```bash
mvn clean install -DskipTests
```

### 3. Start Backend Services

```bash
# In separate terminals:
cd api-gateway           && mvn spring-boot:run
cd card-service          && mvn spring-boot:run
cd account-service       && mvn spring-boot:run
cd transaction-service   && mvn spring-boot:run
cd authorization-service && mvn spring-boot:run
cd notification-service  && mvn spring-boot:run
```

### 4. Start Frontend

```bash
cd frontend
npm install
npm run dev
# → http://localhost:5173
```

### 5. Full Docker Stack

```bash
docker-compose up --build
```

---

## REST API Reference

### Card Service (`/api/v1/cards`)

| Method | Endpoint                            | Description      |
|--------|-------------------------------------|------------------|
| `POST` | `/api/v1/cards`                     | Issue new card   |
| `GET`  | `/api/v1/cards/{id}`                | Get card details |
| `GET`  | `/api/v1/cards/account/{accountId}` | Cards by account |
| `PUT`  | `/api/v1/cards/{id}/block`          | Block card       |
| `PUT`  | `/api/v1/cards/{id}/unblock`        | Unblock card     |
| `PUT`  | `/api/v1/cards/{id}/cancel`         | Cancel card      |
| `POST` | `/api/v1/cards/{id}/renew`          | Renew card       |

### Account Service (`/api/v1/accounts`)

| Method | Endpoint                           | Description           |
|--------|------------------------------------|-----------------------|
| `POST` | `/api/v1/accounts`                 | Create account        |
| `GET`  | `/api/v1/accounts/{id}`            | Get account           |
| `GET`  | `/api/v1/accounts/number/{number}` | Get by account number |
| `PUT`  | `/api/v1/accounts/{id}/activate`   | KYC activate          |
| `PUT`  | `/api/v1/accounts/{id}/suspend`    | Suspend               |

### Authorization Service (`/api/v1/authorize`)

| Method | Endpoint            | Description                       |
|--------|---------------------|-----------------------------------|
| `POST` | `/api/v1/authorize` | ISO 8583 Authorization (MTI 0100) |

#### Example Authorization Request:
```json
{
  "pan": "4111111111111111",
  "expiryDate": "2803",
  "cvv": "123",
  "amount": 99.99,
  "currency": "USD",
  "terminalId": "TERM0001",
  "merchantId": "MERCH001",
  "merchantName": "Amazon",
  "transactionType": "00"
}
```

---

## Database Schema

Each service uses its own independent PostgreSQL database (Database-per-Service pattern):

| Service               | Database                  | Key Tables           |
|-----------------------|---------------------------|----------------------|
| card-service          | `powercard_cards`         | `cards`              |
| account-service       | `powercard_accounts`      | `accounts`           |
| transaction-service   | `powercard_transactions`  | `transactions`       |
| authorization-service | `powercard_authorization` | `authorization_logs` |
| notification-service  | `powercard_notifications` | `notification_logs`  |

---

## Security Notes

>  **Production Checklist:**
> - PAN and CVV must be **encrypted at rest** (AES-256 / HSM)
> - PIN blocks must be **encrypted** using 3DES/AES
> - Replace hardcoded credentials with **environment variables / Vault**
> - Enable **Spring Security** JWT authentication on API Gateway
> - Use **TLS 1.3** for all inter-service communication and JPOS channels

---

## Environment Variables

| Variable                | Description       | Default               |
|-------------------------|-------------------|-----------------------|
| `SPRING_DATASOURCE_URL` | Database JDBC URL | localhost             |
| `MAIL_USERNAME`         | SMTP username     | noreply@powercard.com |
| `MAIL_PASSWORD`         | SMTP password     | changeme              |
| `JPOS_HOST`             | Card switch host  | localhost             |
| `JPOS_PORT`             | Card switch port  | 10000                 |

---

## Dashboard Features

- **Overview Dashboard** — KPI cards, 6-month transaction volume chart, card network split (Pie chart), live authorization feed
- **Card Management** — Visual card widgets, block/unblock/renew, issue new card modal
- **Account Management** — KYC status tracking, account creation modal, activation workflow
- **Transaction History** — Full ledger with color-coded debits/credits
- **Authorization Simulator** — ISO 8583 MTI 0100 request builder with live MTI 0110 response display
