# Low-Level Design (LLD)
## Real-Time Food Delivery Tracking System

**Student:** Shreshta | **UID:** 23BAI70375  
**Subject:** System Design (23CST-390) | **Semester:** 6th

---

## 1. Database Schema Design

### 1.1 Entity-Relationship Overview

```
┌────────────┐       ┌────────────────┐       ┌─────────────────┐
│   Users    │       │  Restaurants   │       │   Menu Items    │
│            │       │                │       │                 │
│ PK: id     │       │ PK: id         │       │ PK: id          │
│ name       │       │ name           │       │ FK: restaurant  │
│ email      │       │ address        │──────▶│ name            │
│ phone      │       │ latitude       │  1:N  │ price           │
│ role       │       │ longitude      │       │ category        │
│ password   │       │ cuisine_type   │       │ is_available    │
└──────┬─────┘       │ rating         │       └────────┬────────┘
       │             │ is_active      │                │
       │             └────────────────┘                │
       │                                               │
       │  1:N                                          │ N:M
       ▼                                               ▼
┌────────────────┐                          ┌─────────────────┐
│    Orders      │                          │  Order Items    │
│                │                          │                 │
│ PK: id         │                          │ PK: id          │
│ FK: customer   │◀────────────────────────▶│ FK: order_id    │
│ FK: restaurant │         1:N              │ FK: menu_item   │
│ FK: agent      │                          │ quantity        │
│ status         │                          │ unit_price      │
│ total_amount   │                          │ special_instr   │
│ created_at     │                          └─────────────────┘
│ updated_at     │
└──────┬─────────┘
       │
       │ 1:1
       ▼
┌────────────────┐       ┌──────────────────┐
│   Payments     │       │ Delivery Agents  │
│                │       │                  │
│ PK: id         │       │ PK: id           │
│ FK: order_id   │       │ FK: user_id      │
│ amount         │       │ vehicle_type     │
│ method         │       │ is_available     │
│ status         │       │ current_lat      │
│ transaction_id │       │ current_lng      │
│ created_at     │       │ rating           │
└────────────────┘       │ total_deliveries │
                         └──────────────────┘
```

---

### 1.2 PostgreSQL Schemas

#### `users` Table
```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    phone           VARCHAR(15) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(20) NOT NULL CHECK (role IN ('customer', 'agent', 'admin')),
    avatar_url      TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index for login lookups
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
```

**Design Decisions:**
- **UUID primary keys**: Avoids sequential ID guessing attacks and works across distributed systems without coordination
- **`password_hash`**: Never store plain passwords; use bcrypt with salt rounds ≥ 12
- **`role` CHECK constraint**: Database-level enforcement prevents invalid roles — defense in depth beyond application validation

---

#### `restaurants` Table
```sql
CREATE TABLE restaurants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id        UUID REFERENCES users(id),
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    address         TEXT NOT NULL,
    latitude        DECIMAL(10, 8) NOT NULL,
    longitude       DECIMAL(11, 8) NOT NULL,
    cuisine_type    VARCHAR(50)[],           -- Array of cuisine tags
    rating          DECIMAL(2, 1) DEFAULT 0.0,
    total_ratings   INTEGER DEFAULT 0,
    avg_prep_time   INTEGER DEFAULT 30,      -- minutes
    is_active       BOOLEAN DEFAULT TRUE,
    opening_time    TIME,
    closing_time    TIME,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Geo-spatial index for nearby restaurant queries
CREATE INDEX idx_restaurants_location ON restaurants 
    USING gist (ll_to_earth(latitude, longitude));
CREATE INDEX idx_restaurants_cuisine ON restaurants USING gin(cuisine_type);
CREATE INDEX idx_restaurants_active ON restaurants(is_active) WHERE is_active = TRUE;
```

