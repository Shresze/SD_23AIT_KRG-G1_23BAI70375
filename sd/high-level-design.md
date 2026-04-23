# High-Level Design (HLD)
## Real-Time Food Delivery Tracking System

**Student:** Shreshta | **UID:** 23BAI70375  
**Subject:** System Design (23CST-390) | **Semester:** 6th

---

## 1. Architecture Overview

The system follows a **microservices + event-driven architecture**, chosen for the following reasons:

| Architecture Decision | Justification |
|----------------------|---------------|
| **Microservices** over Monolith | Independent scaling — the Tracking Service needs 10x more instances than the Payment Service during peak hours. Independent deployments reduce blast radius of failures. |
| **Event-Driven** (Kafka) over Synchronous | Decouples producers from consumers. Location updates (100K/sec) would overwhelm synchronous REST calls. Kafka provides durability — events are persisted even if consumers are temporarily down. |
| **API Gateway** as single entry point | Centralizes cross-cutting concerns (auth, rate limiting, SSL termination). Clients need only one endpoint, simplifying mobile app networking. |

---

## 2. System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                        │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐               │
│  │  Customer App    │  │ Delivery Agent   │  │  Admin Dashboard  │               │
│  │  (iOS/Android)   │  │  App (Android)   │  │  (React Web App)  │               │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬──────────┘               │
└───────────┼──────────────────────┼──────────────────────┼────────────────────────┘
            │                      │                      │
            ▼                      ▼                      ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           EDGE LAYER                                             │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │                    Load Balancer (AWS ALB / NGINX)                        │    │
│  │           Round-Robin + Health Checks + SSL Termination                   │    │
│  └────────────────────────────────┬─────────────────────────────────────────┘    │
│                                   │                                              │
│  ┌────────────────────────────────▼─────────────────────────────────────────┐    │
│  │                    API Gateway (Kong / NGINX)                             │    │
│  │        Rate Limiting │ Authentication │ Request Routing │ Logging         │    │
│  └───┬──────────┬──────────┬──────────┬──────────┬──────────┬───────────────┘    │
└──────┼──────────┼──────────┼──────────┼──────────┼──────────┼────────────────────┘
       │          │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼          ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       CORE MICROSERVICES LAYER                                   │
│                                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐│
│  │  Auth    │ │  Order   │ │ Tracking  │ │ Delivery │ │   ETA    │ │ Payment ││
│  │ Service  │ │ Service  │ │ Service   │ │ Mgmt Svc │ │ Service  │ │ Service ││
│  │          │ │          │ │ (Critical)│ │          │ │          │ │         ││
│  └────┬─────┘ └────┬─────┘ └─────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬────┘│
│       │            │             │             │            │            │      │
│  ┌────▼────────────▼─────────────▼─────────────▼────────────▼────────────▼────┐ │
│  │                     Notification Service                                    │ │
│  │              (Push Notifications + SMS + Email)                              │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬─────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       EVENT STREAMING LAYER (Kafka)                              │
│                                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐               │
│  │ Location Updates  │  │  Order Events    │  │ Delivery Events  │               │
│  │     Topic         │  │     Topic        │  │     Topic        │               │
│  │ (Partitioned by   │  │ (Partitioned by  │  │ (Partitioned by  │               │
│  │   agent_id)       │  │   order_id)      │  │   region)        │               │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘               │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │                    Dead Letter Queue (DLQ)                                │    │
│  │           Failed events routed here for investigation                     │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────┬─────────────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER                                              │
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐│
│  │  PostgreSQL   │  │   MongoDB    │  │    Redis      │  │  Elasticsearch      ││
│  │              │  │              │  │   Cluster     │  │  (Search & Analytics)││
│  │ • Orders DB  │  │ • Tracking   │  │              │  │                      ││
│  │ • Users DB   │  │   DB (Geo-   │  │ • Latest     │  │ • Restaurant search  ││
│  │ • Payments   │  │   Spatial)   │  │   location   │  │ • Menu search        ││
│  │              │  │ • 2dsphere   │  │ • ETA cache  │  │ • Order analytics    ││
│  │  ACID for    │  │   indexes    │  │ • Session    │  │                      ││
│  │  financial   │  │              │  │   store      │  │                      ││
│  │  data        │  │  Eventual    │  │              │  │                      ││
│  │              │  │  consistency │  │ Sub-ms reads │  │                      ││
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL INTEGRATIONS                                         │
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐│
│  │  Maps &      │  │  Payment     │  │  SMS Gateway │  │  Push Notification   ││
│  │  Routing API │  │  Gateway     │  │  (Twilio)    │  │  Service (FCM/APNs)  ││
│  │  (Google     │  │  (Razorpay/  │  │              │  │                      ││
│  │   Maps)      │  │   Stripe)    │  │              │  │                      ││
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Deep Dive

