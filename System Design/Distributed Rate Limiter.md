Rate Limiting is a network-based strategy that is used to control the rate of requests that are being sent to a server from a client or a collection of clients. It is typically used to prevent DoS attacks and limit web scraping. Typically, rate-limiting is a client-side response.

# Throttling

Throttling is a server-side response where feedback is provided to the caller indicating that there are too many requests. The objective is similar to Rate Limiting. Implementing, it on the server makes the overall architecture scalable and resilient to distributed denial of service attacks.

>[!question]
>- [ ] What is difference between rate limiting and throttling?
# Noisy Neighbor Problem

Typical web services are built in a multi-tenant model. This means that the underlying resources such as the Network, Disk, Database, etc. are shared between multiple clients. In such an architecture one of the tenants can hog the resources such as network bandwidth by triggering several queries to the system. It leads to a lack of resources for other tenants. We call this problem the _**Noisy Neighbor Problem**_.

# Functional Requirements

- **Request Throttling:** Limit the number of requests per client (user, IP, API key) within specified time windows (e.g., 100 requests/minute).

- **Configurable Rules:** Support flexible, per-client and per-endpoint rules (e.g., 5 marketing messages per day per user).

- **Error Handling:** Return HTTP 429 with clear error messages and rate limit headers when limits are exceeded.

- **Multi-Algorithm Support:** Implement various algorithms (Token Bucket, Fixed Window, Leaky Bucket, Sliding Window, etc.).

- **Distributed Enforcement:** Ensure rate limits are enforced consistently across all nodes in a distributed deployment.

- **Monitoring & Logging:** Track usage, violations, and provide metrics for observability.

# Non-Functional Requirements

- **Low Latency:** Rate limiter decisions must add minimal latency (ideally <10ms).

- **High Availability:** Service must be resilient (99.99% uptime) and handle node failures gracefully.

- **Scalability:** Must support millions of clients and high QPS (queries per second).

- **Consistency:** Counters and rules must be accurate and synchronized across distributed nodes.

- **Configurability:** Rules can be updated without downtime.

- **Security:** Prevent abuse, spoofing, and ensure data integrity.


# Back-of-the-Envelope (BOE) Estimation

## Traffic Estimation

- Assume 10M users, each making up to 100 requests/minute ≈ 1B requests/hour peak.
## Storage Estimation

- Assuming 10M users defines max 10 rules ≈ 100M rules.
- Assuming size of each rule data to be 100 bytes long, then total require storage: 100M rules * 100 bytes ≈ 10 GB for storing rule definitions.
## Memory Estimation

- **Counters:** For sliding window, ~20B counters/hour (if per-minute), stored in Redis with TTL.

## Compute Estimation

- 1 vCPU can handle ~10K requests/sec; for 1M QPS, need ~100 vCPUs.

## Network Bandwidth Estimation

- Each request may require 1KB for metadata/counter updates ⇒ 1M QPS = 1GB/sec.

# API Design

**Endpoint:** `POST /rate-limit/check`

**Request:**

```
{
  "user_id": "12345",
  "ip": "1.2.3.4",
  "api_key": "abcdef",
  "endpoint": "/api/v1/resource"
}
```

**Response (Allowed):**

- HTTP 200 OK
- Headers:
    - `X-RateLimit-Limit`: 100
    - `X-RateLimit-Remaining`: 42
    - `X-RateLimit-Reset`: 1715000400

**Response (Rate Limited):**

- HTTP 429 Too Many Requests
- Headers:
    - `Retry-After`: 60
- Body:

```
{
  "code": "rate_limited",
  "message": "Rate limit exceeded"
}
```


# Database Design

### Rules Storage (NoSQL/SQL)

- `rule_id`: Unique rule identifier
- `client_type`: Type of client (e.g., user_id, ip, api_key)
- `limit`: Maximum number of requests (e.g., 100)
- `window`: Time window (e.g., minute, hour, day)
- `scope`: Whether the rule is endpoint-specific or global

### Counter Storage (Redis/Memcached)

- **Key:** `{client_id}:{endpoint}:{window_start}`
- **Value:** Integer count with TTL matching the window
- **Commands:** `INCR`, `EXPIRE` (atomic)

---

## 7. System Architecture & Flow

**Components:**

- **API Gateway/Middleware:** Entry point, forwards requests to Rate Limiter.
    
- **Rate Limiter Service:** Stateless service, enforces rules, updates counters, returns allow/deny.
    
- **Rules Database:** Stores rate limit configs, hot-reloaded by Rate Limiter.
    
- **Counter Store:** Redis/Memcached for fast, atomic counter updates.
    
- **Monitoring/Logging:** Tracks usage, violations, and system health.
    

**Request Flow:**

1. API Gateway receives request.
    
2. Forwards metadata to Rate Limiter.
    
3. Rate Limiter:
    
    - Loads applicable rules from cache.
        
    - Fetches/increments counter in Redis.
        
    - Checks limit:
        
        - **Within limit:** Forwards request, returns headers.
            
        - **Exceeded:** Returns 429 with headers and error body.
            
4. Monitoring/logging updated asynchronously.
    

---

# Distributed Challenges & Optimizations

- **Consistency:** Use Redis atomic ops or Lua scripts for counter updates.

- **Sharding:** Distribute counters by user or endpoint for scalability.

- **Fault Tolerance:** Expire counters, fallback to local cache on Redis outage, fail-open or fail-closed as per business need.

- **Eventual Consistency:** Nodes may cache and periodically sync with central store to reduce latency.

- **Monitoring:** Use Prometheus/Grafana for metrics, alert on anomalies.

## 9. Example Rule Format

```
domain: messaging
descriptors:
  - key: message_type
    value: marketing
    rate_limit:
      unit: day
      requests_per_unit: 5
```

_Allows 5 marketing messages per day per user._

## 12. Final Visual Architecture Diagram

Here’s a high-level architecture diagram of the system:

```
                        +------------------------+
                        |  API Gateway / NGINX   |
                        +------------------------+
                                   |
                                   v
                        +------------------------+
                        |   Rate Limiter Service |
                        +------------------------+
                          |            |        |
         +----------------+            |        +------------------+
         |                             v                           |
         v                  +------------------+         +------------------+
+------------------+        |   Rules Database  |         | Monitoring/Alert |
| Counter Store     |       +------------------+         +------------------+
| (Redis/Memcached) |               ^                        (Prometheus, etc.)
+------------------+               /
                                Hot cache sync
```