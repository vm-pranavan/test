## Ride-Hailing Microservices Project

This repository contains a small ride-hailing microservices example with the following services:

- `rider-service` (port 3001)
- `driver-service` (port 3002)
- `trip-service` (port 3003)
- `payment-service` (port 3004)

Each service is a Node.js/Express app that uses MongoDB for persistence. The project includes a `docker-compose.yml` that spins up one MongoDB instance per service and each Node service.

## High-level architecture

- Each service exposes a REST API under `/v1/<resource>` (e.g. `/v1/riders`, `/v1/drivers`, `/v1/trips`, `/v1/payments`).
- Services communicate via HTTP (internal service-to-service calls implemented with `axios`).
- Each service has its own MongoDB database (separate containers in docker-compose): `mongodb-rider`, `mongodb-driver`, `mongodb-trip`, `mongodb-payment`.
- All services implement a lightweight request correlation header `X-Correlation-Id` and structured logs with Winston.

## Architecture Diagrams

### System Architecture Overview

```mermaid
graph TB
    subgraph "Client Layer"
        Client[API Clients<br/>Web/Mobile/Third-party]
    end
    
    subgraph "Application Layer - Microservices"
        subgraph "Rider Service :3001"
            RS[Rider API<br/>Express.js]
            RSL[Winston Logger]
        end
        
        subgraph "Driver Service :3002"
            DS[Driver API<br/>Express.js]
            DSL[Winston Logger]
        end
        
        subgraph "Trip Service :3003"
            TS[Trip API<br/>Express.js]
            TSL[Winston Logger]
        end
        
        subgraph "Payment Service :3004"
            PS[Payment API<br/>Express.js]
            PSL[Winston Logger]
        end
    end
    
    subgraph "Data Layer - MongoDB Databases"
        RDB[(rider_db<br/>MongoDB)]
        DDB[(driver_db<br/>MongoDB)]
        TDB[(trip_db<br/>MongoDB)]
        PDB[(payment_db<br/>MongoDB)]
    end
    
    subgraph "Monitoring Layer"
        ME1[Mongo Express<br/>:8081]
        ME2[Mongo Express<br/>:8082]
        ME3[Mongo Express<br/>:8083]
        ME4[Mongo Express<br/>:8084]
    end
    
    Client -->|HTTP REST| RS
    Client -->|HTTP REST| DS
    Client -->|HTTP REST| TS
    Client -->|HTTP REST| PS
    
    RS -->|CRUD| RDB
    DS -->|CRUD| DDB
    TS -->|CRUD| TDB
    PS -->|CRUD| PDB
    
    RS -.->|Log| RSL
    DS -.->|Log| DSL
    TS -.->|Log| TSL
    PS -.->|Log| PSL
    
    ME1 -.->|Monitor| RDB
    ME2 -.->|Monitor| DDB
    ME3 -.->|Monitor| TDB
    ME4 -.->|Monitor| PDB
    
    TS -->|Validate| RS
    TS -->|Assign/Update| DS
    TS -->|Charge| PS
    PS -->|Verify| TS
    PS -->|Validate| RS
    
    style RDB fill:#4CAF50
    style DDB fill:#2196F3
    style TDB fill:#FF9800
    style PDB fill:#9C27B0
    style Client fill:#607D8B
```

### Service Intercommunication Flow