### 3.1 Edge Layer

#### Load Balancer (AWS ALB / NGINX)
```
                    ┌─────────────────┐
                    │  Load Balancer   │
                    │                  │
                    │  • Round-robin   │
                    │  • Least-conn    │
                    │  • Health checks │
                    │  • SSL terminate │
                    └───┬────┬────┬────┘
                        │    │    │
                   ┌────▼┐ ┌▼───┐▼────┐
                   │ GW1 │ │GW2 │ GW3 │
                   └─────┘ └────┘└────┘
```

**Why ALB/NGINX?**
- **Layer 7 routing**: Can route `/tracking/*` to Tracking Service, `/orders/*` to Order Service
- **Health checks**: Automatically removes unhealthy instances from the pool
- **SSL termination**: Offloads TLS handshake from microservices, reducing CPU overhead on application pods

#### API Gateway (Kong / NGINX)
| Responsibility | Implementation | Why |
|---------------|----------------|-----|
| **Authentication** | JWT validation middleware | Validates tokens without hitting Auth Service for every request |
| **Rate Limiting** | Token bucket algorithm | Prevents abuse; limits to 100 req/sec per user |
| **Request Routing** | Path-based routing | `/api/v1/orders/*` → Order Service |
| **Logging** | Structured JSON logs → ELK | Centralized observability |
| **CORS** | Whitelist client origins | Security for web dashboard |

---

### 3.2 Core Microservices

#### Service Communication Pattern
```
┌───────────┐  Sync (REST)  ┌───────────┐
│   Order    │─────────────→│  Payment   │   ← Synchronous because we need
│  Service   │              │  Service   │     immediate payment confirmation
└─────┬──────┘              └────────────┘
      │
      │ Async (Kafka)
      ▼
┌───────────┐  Async (Kafka) ┌───────────┐
│ Delivery   │──────────────→│   ETA     │   ← Asynchronous because ETA
│ Mgmt Svc   │               │  Service  │     can be eventually consistent
└─────┬──────┘               └───────────┘
      │
      │ Async (Kafka)
      ▼
┌───────────┐
│Notification│   ← Fire-and-forget; no response needed
│  Service   │
└────────────┘
```

**Sync vs Async Decision Matrix:**

| Communication | Pattern | Reason |
|--------------|---------|--------|
| Order → Payment | **Synchronous (REST)** | Payment must succeed before confirming order; user is waiting |
| Order → Delivery Mgmt | **Asynchronous (Kafka)** | Agent assignment can happen in background; order is already confirmed |
| Tracking → ETA | **Asynchronous (Kafka)** | ETA is eventually consistent; doesn't block location ingestion |
| Any → Notification | **Asynchronous (Kafka)** | Fire-and-forget; notification delays are acceptable |

---

#### Auth Service
- **Technology:** Node.js + Express
- **Database:** PostgreSQL (users table)
- **Responsibilities:** Registration, login, JWT issuance, token refresh, RBAC
- **Why separate?** Authentication logic changes independently of business logic; can be replaced with Auth0/Cognito later

