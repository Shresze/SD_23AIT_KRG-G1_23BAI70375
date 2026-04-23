# Scaling Strategy
## Real-Time Food Delivery Tracking System

**Student:** Shreshta | **UID:** 23BAI70375  
**Subject:** System Design (23CST-390) | **Semester:** 6th

---

## 1. Load Balancing

To handle massive traffic, we employ a multi-tier load balancing strategy:

- **DNS Load Balancing (Route 53):** Distributes traffic across different geographic regions based on latency.
- **Layer 7 Load Balancer (AWS ALB):**
    - Handles SSL termination.
    - Routes requests to specific microservices (Path-based routing: `/orders` vs `/tracking`).
    - Performs health checks to ensure traffic only reaches healthy pods.
- **Service Mesh (Istio):** Manages internal communication between microservices with retries, timeouts, and circuit breaking.

---

## 2. Caching Strategy

Caching is critical for meeting sub-300ms latency requirements.

### 2.1 Edge Caching (CDN)
- **Content:** Static assets like restaurant images, menu icons, and CSS/JS files.
- **Technology:** CloudFront or Cloudflare.

### 2.2 Application Caching (Redis)
- **Agent Locations:** The "Latest Location" for every active agent is stored in Redis. This avoids hitting the primary DB for every map refresh.
- **Session Data:** User authentication tokens and session info.
- **Frequent Metadata:** Popular restaurant details and menus in high-demand areas.
- **ETA Results:** Cached results of recent routing calculations to avoid redundant expensive API calls.

---

## 3. Database Scaling

### 3.1 Relational Database (PostgreSQL - Orders/Users)
- **Vertical Scaling:** Increasing CPU/RAM for the primary node (Initial phase).
- **Read Replicas:** Offloading heavy "Read" queries (Order History, User Profile) to multiple read-only replicas.
- **Database Sharding:** Once the dataset grows beyond a single server's capacity, we shard the `Orders` table by `city_id` or `user_id`.

### 3.2 NoSQL Database (MongoDB - Tracking)
- **Geo-Sharding:** Data is sharded based on geographic location (e.g., North, South, East, West regions).
- **TTL Indexes:** Automatically expire and delete old location history to keep the working set small and fast.

---

## 4. Scaling the Ingestion Pipeline

To handle **100,000 location updates per second**:

1. **Kafka Partitioning:** We partition the `location-updates` topic by `agent_id`. This ensures updates for the same agent are processed in order.
2. **Horizontal Auto-Scaling (HPA):** Kubernetes automatically scales the number of Tracking Service pods based on CPU usage and Kafka consumer lag.
3. **Write Buffering:** Instead of writing every single GPS ping to the persistent DB, we buffer updates and perform batch writes or only persist significant movements.

---

## 5. Summary of Architectural Decisions

| Challenge | Solution | Justification |
|-----------|----------|---------------|
| High Write Load | Kafka + MongoDB | Decouples ingestion from processing; MongoDB handles high-volume geo-writes. |
| High Read Latency| Redis Cache | Sub-millisecond retrieval of the most volatile data (locations/ETA). |
| Data Growth | Database Sharding | Ensures linear scalability as the user base expands. |
| Network Failures | Circuit Breakers | Prevents cascading failures when external Map APIs are down. |
| Spike in Traffic | Auto-scaling Pods | Dynamically adjusts resources during peak hours (lunch/dinner). |