```mermaid
graph LR
    subgraph "External"
        API[API Consumer]
    end
    
    subgraph "Rider Service"
        R[Rider API<br/>:3001]
        RDB[(rider_db)]
    end
    
    subgraph "Driver Service"
        D[Driver API<br/>:3002]
        DDB[(driver_db)]
    end
    
    subgraph "Trip Service"
        T[Trip API<br/>:3003]
        TDB[(trip_db)]
    end
    
    subgraph "Payment Service"
        P[Payment API<br/>:3004]
        PDB[(payment_db)]
    end
    
    API -->|1. Create Rider| R
    API -->|2. Create Driver| D
    API -->|3. Request Trip| T
    API -->|4. Accept Trip| T
    API -->|5. Complete Trip| T
    
    R <-->|Own Data| RDB
    D <-->|Own Data| DDB
    T <-->|Own Data| TDB
    P <-->|Own Data| PDB
    
    T -->|GET /v1/riders/:id<br/>Validate Rider| R
    T -->|GET /v1/drivers/active<br/>Assign Driver| D
    T -->|PATCH /v1/drivers/:id/status<br/>Update Availability| D
    T -->|POST /v1/payments/charge<br/>Process Payment| P
    
    P -->|GET /v1/trips/:id<br/>Verify Completed| T
    P -->|GET /v1/riders/:id<br/>Validate Rider| R
    
    style R fill:#4CAF50
    style D fill:#2196F3
    style T fill:#FF9800
    style P fill:#9C27B0
```

### Trip Lifecycle Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant TripService
    participant RiderService
    participant DriverService
    participant PaymentService
    
    Note over Client,PaymentService: Complete Trip Flow
    
    Client->>TripService: POST /v1/trips<br/>{rider_id, coordinates}
    TripService->>RiderService: GET /v1/riders/:id
    RiderService-->>TripService: Rider Details
    TripService->>DriverService: GET /v1/drivers/active
    DriverService-->>TripService: Active Drivers List
    TripService->>TripService: Calculate fare & assign driver
    TripService-->>Client: Trip Created (REQUESTED)
    
    Client->>TripService: POST /v1/trips/:id/accept<br/>{driver_id}
    TripService->>DriverService: GET /v1/drivers/:id
    DriverService-->>TripService: Driver Details
    TripService->>DriverService: PATCH /v1/drivers/:id/status<br/>{is_active: false}
    DriverService-->>TripService: Driver Updated
    TripService-->>Client: Trip Accepted
    
    Client->>TripService: POST /v1/trips/:id/complete
    TripService->>TripService: Update status to COMPLETED
    TripService->>PaymentService: POST /v1/payments/charge<br/>{trip_id, rider_id, amount}
    PaymentService->>TripService: GET /v1/trips/:id
    TripService-->>PaymentService: Trip Details (COMPLETED)
    PaymentService->>RiderService: GET /v1/riders/:id
    RiderService-->>PaymentService: Rider Details
    PaymentService->>PaymentService: Process Payment & Generate Receipt
    PaymentService-->>TripService: Payment Result
    TripService->>DriverService: PATCH /v1/drivers/:id/status<br/>{is_active: true}
    DriverService-->>TripService: Driver Reactivated
    TripService-->>Client: Trip Completed + Payment Details
```

### Data Flow Architecture

```mermaid
flowchart TD
    Start([Client Request]) --> Gateway{Which Service?}
    
    Gateway -->|Rider Operations| RiderFlow[Rider Service]
    Gateway -->|Driver Operations| DriverFlow[Driver Service]
    Gateway -->|Trip Operations| TripFlow[Trip Service]
    Gateway -->|Payment Operations| PaymentFlow[Payment Service]
    
    RiderFlow --> RiderDB[(rider_db<br/>riders<br/>payment_instruments)]
    DriverFlow --> DriverDB[(driver_db<br/>drivers<br/>vehicles)]
    TripFlow --> TripDB[(trip_db<br/>trips)]
    PaymentFlow --> PaymentDB[(payment_db<br/>payments<br/>receipts)]
    
    TripFlow -.->|Validate| RiderFlow
    TripFlow -.->|Assign/Update| DriverFlow
    TripFlow -.->|Charge| PaymentFlow
    
    PaymentFlow -.->|Verify| TripFlow
    PaymentFlow -.->|Validate| RiderFlow
    
    RiderDB --> Response([JSON Response + X-Correlation-Id])
    DriverDB --> Response
    TripDB --> Response
    PaymentDB --> Response
    
    Response --> Logging[Winston Logs]
    
    style RiderDB fill:#4CAF50
    style DriverDB fill:#2196F3
    style TripDB fill:#FF9800
    style PaymentDB fill:#9C27B0