**Design Decisions:**
- **`cuisine_type` as PostgreSQL array**: Enables GIN indexing for fast containment queries (`WHERE 'Italian' = ANY(cuisine_type)`)
- **Partial index on `is_active`**: Only indexes active restaurants — smaller index, faster queries since most queries filter by active status
- **`DECIMAL(10,8)` for coordinates**: Provides ~1mm precision — more than sufficient for delivery use cases

---

#### `menu_items` Table
```sql
CREATE TABLE menu_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id   UUID NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    price           DECIMAL(10, 2) NOT NULL CHECK (price > 0),
    category        VARCHAR(50) NOT NULL,    -- 'appetizer', 'main', 'dessert', 'beverage'
    image_url       TEXT,
    is_vegetarian   BOOLEAN DEFAULT FALSE,
    is_available    BOOLEAN DEFAULT TRUE,
    preparation_time INTEGER DEFAULT 15,     -- minutes
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_menu_restaurant ON menu_items(restaurant_id);
CREATE INDEX idx_menu_available ON menu_items(restaurant_id, is_available) 
    WHERE is_available = TRUE;
```

---

#### `orders` Table
```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES users(id),
    restaurant_id   UUID NOT NULL REFERENCES restaurants(id),
    agent_id        UUID REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'created'
                    CHECK (status IN ('created', 'confirmed', 'preparing', 
                                      'picked_up', 'in_transit', 'delivered', 'cancelled')),
    total_amount    DECIMAL(10, 2) NOT NULL,
    delivery_fee    DECIMAL(10, 2) DEFAULT 0.00,
    delivery_address TEXT NOT NULL,
    delivery_lat    DECIMAL(10, 8) NOT NULL,
    delivery_lng    DECIMAL(11, 8) NOT NULL,
    special_instructions TEXT,
    estimated_delivery TIMESTAMP WITH TIME ZONE,
    idempotency_key UUID UNIQUE,             -- Prevents duplicate orders
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Composite indexes for common query patterns
CREATE INDEX idx_orders_customer ON orders(customer_id, created_at DESC);
CREATE INDEX idx_orders_agent ON orders(agent_id, status) WHERE status IN ('picked_up', 'in_transit');
CREATE INDEX idx_orders_status ON orders(status) WHERE status NOT IN ('delivered', 'cancelled');
CREATE INDEX idx_orders_restaurant ON orders(restaurant_id, created_at DESC);
```

**Design Decisions:**
- **`idempotency_key`**: UNIQUE constraint at DB level ensures that even if the application layer fails, a duplicate order submission with the same key is rejected
- **Partial indexes on `status`**: Active orders (`NOT IN ('delivered', 'cancelled')`) are a small fraction of total orders — partial indexes keep the index small and fast
- **`agent_id` is nullable**: Orders start without an agent; assignment happens asynchronously after payment confirmation

---

#### `order_items` Table
```sql
CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    menu_item_id    UUID NOT NULL REFERENCES menu_items(id),
    quantity        INTEGER NOT NULL CHECK (quantity > 0),
    unit_price      DECIMAL(10, 2) NOT NULL,  -- Snapshot price at order time
    special_instructions TEXT,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
```

**Design Decision:** `unit_price` is a **snapshot** of the menu item's price at order time, not a FK reference. This is critical because menu prices change — we need the historical price for financial accuracy.

---

#### `payments` Table
```sql
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    amount          DECIMAL(10, 2) NOT NULL,
    currency        VARCHAR(3) DEFAULT 'INR',
    method          VARCHAR(20) NOT NULL CHECK (method IN ('card', 'upi', 'wallet', 'cod')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded')),
    gateway_txn_id  VARCHAR(255),            -- External payment gateway reference
    gateway_response JSONB,                  -- Raw response for audit
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_status ON payments(status) WHERE status = 'pending';
CREATE UNIQUE INDEX idx_payments_gateway ON payments(gateway_txn_id) WHERE gateway_txn_id IS NOT NULL;
```