#### Order Service
- **Technology:** Spring Boot (Java)
- **Database:** PostgreSQL (ACID transactions)
- **Responsibilities:** Order CRUD, state machine management, order history
- **Why PostgreSQL?** Orders involve financial data and must be ACID-compliant. JOIN queries across orders, users, and restaurants are natural in relational DBs.

#### Tracking Service ⚡ (Critical Path)
- **Technology:** Node.js (high I/O throughput)
- **Database:** MongoDB with 2dsphere indexes
- **Cache:** Redis (latest agent location)
- **Responsibilities:** Ingest GPS updates, validate sequence numbers, serve real-time location
- **Why MongoDB?** Native geo-spatial indexing (`2dsphere`) enables efficient `$near` queries for finding closest agents. Schema flexibility handles varying GPS metadata across device types.
- **Why Node.js?** Non-blocking I/O is ideal for handling 100K+ concurrent WebSocket connections and high-frequency GPS ingestion.

#### Delivery Management Service
- **Technology:** Spring Boot (Java)
- **Database:** PostgreSQL
- **Responsibilities:** Agent assignment, availability tracking, reassignment logic
- **Why?** Complex assignment algorithms (nearest agent + rating + load) benefit from Java's robust computation capabilities

#### ETA Service
- **Technology:** Python (ML-based predictions)
- **Database:** Redis (cache computed ETAs)
- **External:** Google Maps Directions API
- **Responsibilities:** Compute and update ETA based on distance, traffic, historical patterns
- **Why Python?** Leverages ML libraries (scikit-learn, TensorFlow) for traffic prediction models trained on historical data

#### Payment Service
- **Technology:** Spring Boot (Java)
- **Database:** PostgreSQL (ACID)
- **External:** Razorpay / Stripe SDK
- **Responsibilities:** Process payments, handle refunds, maintain transaction ledger
- **Why separate?** Payment logic requires strict ACID compliance, PCI-DSS considerations, and independent audit trails

#### Notification Service
- **Technology:** Node.js
- **External:** FCM (push), Twilio (SMS)
- **Responsibilities:** Template-based multi-channel notifications
- **Why event-driven?** Notifications are fire-and-forget; Kafka ensures no notification is lost even if the service is temporarily down

---

### 3.3 Data Layer

