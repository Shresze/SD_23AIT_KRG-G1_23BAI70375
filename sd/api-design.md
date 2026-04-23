# 🌐 API Design
## Real-Time Food Delivery Tracking System

**Student:** Shreshta | **UID:** 23BAI70375  
**Subject:** System Design (23CST-390) | **Semester:** 6th

---

## 1. Overview
The system uses a RESTful API architecture for request-response cycles and WebSockets for real-time tracking.

### Base URL: `https://api.fooddelivery.com/v1`

---

## 2. API Endpoints

### 2.1 User & Auth Service
| Endpoint | Method | Description | Role |
|----------|--------|-------------|------|
| `/auth/register` | `POST` | Register a new user | All |
| `/auth/login` | `POST` | Authenticate and get JWT | All |
| `/users/profile` | `GET` | Get user profile | All |

### 2.2 Order Service
| Endpoint | Method | Description | Role |
|----------|--------|-------------|------|
| `/orders` | `POST` | Place a new order | Customer |
| `/orders/{id}` | `GET` | Get order details & status | Customer/Agent |
| `/orders/{id}/cancel`| `PATCH`| Cancel an active order | Customer/Admin|
| `/orders/history` | `GET` | List past orders | All |

### 2.3 Delivery & Tracking Service
| Endpoint | Method | Description | Role |
|----------|--------|-------------|------|
| `/tracking/{order_id}`| `GET` | Get current location & ETA | Customer |
| `/agent/location` | `POST` | Update agent GPS coordinates | Agent |
| `/agent/status` | `PATCH`| Update availability (Online/Offline)| Agent |

---

## 3. Request/Response Formats

### 3.1 Place Order
**POST** `/orders`
**Request:**
```json
{
  "restaurant_id": "rest_123",
  "items": [
    {"menu_item_id": "item_1", "quantity": 2},
    {"menu_item_id": "item_5", "quantity": 1}
  ],
  "delivery_address": "123 Main St, Apartment 4B",
  "payment_method": "CREDIT_CARD"
}
```
**Response (201 Created):**
```json
{
  "order_id": "ord_98765",
  "status": "CREATED",
  "total_amount": 45.50,
  "estimated_delivery_time": "2024-05-20T18:30:00Z"
}
```

### 3.2 Update Location (Agent)
**POST** `/agent/location`
**Request:**
```json
{
  "agent_id": "agent_007",
  "latitude": 40.7128,
  "longitude": -74.0060,
  "timestamp": "2024-05-20T18:05:00Z"
}
```
**Response (200 OK):**
```json
{
  "message": "Location updated successfully"
}
```

---

## 4. Status Codes
- `200 OK`: Success.
- `201 Created`: Successfully created a resource (e.g., Order).
- `400 Bad Request`: Invalid input parameters.
- `401 Unauthorized`: Authentication failed or missing.
- `403 Forbidden`: Authenticated but lacks permissions.
- `404 Not Found`: Resource not found.
- `429 Too Many Requests`: Rate limit exceeded.
- `500 Internal Server Error`: Unexpected server failure.

---

## 5. WebSocket Communication (Real-time Tracking)
**Endpoint:** `wss://ws.fooddelivery.com/tracking/{order_id}`

**Event: `location_update`**
```json
{
  "event": "location_update",
  "data": {
    "lat": 40.7129,
    "lng": -74.0061,
    "eta": 12,
    "unit": "minutes"
  }
}
```