**Design Decision:** `gateway_response` is stored as **JSONB** because each payment gateway returns different response structures. JSONB allows flexible storage while still supporting indexed queries if needed.

---

#### `delivery_agents` Table
```sql
CREATE TABLE delivery_agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID UNIQUE NOT NULL REFERENCES users(id),
    vehicle_type    VARCHAR(20) CHECK (vehicle_type IN ('bicycle', 'motorcycle', 'car')),
    license_number  VARCHAR(50),
    is_available    BOOLEAN DEFAULT FALSE,
    is_verified     BOOLEAN DEFAULT FALSE,
    current_latitude  DECIMAL(10, 8),
    current_longitude DECIMAL(11, 8),
    rating          DECIMAL(2, 1) DEFAULT 5.0,
    total_deliveries INTEGER DEFAULT 0,
    total_earnings  DECIMAL(12, 2) DEFAULT 0.00,
    last_heartbeat  TIMESTAMP WITH TIME ZONE,  -- Agent app heartbeat
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_agents_available ON delivery_agents(is_available, is_verified)
    WHERE is_available = TRUE AND is_verified = TRUE;
CREATE INDEX idx_agents_location ON delivery_agents 
    USING gist (ll_to_earth(current_latitude, current_longitude))
    WHERE is_available = TRUE;
```

**Design Decision:** `last_heartbeat` enables the system to detect stale agent status. If `NOW() - last_heartbeat > 60 seconds`, the agent is considered unavailable regardless of `is_available` flag — handles app crashes gracefully.

---

### 1.3 MongoDB Schema (Tracking Service)

#### `agent_locations` Collection
```javascript
// Collection: agent_locations
{
    _id: ObjectId,
    agent_id: String,           // References delivery_agents.id
    order_id: String,           // Active order being delivered
    location: {
        type: "Point",
        coordinates: [longitude, latitude]  // GeoJSON format [lng, lat]
    },
    altitude: Number,           // Meters (optional, from GPS)
    accuracy: Number,           // GPS accuracy in meters
    speed: Number,              // km/h (computed)
    heading: Number,            // Direction in degrees (0-360)
    timestamp: NumberLong,      // Unix timestamp (milliseconds)
    sequence_number: NumberLong, // Monotonically increasing per agent
    battery_level: Number,      // Agent device battery percentage
    network_type: String,       // "wifi", "4g", "3g"
    created_at: ISODate
}

// Indexes
db.agent_locations.createIndex({ "location": "2dsphere" });
db.agent_locations.createIndex({ "agent_id": 1, "timestamp": -1 });
db.agent_locations.createIndex({ "order_id": 1 });
db.agent_locations.createIndex({ "timestamp": 1 }, { expireAfterSeconds: 86400 }); // TTL: 24 hours
```

**Design Decisions:**
- **2dsphere index**: Enables efficient geo-spatial queries like `$near` (find nearest agent) and `$geoWithin` (agents within a radius)
- **TTL index on `timestamp`**: Automatically deletes location records older than 24 hours — prevents unbounded collection growth without manual cleanup
- **Compound index `{agent_id, timestamp}`**: Optimizes "latest N locations for agent X" queries for trajectory rendering
- **`sequence_number`**: Enables idempotent processing — if a retried update has a sequence number ≤ the last processed, it's safely ignored

#### `location_history` Collection (Cold Storage)
```javascript
// Collection: location_history (for analytics & dispute resolution)
{
    _id: ObjectId,
    agent_id: String,
    order_id: String,
    route: {
        type: "LineString",
        coordinates: [[lng1, lat1], [lng2, lat2], ...]  // Full delivery path
    },
    total_distance: Number,     // km
    total_duration: Number,     // seconds
    start_time: ISODate,
    end_time: ISODate,
    waypoints_count: Number
}

db.location_history.createIndex({ "agent_id": 1, "start_time": -1 });
db.location_history.createIndex({ "order_id": 1 });
```

---

## 2. Class Design / Entities