#### Database Selection Justification

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DATA LAYER DECISION MATRIX                        │
├──────────────┬──────────────┬────────────────────────────────────────┤
│ Database     │ Use Case     │ Why This DB?                           │
├──────────────┼──────────────┼────────────────────────────────────────┤
│ PostgreSQL   │ Orders,      │ ACID compliance for financial data.    │
│              │ Users,       │ Rich SQL for complex queries.          │
│              │ Payments     │ Mature replication & failover.         │
├──────────────┼──────────────┼────────────────────────────────────────┤
│ MongoDB      │ Location     │ Native 2dsphere geo-spatial indexes.   │
│              │ Tracking     │ High write throughput for GPS data.    │
│              │              │ Flexible schema for device metadata.   │
├──────────────┼──────────────┼────────────────────────────────────────┤
│ Redis        │ Caching      │ Sub-millisecond reads for latest       │
│              │              │ location & ETA. TTL-based eviction.    │
│              │              │ Pub/Sub for WebSocket fan-out.         │
├──────────────┼──────────────┼────────────────────────────────────────┤
│ Elasticsearch│ Search       │ Full-text search for restaurants &     │
│              │              │ menus. Analytics aggregations.         │
└──────────────┴──────────────┴────────────────────────────────────────┘
```

---

### 3.4 Event Streaming Layer (Apache Kafka)

**Why Kafka over RabbitMQ/SQS?**

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| Throughput | **1M+ msg/sec** | ~50K msg/sec | ~3K msg/sec |
| Message Retention | Configurable (days/weeks) | Until consumed | 14 days |
| Replay Capability | ✅ Yes | ❌ No | ❌ No |
| Ordering | ✅ Per partition | ❌ No guarantee | ✅ FIFO queues |

**Kafka Topics Design:**

| Topic | Partition Key | Partitions | Consumers |
|-------|--------------|------------|-----------|
| `location-updates` | `agent_id` | 64 | Tracking Service, ETA Service |
| `order-events` | `order_id` | 32 | Delivery Mgmt, Notification |
| `delivery-events` | `region` | 16 | ETA Service, Admin Dashboard |
| `dead-letter-queue` | — | 8 | Alert Service (manual review) |

**Partitioning by `agent_id`** ensures all location updates for a given agent are processed in order by the same consumer — critical for sequence number validation and trajectory computation.

---

## 4. Data Flow Diagrams

### 4.1 Order Placement Flow
```
Customer App          API Gateway        Order Service      Payment Service    Kafka         Delivery Mgmt    Notification
    │                     │                   │                   │              │                │               │
    │  POST /orders       │                   │                   │              │                │               │
    │────────────────────▶│                   │                   │              │                │               │
    │                     │  Validate JWT     │                   │              │                │               │
    │                     │  Rate limit check │                   │              │                │               │
    │                     │──────────────────▶│                   │              │                │               │
    │                     │                   │  Create Order     │              │                │               │
    │                     │                   │  (status=CREATED) │              │                │               │
    │                     │                   │                   │              │                │               │
    │                     │                   │  POST /payments   │              │                │               │
    │                     │                   │──────────────────▶│              │                │               │
    │                     │                   │                   │  Process     │                │               │
    │                     │                   │                   │  Payment     │                │               │
    │                     │                   │  Payment Success  │              │                │               │
    │                     │                   │◀──────────────────│              │                │               │
    │                     │                   │                   │              │                │               │
    │                     │                   │  Publish: ORDER_CONFIRMED        │                │               │
    │                     │                   │─────────────────────────────────▶│                │               │
    │                     │                   │                   │              │  Consume event │               │
    │                     │                   │                   │              │───────────────▶│               │
    │                     │                   │                   │              │                │  Assign Agent │
    │                     │                   │                   │              │                │               │
    │                     │                   │                   │              │  Publish: AGENT_ASSIGNED       │
    │                     │                   │                   │              │──────────────────────────────▶│
    │                     │                   │                   │              │                │  Send Push   │
    │  200 OK             │                   │                   │              │                │  Send SMS    │
    │◀────────────────────│                   │                   │              │                │               │
```

### 4.2 Real-Time Tracking Flow
```
Agent App            API Gateway       Tracking Service     Kafka          ETA Service      Redis         WebSocket       Customer App
    │                     │                   │               │               │              │              │                │
    │ POST /tracking/     │                   │               │               │              │              │                │
    │ location            │                   │               │               │              │              │                │
    │────────────────────▶│                   │               │               │              │              │                │
    │                     │──────────────────▶│               │               │              │              │                │
    │                     │                   │               │               │              │              │                │
    │                     │                   │ Validate:     │               │              │              │                │
    │                     │                   │ • Sequence #  │               │              │              │                │
    │                     │                   │ • Speed check │               │              │              │                │
    │                     │                   │ • Kalman filter│              │              │              │                │
    │                     │                   │               │               │              │              │                │
    │                     │                   │ Publish to    │               │              │              │                │
    │                     │                   │ Kafka         │               │              │              │                │
    │                     │                   │──────────────▶│               │              │              │                │
    │                     │                   │               │  Consume      │              │              │                │
    │                     │                   │               │──────────────▶│              │              │                │
    │                     │                   │               │               │ Recalculate  │              │                │
    │                     │                   │               │               │ ETA          │              │                │
    │                     │                   │               │               │              │              │                │
    │                     │                   │ Update latest location + ETA  │              │              │                │
    │                     │                   │─────────────────────────────────────────────▶│              │                │
    │                     │                   │               │               │              │              │                │
    │                     │                   │               │               │              │  Push update │                │
    │                     │                   │               │               │              │─────────────▶│                │
    │                     │                   │               │               │              │              │  WS: location  │
    │                     │                   │               │               │              │              │  + ETA update  │
    │                     │                   │               │               │              │              │───────────────▶│
    │  200 OK             │                   │               │               │              │              │                │
    │◀────────────────────│                   │               │               │              │              │                │