```

### Deployment Architecture (Docker Compose)

```mermaid
graph TB
    subgraph "Docker Network: ride-sharing-network"
        subgraph "Service Containers"
            RS[rider-service<br/>Port: 3001<br/>Image: Node 18 Alpine]
            DS[driver-service<br/>Port: 3002<br/>Image: Node 18 Alpine]
            TS[trip-service<br/>Port: 3003<br/>Image: Node 18 Alpine]
            PS[payment-service<br/>Port: 3004<br/>Image: Node 18 Alpine]
        end
        
        subgraph "Database Containers"
            MR[mongodb-rider<br/>Port: 27017<br/>Image: mongo:6.0]
            MD[mongodb-driver<br/>Port: 27017<br/>Image: mongo:6.0]
            MT[mongodb-trip<br/>Port: 27017<br/>Image: mongo:6.0]
            MP[mongodb-payment<br/>Port: 27017<br/>Image: mongo:6.0]
        end
        
        subgraph "Monitoring Containers"
            MER[mongo-express-rider<br/>Port: 8081]
            MED[mongo-express-driver<br/>Port: 8082]
            MET[mongo-express-trip<br/>Port: 8083]
            MEP[mongo-express-payment<br/>Port: 8084]
        end
        
        RS --> MR
        DS --> MD
        TS --> MT
        PS --> MP
        
        MER -.-> MR
        MED -.-> MD
        MET -.-> MT
        MEP -.-> MP
        
        TS -.->|HTTP| RS
        TS -.->|HTTP| DS
        TS -.->|HTTP| PS
        PS -.->|HTTP| TS
        PS -.->|HTTP| RS
    end
    
    subgraph "Host Machine"
        Host[localhost]
    end
    
    Host -->|:3001| RS
    Host -->|:3002| DS
    Host -->|:3003| TS
    Host -->|:3004| PS
    Host -->|:8081-8084| MER
    
    subgraph "Persistent Volumes"
        VR[mongodb_rider_data]
        VD[mongodb_driver_data]
        VT[mongodb_trip_data]
        VP[mongodb_payment_data]
    end
    
    MR -.-> VR
    MD -.-> VD
    MT -.-> VT
    MP -.-> VP