### 2.1 Core Domain Models

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLASS DIAGRAM                                    │
│                                                                          │
│  ┌──────────────┐         ┌──────────────────┐                          │
│  │   <<enum>>   │         │    <<enum>>       │                          │
│  │  UserRole    │         │  OrderStatus      │                          │
│  │─────────────│         │─────────────────  │                          │
│  │ CUSTOMER    │         │ CREATED           │                          │
│  │ AGENT       │         │ CONFIRMED         │                          │
│  │ ADMIN       │         │ PREPARING         │                          │
│  └──────────────┘         │ PICKED_UP        │                          │
│                           │ IN_TRANSIT        │                          │
│                           │ DELIVERED         │                          │
│                           │ CANCELLED         │                          │
│                           └──────────────────┘                          │
│                                                                          │
│  ┌──────────────────────────────┐      ┌──────────────────────────────┐ │
│  │          User                │      │        Restaurant            │ │
│  │──────────────────────────────│      │──────────────────────────────│ │
│  │ - id: UUID                   │      │ - id: UUID                   │ │
│  │ - name: String               │      │ - ownerId: UUID              │ │
│  │ - email: String              │      │ - name: String               │ │
│  │ - phone: String              │      │ - address: String            │ │
│  │ - passwordHash: String       │      │ - location: GeoPoint         │ │
│  │ - role: UserRole             │      │ - cuisineType: String[]      │ │
│  │ - isActive: Boolean          │      │ - rating: Double             │ │
│  │──────────────────────────────│      │ - isActive: Boolean          │ │
│  │ + register(): User          │      │──────────────────────────────│ │
│  │ + login(): AuthToken        │      │ + getMenu(): MenuItem[]      │ │
│  │ + updateProfile(): User     │      │ + updateAvailability(): void │ │
│  └──────────────────────────────┘      └──────────────────────────────┘ │
│            △                                      △                     │
│            │ extends                               │ has many            │
│            │                                       │                     │
│  ┌──────────────────────────────┐      ┌──────────────────────────────┐ │
│  │      DeliveryAgent           │      │         MenuItem             │ │
│  │──────────────────────────────│      │──────────────────────────────│ │
│  │ - vehicleType: VehicleType   │      │ - id: UUID                   │ │
│  │ - isAvailable: Boolean       │      │ - restaurantId: UUID         │ │
│  │ - currentLocation: GeoPoint  │      │ - name: String               │ │
│  │ - rating: Double             │      │ - price: Decimal             │ │
│  │ - totalDeliveries: Integer   │      │ - category: String           │ │
│  │ - lastHeartbeat: Timestamp   │      │ - isAvailable: Boolean       │ │
│  │──────────────────────────────│      │──────────────────────────────│ │
│  │ + updateLocation(): void    │      │ + updatePrice(): void        │ │
│  │ + toggleAvailability(): void│      │ + toggleAvailability(): void │ │
│  │ + acceptOrder(): Boolean    │      └──────────────────────────────┘ │
│  └──────────────────────────────┘                                       │
│                                                                          │
│  ┌──────────────────────────────┐      ┌──────────────────────────────┐ │
│  │           Order              │      │        OrderItem             │ │
│  │──────────────────────────────│      │──────────────────────────────│ │
│  │ - id: UUID                   │◆────▶│ - id: UUID                   │ │
│  │ - customerId: UUID           │ 1:N  │ - orderId: UUID              │ │
│  │ - restaurantId: UUID         │      │ - menuItemId: UUID           │ │
│  │ - agentId: UUID?             │      │ - quantity: Integer           │ │
│  │ - status: OrderStatus        │      │ - unitPrice: Decimal         │ │
│  │ - totalAmount: Decimal       │      │ - specialInstructions: String│ │
│  │ - deliveryAddress: String    │      └──────────────────────────────┘ │
│  │ - deliveryLocation: GeoPoint │                                       │
│  │ - idempotencyKey: UUID       │      ┌──────────────────────────────┐ │
│  │ - createdAt: Timestamp       │      │         Payment              │ │
│  │──────────────────────────────│      │──────────────────────────────│ │
│  │ + create(): Order           │◆────▶│ - id: UUID                   │ │
│  │ + confirm(): Order          │ 1:1  │ - orderId: UUID              │ │
│  │ + cancel(): Order           │      │ - amount: Decimal            │ │
│  │ + updateStatus(): Order     │      │ - method: PaymentMethod      │ │
│  │ + assignAgent(): Order      │      │ - status: PaymentStatus      │ │
│  └──────────────────────────────┘      │ - gatewayTxnId: String       │ │
│                                         │──────────────────────────────│ │
│                                         │ + process(): Payment        │ │
│                                         │ + refund(): Payment         │ │
│                                         └──────────────────────────────┘ │
│                                                                          │
│  ┌──────────────────────────────┐      ┌──────────────────────────────┐ │
│  │      AgentLocation           │      │       LocationHistory        │ │
│  │──────────────────────────────│      │──────────────────────────────│ │
│  │ - agentId: String            │      │ - agentId: String            │ │
│  │ - orderId: String            │      │ - orderId: String            │ │
│  │ - location: GeoPoint         │      │ - route: GeoLineString       │ │
│  │ - speed: Double              │      │ - totalDistance: Double       │ │
│  │ - heading: Double            │      │ - totalDuration: Integer      │ │
│  │ - timestamp: Long            │      │ - startTime: Timestamp       │ │
│  │ - sequenceNumber: Long       │      │ - endTime: Timestamp         │ │
│  │ - batteryLevel: Integer      │      └──────────────────────────────┘ │
│  │──────────────────────────────│                                       │
│  │ + validate(): Boolean       │                                       │
│  │ + isOutOfOrder(): Boolean   │                                       │
│  │ + isSpeedRealistic(): Boolean│                                      │
│  └──────────────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Service Layer Classes

