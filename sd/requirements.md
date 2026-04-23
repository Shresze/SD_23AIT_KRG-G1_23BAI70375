# 📋 Software Requirements Specification (SRS)
## Real-Time Food Delivery Tracking System

**Student:** Shreshta | **UID:** 23BAI70375  
**Subject:** System Design (23CST-390) | **Semester:** 6th  
**Branch:** BE CSE AIML | **Section:** 23AIT_KRG_G1(A)

---

## 1. Functional Requirements (FRs)

### FR1: User Registration & Authentication
| ID | Requirement | Priority |
|----|-------------|----------|
| FR1.1 | Users (customers, delivery agents, admins) must register and log in via email/phone | High |
| FR1.2 | System shall support **JWT-based authentication** with refresh token rotation | High |
| FR1.3 | **Role-based access control (RBAC)** must be enforced across all endpoints | High |
| FR1.4 | OAuth 2.0 social login support (Google, Facebook) | Medium |

**Justification:** JWT tokens are stateless and scale horizontally without session affinity. RBAC ensures least-privilege access — customers cannot access admin dashboards, agents cannot modify menus, and admins have full oversight. Refresh token rotation mitigates token theft risks.

---

### FR2: Order Placement
| ID | Requirement | Priority |
|----|-------------|----------|
| FR2.1 | Customer shall browse restaurants and menus filtered by location, cuisine, and ratings | High |
| FR2.2 | Customer shall place orders with item customization (quantity, add-ons, special instructions) | High |
| FR2.3 | System shall generate a **unique order ID** (UUID v4) for every order | High |
| FR2.4 | System shall process payments via external payment gateway (Razorpay/Stripe) | High |
| FR2.5 | Order status shall transition through a well-defined state machine | High |

**Order State Machine:**
```
Created → Confirmed → Preparing → Picked Up → In Transit → Delivered
                 ↘ Cancelled       ↘ Cancelled
```

**Justification:** A strict state machine prevents invalid transitions (e.g., jumping from "Created" to "Delivered") and provides an auditable trail. UUID v4 guarantees globally unique order IDs without centralized coordination — critical for distributed systems.

---

### FR3: Delivery Agent Assignment
| ID | Requirement | Priority |
|----|-------------|----------|
| FR3.1 | System shall assign the **nearest available agent** using geo-spatial queries | High |
| FR3.2 | Agent availability status shall be maintained in real time via heartbeat mechanism | High |
| FR3.3 | Reassignment shall occur if agent fails to accept within **30-second timeout** | High |
| FR3.4 | System shall support priority-based assignment (rating, delivery history) | Medium |

**Justification:** Nearest-agent assignment minimizes delivery time and fuel costs. The 30-second timeout with automatic reassignment prevents order stalls. Heartbeat-based availability (vs. manual toggle) ensures accuracy — if an agent's app crashes, the system detects unavailability within seconds.

---

### FR4: Real-Time Location Tracking
| ID | Requirement | Priority |
|----|-------------|----------|
| FR4.1 | Delivery agent app shall send GPS updates every **3–5 seconds** | High |
| FR4.2 | Tracking service shall validate and store location with sequence numbers | High |
| FR4.3 | Customer shall view real-time agent movement on an interactive map | High |
| FR4.4 | System shall support **WebSocket-based live updates** to connected clients | High |
| FR4.5 | System shall apply Kalman filtering to smooth GPS drift | Medium |

**Justification:** WebSockets provide full-duplex, persistent connections — far more efficient than HTTP polling for high-frequency updates (3–5 sec intervals). Sequence numbers enable idempotent processing and out-of-order detection. Kalman filtering corrects GPS inaccuracies, especially in urban canyons where satellite signals bounce off buildings.

---

### FR5: ETA Calculation
| ID | Requirement | Priority |
|----|-------------|----------|
| FR5.1 | System shall compute **dynamic ETA** based on distance, traffic, and historical data | High |
| FR5.2 | ETA shall update with every location refresh cycle | High |
| FR5.3 | System shall use fallback ETA model if external map API is unavailable | Medium |