```

---

## 5. Technology Stack Summary

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **API Gateway** | NGINX / Kong | Central routing, throttling, JWT validation at edge |
| **Microservices** | Spring Boot / Node.js | Spring Boot for computation-heavy services; Node.js for I/O-heavy services |
| **Relational DB** | PostgreSQL | ACID for orders & payments; mature ecosystem |
| **Document DB** | MongoDB (2dsphere) | Native geo-spatial queries; high write throughput |
| **Cache** | Redis Cluster | Sub-millisecond reads; Pub/Sub for WebSocket fan-out |
| **Search** | Elasticsearch | Full-text restaurant/menu search with geo-filtering |
| **Streaming** | Apache Kafka | High throughput (1M+ msg/sec); durable event log with replay |
| **Containerization** | Docker + Kubernetes | Auto-scaling pods; rolling deployments; self-healing |
| **Monitoring** | Prometheus + Grafana | Metrics collection + visualization; alerting |
| **Logging** | ELK Stack | Centralized log aggregation and search |
| **CI/CD** | GitHub Actions | Automated testing, building, and deployment pipelines |

---

## 6. Deployment Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     AWS Cloud Infrastructure                       │
│                                                                    │
│  ┌─────────────┐     ┌─────────────────────────────────────────┐ │
│  │   Route 53   │     │        VPC (Virtual Private Cloud)       │ │
│  │   (DNS)      │────▶│                                          │ │
│  └─────────────┘     │  ┌─────────────────────────────────┐    │ │
│                       │  │     Public Subnet                │    │ │
│  ┌─────────────┐     │  │  ┌──────────┐  ┌──────────┐     │    │ │
│  │ CloudFront   │     │  │  │   ALB    │  │   NAT    │     │    │ │
│  │   (CDN)      │     │  │  │          │  │ Gateway  │     │    │ │
│  └─────────────┘     │  │  └──────────┘  └──────────┘     │    │ │
│                       │  └─────────────────────────────────┘    │ │
│                       │                                          │ │
│                       │  ┌─────────────────────────────────┐    │ │
│                       │  │     Private Subnet (AZ-1)        │    │ │
│                       │  │  ┌──────────────────────────┐   │    │ │
│                       │  │  │   Kubernetes Cluster      │   │    │ │
│                       │  │  │   (EKS)                   │   │    │ │
│                       │  │  │   • Auth Pods             │   │    │ │
│                       │  │  │   • Order Pods            │   │    │ │
│                       │  │  │   • Tracking Pods         │   │    │ │
│                       │  │  │   • Delivery Pods         │   │    │ │
│                       │  │  │   • ETA Pods              │   │    │ │
│                       │  │  │   • Payment Pods          │   │    │ │
│                       │  │  │   • Notification Pods     │   │    │ │
│                       │  │  └──────────────────────────┘   │    │ │
│                       │  │                                  │    │ │
│                       │  │  ┌──────┐ ┌──────┐ ┌──────┐    │    │ │
│                       │  │  │Kafka │ │Redis │ │ PG   │    │    │ │
│                       │  │  │Broker│ │Cluster│ │Primary│   │    │ │
│                       │  │  └──────┘ └──────┘ └──────┘    │    │ │
│                       │  └─────────────────────────────────┘    │ │
│                       │                                          │ │
│                       │  ┌─────────────────────────────────┐    │ │
│                       │  │     Private Subnet (AZ-2)        │    │ │
│                       │  │   (Replica for High Availability) │   │ │
│                       │  │   • K8s Worker Nodes (replica)   │    │ │
│                       │  │   • PG Read Replica              │    │ │
│                       │  │   • Redis Replica                │    │ │
│                       │  └─────────────────────────────────┘    │ │
│                       └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```