#### Order Service
```java
public class OrderService {
    private final OrderRepository orderRepo;
    private final PaymentService paymentService;
    private final KafkaProducer kafkaProducer;

    // Creates an order with idempotency protection
    public Order createOrder(CreateOrderRequest request) {
        // 1. Check idempotency key
        // 2. Validate restaurant is active
        // 3. Validate menu items are available
        // 4. Calculate total amount
        // 5. Process payment (synchronous)
        // 6. Create order record
        // 7. Publish ORDER_CONFIRMED event to Kafka
        // 8. Return order with status
    }

    // State machine transition with validation
    public Order updateStatus(UUID orderId, OrderStatus newStatus) {
        // Validates: CREATED→CONFIRMED→PREPARING→PICKED_UP→IN_TRANSIT→DELIVERED
        // Rejects invalid transitions (e.g., CREATED→DELIVERED)
    }
}
```

#### Tracking Service
```javascript
class TrackingService {
    constructor(mongoClient, redisClient, kafkaProducer) {
        this.db = mongoClient.db('tracking');
        this.redis = redisClient;
        this.kafka = kafkaProducer;
    }

    async processLocationUpdate(update) {
        // 1. Validate sequence number (reject out-of-order)
        // 2. Validate speed (reject unrealistic jumps > 120 km/h)
        // 3. Apply Kalman filter for GPS smoothing
        // 4. Store in MongoDB (agent_locations collection)
        // 5. Update Redis with latest location (SET agent:{id}:location)
        // 6. Publish to Kafka location-updates topic
        // 7. Return 200 OK
    }

    async getCurrentLocation(orderId) {
        // 1. Try Redis first (cache hit path — sub-millisecond)
        // 2. Fallback to MongoDB if cache miss
        // 3. Return location + ETA
    }
}
```