**Justification:** Dynamic ETA (vs. static distance/speed calculation) accounts for real-world variability — traffic congestion, road closures, and time-of-day patterns. The fallback model (`ETA = distance / historical_avg_speed`) ensures graceful degradation when third-party APIs fail, maintaining user trust.

---

### FR6: Notifications
| ID | Requirement | Priority |
|----|-------------|----------|
| FR6.1 | System shall send notifications for: order confirmation, agent assigned, picked up, delivered | High |
| FR6.2 | Notifications shall be delivered via **push notifications** and **SMS** | High |
| FR6.3 | Users shall be able to configure notification preferences | Low |

**Justification:** Multi-channel notifications (push + SMS) ensure delivery even when the app is closed or the user has poor data connectivity. Event-driven notification triggers (via Kafka) decouple notification logic from core business flows.

---

### FR7: Admin Monitoring
| ID | Requirement | Priority |
|----|-------------|----------|
| FR7.1 | Admin shall monitor active deliveries on a real-time dashboard | High |
| FR7.2 | Admin shall view agent performance metrics (delivery time, ratings, acceptance rate) | Medium |
| FR7.3 | Admin shall access order statistics with filtering and export capabilities | Medium |

**Justification:** Real-time admin dashboards enable proactive incident management — identifying stuck deliveries, underperforming agents, or regional demand spikes before they impact customer experience.

---

## 2. Non-Functional Requirements (NFRs)

### 2.1 Scalability
| Metric | Target | Justification |
|--------|--------|---------------|
| Concurrent Users | ≥ **1,000,000** | Peak dinner hours in metro cities can generate massive simultaneous demand |
| Location Updates | ≥ **100,000/second** | Each active agent sends updates every 3–5 sec; 300K active agents = 100K updates/sec |
| Scaling Model | **Horizontal** for stateless services | Adding more pods is cheaper and faster than vertical scaling; no single point of failure |

### 2.2 Availability
| Metric | Target | Justification |
|--------|--------|---------------|
| Uptime SLA | ≥ **99.9%** (< 8.76 hrs downtime/year) | Food delivery is time-sensitive; even 10 min of downtime during peak hours causes significant revenue and trust loss |
| Deployment | **Multi-AZ** (Availability Zone) | Protects against datacenter-level failures (power, network, natural disasters) |
| Database Failover | **Automatic** (< 30 sec) | Manual failover is too slow for real-time systems; automated promotion of replicas minimizes data loss |

### 2.3 Latency
| Operation | Target | Justification |
|-----------|--------|---------------|
| Location update processing | **< 200 ms** | Users expect real-time map movement; >200 ms creates visible lag |
| ETA recalculation | **< 500 ms** | ETA updates trigger UI changes; must feel instantaneous |
| API response time (p95) | **< 300 ms** | Industry benchmark for mobile app APIs; higher latency increases bounce rate |

### 2.4 Fault Tolerance
| Pattern | Application | Justification |
|---------|-------------|---------------|
| **Retry + Exponential Backoff** | All inter-service calls | Prevents thundering herd during transient failures; gives downstream services recovery time |
| **Circuit Breaker** | Payment & Map API calls | Stops cascading failures; fails fast instead of holding resources during extended outages |
| **Dead Letter Queue (DLQ)** | Kafka consumers | Captures failed events for manual investigation without blocking the main processing pipeline |

### 2.5 Data Consistency
| Domain | Consistency Model | Justification |
|--------|-------------------|---------------|
| **Payments** | Strong consistency (ACID) | Financial transactions must never be duplicated or lost; double-charging destroys trust |
| **Location Tracking** | Eventual consistency | Minor lag (< 1 sec) in location display is acceptable; strong consistency would bottleneck throughput |
| **API Idempotency** | Idempotent keys on all write APIs | Network retries must not create duplicate orders or double-charge users |