```

### Key Architecture Principles

1. **Database per Service**: Each microservice has its own MongoDB instance
2. **Service Independence**: Services can be deployed, scaled, and updated independently
3. **Loose Coupling**: Services communicate via REST APIs (HTTP)
4. **Eventual Consistency**: No distributed transactions; services validate via API calls
5. **Correlation Tracking**: X-Correlation-Id header propagates through all service calls
6. **Health Monitoring**: Each service exposes /health endpoint
7. **Containerization**: All services and databases run in Docker containers
8. **Idempotency**: Payment service supports idempotent operations

## Common patterns & cross-cutting concerns

- Health endpoint: `GET /health` returns service status.
- Logging: each service uses Winston and stores logs in a `logs/` folder inside the container.
- DB initialization: each service has `config/database.js` that initializes indexes and exposes `init()`, `getDb()`, `getClient()`.
- Correlation ID: incoming requests get `X-Correlation-Id` (generated if missing) and added to logs and outbound requests.
- Docker: each service includes a `Dockerfile` (Node 18 Alpine, installs production deps and exposes service port).

## Service details (endpoints, ports, env variables)

Driver Service
- Location: `driver-service/driver-service`
- Port: 3002 (default)
- Base route: `/v1/drivers`
- Health: `GET /health`
- Endpoints:
  - `GET /v1/drivers` — list drivers
  - `GET /v1/drivers/active` — list active drivers
  - `GET /v1/drivers/:id` — get driver by id
  - `POST /v1/drivers` — create driver (payload: name, email, license_number, optional vehicle data)
  - `PUT /v1/drivers/:id` — update driver
  - `PATCH /v1/drivers/:id/status` — update driver active status (body: { is_active: boolean })
  - `DELETE /v1/drivers/:id` — delete driver
- Env variables (used): `PORT`, `DB_HOST`, `DB_PORT`, `DB_NAME` (see docker-compose which sets MONGO_URL for containers)

Rider Service
- Location: `rider-service/rider-service`
- Port: 3001 (default)
- Base route: `/v1/riders`
- Health: `GET /health`
- Endpoints:
  - `GET /v1/riders` — list riders
  - `GET /v1/riders/:id` — get rider by id
  - `POST /v1/riders` — create rider (payload: name, email, phone)
  - `PUT /v1/riders/:id` — update rider
  - `DELETE /v1/riders/:id` — delete rider
- Env variables: same pattern as other services (`PORT`, `DB_HOST`, ...)

Trip Service
- Location: `trip-service/trip-service`
- Port: 3003 (default)
- Base route: `/v1/trips`
- Health: `GET /health`
- Endpoints:
  - `GET /v1/trips` — list trips
  - `GET /v1/trips/:id` — get trip by id
  - `POST /v1/trips` — create trip (payload requires rider_id and pickup/drop coordinates)
  - `POST /v1/trips/:id/accept` — accept trip (body: { driver_id })
  - `POST /v1/trips/:id/complete` — complete trip (triggers payment flow)
  - `POST /v1/trips/:id/cancel` — cancel trip
  - `PUT /v1/trips/:id` — update certain trip fields
- Integration: calls `driver-service`, `rider-service`, and `payment-service` using `services` config + `services/externalService.js`.

Payment Service
- Location: `payment-service/payment-service`
- Port: 3004 (default)
- Base route: `/v1/payments`
- Health: `GET /health`
- Endpoints:
  - `GET /v1/payments` — list payments
  - `GET /v1/payments/:id` — get payment by id
  - `POST /v1/payments/charge` — process a payment (payload: trip_id, rider_id, amount, currency, idempotency_key)
  - `POST /v1/payments/:id/refund` — refund a payment
  - `GET /v1/payments/trip/:tripId` — payments for a trip
- Important behaviors:
  - Idempotency: `chargePayment` supports `idempotency_key` to prevent duplicate charges (unique index on `idempotency_key`).
  - External calls: payment verifies trip via Trip Service and rider via Rider Service before charging.

## Docker & docker-compose

- Use the repo root `docker-compose.yml`. It defines separate MongoDB containers and Node services. Each Node service's container exposes the port mapped to the same host port (3001-3004).
- Example: `driver-service` in compose builds from `./driver-service` and maps `3002:3002`.

To start everything with Docker (recommended for a full environment):

1. From the repository root:

```powershell
docker-compose up --build
```

2. Health endpoints:

- Rider: http://localhost:3001/health
- Driver: http://localhost:3002/health
- Trip: http://localhost:3003/health
- Payment: http://localhost:3004/health

## How to run a single service locally (example: rider)

- cd into the service folder, install deps, and start:

```powershell
cd rider-service/rider-service
npm install
npm run start
```

Notes: each service expects a MongoDB available. You can either run the corresponding mongo container from docker-compose (recommended) or point the service to an external MongoDB by setting env vars.

## Observations, assumptions & next steps

- The services are intentionally small and synchronous (HTTP-based) for clarity. In production you may want to:
  - Add authentication/authorization.
  - Use service discovery or a gateway (instead of hard-coded service URLs in `config/services.js`).
  - Add retries/backoff and circuit breakers for inter-service HTTP calls.
  - Add proper API input validation and typed request/response shapes.
  - Add tests (unit/integration) and CI.

## Where to find code

- `*/index.js` — app bootstrap
- `*/routes/*` — route definitions
- `*/controllers/*` — implementation of endpoints
- `*/config/*` — DB and service URLs
- `*/services/*` — external service calls (used by trip/payment services)

---

If you want, I can: add full README.md files in each service folder, produce a Postman collection for the APIs, or generate OpenAPI specs for each service. Which would you prefer next?