#### Delivery Assignment Service
```java
public class DeliveryAssignmentService {
    private final AgentRepository agentRepo;
    private final KafkaProducer kafkaProducer;

    // Assigns nearest available agent using geo-spatial query
    public DeliveryAgent assignAgent(Order order) {
        // 1. Query available agents within 5km radius
        //    (PostgreSQL: earth_distance or PostGIS)
        // 2. Sort by: distance (40%) + rating (30%) + idle_time (30%)
        // 3. Send assignment notification to top agent
        // 4. Wait 30 seconds for acceptance
        // 5. If timeout → reassign to next agent
        // 6. If accepted → update order.agent_id
        // 7. Publish AGENT_ASSIGNED event to Kafka
    }
}
```

#### ETA Service
```python
class ETAService:
    def __init__(self, redis_client, maps_client):
        self.redis = redis_client
        self.maps = maps_client

    def calculate_eta(self, agent_location, delivery_location):
        """
        Calculate dynamic ETA using multiple signals:
        1. Google Maps Directions API (real-time traffic)
        2. Historical average for this route + time of day
        3. Kalman-filtered agent speed
        
        Fallback: distance / historical_avg_speed (if Maps API fails)
        """
        try:
            # Primary: Maps API with real-time traffic
            eta = self.maps.directions(
                origin=agent_location,
                destination=delivery_location,
                departure_time='now',
                traffic_model='best_guess'
            ).duration_in_traffic
        except MapsAPIError:
            # Fallback: distance / historical average speed
            distance = haversine(agent_location, delivery_location)
            avg_speed = self.get_historical_speed(agent_location, time_of_day)
            eta = distance / avg_speed

        # Cache in Redis with 30-second TTL
        self.redis.setex(f"eta:{order_id}", 30, eta)
        return eta
```

---

### 2.3 Design Patterns Used

| Pattern | Where Applied | Justification |
|---------|---------------|---------------|
| **State Machine** | Order status transitions | Prevents invalid state transitions; auditable trail |
| **Repository Pattern** | All data access layers | Abstracts DB implementation; enables unit testing with mocks |
| **Observer/Pub-Sub** | Kafka event publishing | Decouples services; producers don't know about consumers |
| **Circuit Breaker** | Payment & Maps API calls | Prevents cascading failures during external service outages |
| **Strategy Pattern** | ETA calculation (primary vs fallback) | Swappable algorithms; graceful degradation |
| **Idempotency Key** | Order creation, payment processing | Network retries don't create duplicates |
| **Singleton** | Database connection pools, Kafka producers | Avoids resource exhaustion from multiple connections |
| **Factory Method** | Notification creation (push/SMS/email) | Extensible notification channel support |

---

### 2.4 Order State Machine Validation

```java
public class OrderStateMachine {
    private static final Map<OrderStatus, Set<OrderStatus>> VALID_TRANSITIONS = Map.of(
        OrderStatus.CREATED,     Set.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED),
        OrderStatus.CONFIRMED,   Set.of(OrderStatus.PREPARING, OrderStatus.CANCELLED),
        OrderStatus.PREPARING,   Set.of(OrderStatus.PICKED_UP, OrderStatus.CANCELLED),
        OrderStatus.PICKED_UP,   Set.of(OrderStatus.IN_TRANSIT),
        OrderStatus.IN_TRANSIT,  Set.of(OrderStatus.DELIVERED),
        OrderStatus.DELIVERED,   Set.of(),    // Terminal state
        OrderStatus.CANCELLED,   Set.of()     // Terminal state
    );

    public static boolean isValidTransition(OrderStatus from, OrderStatus to) {
        return VALID_TRANSITIONS.getOrDefault(from, Set.of()).contains(to);
    }
}
```

**Justification:** Encoding valid transitions in a map (vs. if-else chains) makes the state machine:
1. **Self-documenting** — the transition rules are visible at a glance
2. **Easily extensible** — adding a new state (e.g., `RETURNED`) requires one line
3. **Testable** — each transition can be verified independently
