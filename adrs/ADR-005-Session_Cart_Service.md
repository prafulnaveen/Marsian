# ADR-005 — Choice of Memorystore (Redis) for Session and Cart Cache

## Context

The ticketing system requires fast, temporary storage for user sessions and shopping carts:

**Session Requirements:**
- Store user login sessions with authentication tokens
- Track session state across multiple API requests
- Maintain user context (customer_id, permissions, preferences)
- Session timeout and automatic expiration (typically 30 minutes to 24 hours)
- Support concurrent sessions from thousands of visitors

**Cart Requirements:**
- Store in-progress shopping carts (ticket selections, quantities, promotions applied)
- Temporary state during checkout flow before payment
- Fast retrieval during booking process (sub-100ms latency expected)
- Ability to persist cart across multiple browsing sessions (recovery from browser crashes)
- Handle concurrent cart modifications if multi-window browsing

**Access Patterns:**
- Very high read frequency (every API request reads session state)
- Moderate write frequency (adding/removing items from cart, applying promotions)
- Sub-100ms latency required for responsive user experience
- Temporary data (no permanent storage requirement)
- High concurrency (thousands of visitors with active sessions simultaneously)

**Scale Requirements:**
- 5,000-15,000 daily visitors with potential 3x growth
- Peak concurrency: ~500-1,000 active sessions during peak hours
- Average session size: ~2-5 KB (customer context, cart items)
- Cart TTL: 30 minutes to 2 hours
- Session TTL: 8-24 hours

**Platform Context:**
- Deployed on GKE within GCP ecosystem
- Accessed from multiple microservices (Booking Service, Auth Service, API Gateway)
- Must be highly available; session loss frustrates users mid-checkout

## Decision

**We will use Google Cloud Memorystore for Redis as the session and cart cache layer.**

Redis provides an in-memory data store optimized for fast read/write operations with sub-millisecond latency, automatic expiration support, and distributed session management—ideal for caching temporary data across microservices without the operational complexity of self-managed infrastructure.

## Key Differentiators

- **Sub-Millisecond Latency**  
  Redis operations (GET, SET, LPUSH) typically complete in <1ms. For session lookups happening on every API request, this latency is imperceptible to users. Database round trips (Cloud SQL) would add 5-50ms per request, creating noticeable UI delays.

- **In-Memory Performance**  
  Redis stores data in RAM, avoiding disk I/O. With 5,000-15,000 daily visitors generating ~500-1,000 concurrent sessions, the working set (session data) fits entirely in memory, ensuring consistent performance without cache misses.

- **Automatic Expiration (TTL)**  
  Redis TTL support automatically evicts expired sessions and cart data without application logic. Sessions don't need cleanup jobs; they expire server-side, freeing memory and preventing stale data accumulation.

- **Data Structures for Cart Use Cases**  
  Redis provides native data structures (Hashes, Lists, Sets) perfect for cart representation:
  - Hashes store cart metadata (customer_id, created_time, total_price)
  - Lists/Sets store line items (ticket_type, quantity, promotion_id)
  - Sorted Sets can track cart updates chronologically
  No need for complex serialization; Redis handles structure natively.

- **Atomic Operations**  
  Redis commands are atomic. Adding/removing cart items, applying promotions, and updating quantities are single atomic operations. No race conditions or partial updates—guaranteed consistency for concurrent cart modifications.

- **Pub/Sub for Real-Time Updates**  
  Redis Pub/Sub enables real-time communication (e.g., cart state updates across browser tabs, session invalidation on logout). Prevents scenarios where user updates cart in one tab but other tab shows stale data.

- **Persistence Options**  
  Redis can optionally persist data to disk (RDB snapshots, AOF logs) for durability. For non-critical cart data, persistence isn't required; for sessions where loss is acceptable, RDB snapshots provide protection against total data loss.

- **Managed Service (Memorystore)**  
  Google Cloud Memorystore for Redis is fully managed: automatic backups, automatic failover, patching, scaling, and monitoring. Team avoids Redis cluster management, replication setup, and infrastructure maintenance.

- **Simple Integration Pattern**  
  Sessions/carts are standard key-value patterns. Redis client libraries exist for all programming languages. No impedance mismatch; straightforward mapping from application session objects to Redis keys.

- **High Availability Configuration**  
  Memorystore offers:
  - High-availability with automatic failover
  - Read replicas for distributing query load
  - 99.9% uptime SLA
  - Automatic backups with point-in-time recovery
  Sessions remain available during maintenance or zone failures.

- **Horizontal Scalability with Clustering**  
  Redis Cluster (available in Memorystore) distributes data across multiple nodes, supporting unlimited cache size and throughput. As user base grows, seamlessly add cluster nodes.

- **Cost-Effective**  
  In-memory caching is cheaper than storing sessions in persistent databases. Redis uses RAM instead of disk storage, reducing per-operation costs compared to Cloud SQL.

## Alternatives Considered

- **Google Cloud SQL (PostgreSQL/MySQL)**  
  Storing sessions in a relational database:
  - **Latency Overhead**: Database queries (round trip, query parsing, execution) add 5-50ms per session lookup. On a 100-request page load, session lookups alone add 500ms-5s. Redis's <1ms response is unacceptable replacement.
  - **Unnecessary Durability**: Sessions and carts are temporary data; paying for persistent storage durability is wasteful.
  - **Write Amplification**: Every API request reads session; every cart modification writes. Cloud SQL can become a bottleneck. Redis handles thousands of operations/second effortlessly.
  - **Connection Pool Contention**: Session lookups would consume connection pool slots needed for transactional operations (payment, ticket issuance).
  - **Cost**: Cloud SQL charges per instance regardless of utilization. Storing temporary data is inefficient; Redis's memory-based pricing aligns better with actual usage.

- **Google Cloud Firestore**  
  Cloud Native NoSQL for sessions/carts:
  - **Latency**: Firestore queries are 10-100ms—slower than Redis for session lookups. Real-time pricing and network round trips add overhead.
  - **Cost at Scale**: Firestore charges per read/write operation. 15,000 daily visitors × 100 API requests/visitor = 1.5M session reads/day. Firestore would cost ~$0.015 per 100 reads × 15,000 = $22.50/day (~$675/month). Redis instance: $10-50/month. Cost inefficient for high-read workloads.
  - **TTL Limitations**: Firestore TTL support exists but isn't as efficient as Redis's native expiration. Expired documents still consume reads until deleted.
  - **Transactional Overhead**: Firestore transactions can't span cart and session; mutations must be separate, increasing latency.

- **Google Cloud Bigtable**  
  High-throughput, low-latency NoSQL:
  - **Overkill for Sessions**: Bigtable is optimized for time-series analytics, not transactional caching. Requires expertise in row key design, column families, and read/write hotspot optimization.
  - **Cost**: Bigtable has minimum cluster cost ($5,000+/month). Sessions/carts don't justify this.
  - **Complexity**: Bigtable requires operational expertise; Redis is simpler.

- **Memcached (Open-Source)**  
  Simple, fast in-memory cache:
  - **No Durability**: Unlike Redis with RDB/AOF, Memcached has no persistence. Cache restarts lose all data. For sessions, this is acceptable; for carts, users lose shopping progress.
  - **No TTL Precision**: Memcached relies on LRU eviction; no guaranteed expiration times. Sessions might persist longer than desired.
  - **No Advanced Data Structures**: Memcached stores only strings; cart representation requires application-level serialization.
  - **No Pub/Sub**: Can't implement real-time cart/session updates across browser tabs.
  - **Operational Complexity (Self-Managed)**: If not using managed Memcached (which Google doesn't offer), team manages clustering, failover, and monitoring.

- **Self-Managed Redis**  
  Running Redis on Compute Engine or GKE pods:
  - **Operational Burden**: Team responsible for Redis clustering, replication, failover orchestration, backup/recovery, and monitoring.
  - **HA Complexity**: Redis Sentinel or Cluster must be configured and maintained; Memorystore automates this.
  - **Patching and Upgrades**: Manual Redis version upgrades risk downtime. Memorystore handles transparently.
  - **Staffing**: Requires Redis expertise; Memorystore eliminates need.
  - **Cluster Management**: Managing Redis nodes, memory allocation, and data distribution is non-trivial.

- **Session Storage in Cookies**  
  Storing session data client-side in browser cookies:
  - **Size Limitation**: Cookies limited to ~4KB total per domain; session + cart data often exceeds this.
  - **Security Risk**: Session tokens in cookies vulnerable to XSS; encrypted token still requires server-side validation.
  - **Serialization Overhead**: Browser must serialize/deserialize on every request; adding latency.
  - **Offline Inconsistency**: No single source of truth; distributed session state prone to anomalies.
  - **Logout Complexity**: Invalidating sessions requires server tracking; defeats cookie-only benefit.

- **Application-Level Memory (In-Process Caching)**  
  Storing sessions in application memory (each microservice):
  - **No Sharing Across Services**: Booking Service and API Gateway can't share session data; requires duplicated lookups.
  - **No HA**: Service crash loses all session data; users must re-login mid-checkout.
  - **Horizontal Scaling Issues**: Scaling to multiple service replicas means session affinity needed (sticky sessions); reduces load balancing flexibility.
  - **Memory Bloat**: Each service instance maintains full session copy; wasteful.

- **Elasticsearch**  
  Distributed search/analytics engine for session caching:
  - **Overkill**: Elasticsearch is designed for full-text search and analytics, not key-value caching.
  - **Latency**: Elasticsearch queries 10-50ms; slower than Redis.
  - **Resource Overhead**: Elasticsearch requires more CPU/memory than Redis for simple key lookups.
  - **Operational Complexity**: Elasticsearch cluster management is more complex than Redis.

- **Apache Cassandra**  
  Distributed NoSQL database:
  - **Latency**: Cassandra queries 5-50ms with replication; Redis <1ms.
  - **Overkill**: Cassandra is designed for distributed, high-volume analytics. Sessions are temporary, low-volume data.
  - **Operational Complexity**: Cassandra cluster management, node balancing, and repair operations are complex.
  - **Cost**: Cassandra cluster minimum (3 nodes) more expensive than Redis instance.

## Why Memorystore (Redis) is Better Than Alternatives

| Criterion | Redis | Firestore | Memcached | Cloud SQL | Cassandra | Cookies | In-Process |
|-----------|-------|-----------|-----------|-----------|-----------|---------|-----------|
| **Latency (p50)** | <1ms | 10-100ms | <1ms | 5-50ms | 5-50ms | N/A | <0.1ms* |
| **Latency (consistency)** | Consistent | Variable | Consistent | Variable | Variable | N/A | Consistent |
| **TTL Support** | Excellent | Good | Fair | Manual | Manual | N/A | Manual |
| **Data Structures** | Rich (Hash, List, Set) | Document | Strings only | Tables | Columns | Strings | Any |
| **Atomicity** | Excellent | Fair | Good | Excellent | Good | N/A | Excellent |
| **Cost (1M reads/day)** | ~$20-50/mo | ~$675/mo | ~$20-50/mo | ~$50/mo | $500+/mo | Free | Free |
| **HA & Failover** | Automatic (Managed) | Automatic | Manual (if self-managed) | Automatic | Manual | N/A | None |
| **Operational Burden** | Minimal | Minimal | Medium-High | Minimal | High | None | None |
| **Shared Across Services** | Yes | Yes | Yes | Yes | Yes | No (client-side) | No |
| **Pub/Sub Support** | Yes | No | No | No | No | No | No |
| **Persistence Options** | Yes (RDB/AOF) | Automatic | No | Automatic | Automatic | N/A | N/A |
| **Suitable for Session/Cart** | Excellent | Fair | Fair | Poor | Poor | Poor | Poor |

*In-process memory is faster locally but no remote access/sharing.

**Why Redis wins for session/cart caching:**

1. **Sub-Millisecond Latency**: Sessions are read on every API request. Cloud SQL's 5-50ms adds hundreds of milliseconds to page load. Redis's <1ms is imperceptible.

2. **High-Read Workload Cost Efficiency**: 15,000 daily visitors × 100 API requests = 1.5M session reads/day. Firestore's per-operation cost (~$0.015 per 100 ops) becomes expensive. Redis fixed pricing (~$30-50/month) is 10-20x cheaper.

3. **Temporary Data Pattern**: Sessions and carts are temporary (TTL measured in hours, not permanent). Cloud SQL's durability guarantees are unnecessary overhead. Redis's in-memory model is cost-optimized for transient data.

4. **Native Data Structure Support**: Cart representation (items, quantities, promotions) maps naturally to Redis Hashes and Lists. No serialization/deserialization overhead. Firestore/Cloud SQL require application-level mapping.

5. **Automatic Expiration**: Redis TTL automatically evicts expired sessions without cleanup jobs. Firestore and Cloud SQL require background processes to delete old entries.

6. **Atomic Operations**: Adding/removing cart items is atomic in Redis. No race conditions with concurrent modifications. Cloud SQL requires explicit locking; Firestore doesn't guarantee consistency.

7. **Pub/Sub for Real-Time**: Redis Pub/Sub enables real-time cart/session updates across browser tabs and services. Firestore and Cloud SQL lack this capability.

8. **High Availability with Minimal Ops**: Memorystore automates failover, backups, and monitoring. Self-managed Redis requires expertise. Memcached has no built-in HA.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Data Loss Risk** | Redis is in-memory; unplanned shutdown (crash, zone failure) loses all data without persistence | Enable RDB snapshots (hourly) for durability, use Memorystore high-availability mode with automatic failover, implement graceful session recovery (users re-login), test disaster recovery procedures |
| **Memory Exhaustion** | If session/cart growth exceeds allocated memory, eviction policy (LRU) could prematurely expire valid sessions | Set appropriate TTLs (sessions 8-24 hours, carts 30 min-2 hours), monitor memory utilization via Cloud Monitoring, implement cache expiration cleanup, use Memorystore auto-scaling |
| **Network Latency from GKE to Memorystore** | Even sub-millisecond Redis response adds network latency (a few milliseconds round trip) | Co-locate Redis in same GCP region as GKE cluster, use private VPC peering, keep connection pool active to avoid reconnection overhead, measure p99 latency |
| **Connection Pool Exhaustion** | Multiple microservices connecting to Redis could exhaust connection limits | Use Redis connection pooling libraries (e.g., redis-py-cluster, node-redis), set connection pool size conservatively, implement circuit breakers for Redis failures, monitor active connections |
| **Cache Stampede** | If popular session expires, multiple requests simultaneously rebuild it, overloading the system | Implement cache-aside pattern with locking (mutex), use lazy TTL with refresh-on-access, set staggered expiration times, monitor cache hit ratio |
| **Serialization Overhead** | Serializing complex session objects (JSON, Protocol Buffers) adds latency and CPU | Use efficient serialization (MessagePack vs. JSON), store only essential session data, avoid storing large objects, profile serialization cost |
| **Key Collision/Namespace Pollution** | Multiple services using same Redis instance without proper key prefixing could overwrite data | Establish naming convention (service:session:session_id), use separate Memorystore instances per environment (dev/staging/prod), implement key prefix validation |
| **Monitoring and Debugging Difficulty** | Redis internals (memory usage, eviction, key distribution) are harder to debug than SQL databases | Enable Redis slowlog monitoring, set up Cloud Monitoring dashboards for hit/miss rates and latency, use redis-cli for debugging, establish runbooks |
| **Eventual Consistency with Async Replication** | Memorystore replicas are asynchronously replicated; brief window where replica data stale | Accept brief (sub-second) inconsistency for non-critical cart data, use strong consistency patterns for session validation (read-through), test failover scenarios |
| **Scaling Limits with Single Node** | Single Memorystore instance has CPU/network limits; at 100K+ concurrent sessions, vertical scaling alone insufficient | Design for horizontal scaling early (Redis Cluster), partition sessions by user ID or region, consider multiple Memorystore instances with service-level sharding, load test at expected scale |

## Conclusion

Memorystore for Redis is the optimal choice for session and cart caching because:

1. **Sub-Millisecond Latency**: Sessions are read on every request; <1ms response time is critical for responsive UI. Cloud SQL's 5-50ms introduces unacceptable user-facing latency.

2. **Cost-Efficient for Read-Heavy Workloads**: 1.5M+ session reads/day makes fixed-pricing Redis (vs. Firestore's per-operation model) 10-20x cheaper.

3. **Automatic Expiration**: Redis TTL automates session/cart cleanup without background jobs or manual management.

4. **Native Data Structures**: Hashes, Lists, and Sets map naturally to cart/session representation; no serialization impedance mismatch.

5. **Atomic Operations**: Concurrent cart modifications remain consistent without explicit locking; Redis guarantees atomicity.

6. **Pub/Sub for Real-Time**: Enable cart/session updates across browser tabs and microservices without polling.

7. **Managed Service**: Memorystore automates backups, failover, patching, and monitoring—eliminating operational complexity of self-managed Redis.

8. **HA Built-In**: Automatic failover and replication provide 99.9% uptime without configuration.

9. **Seamless GCP Integration**: Workload Identity, Cloud Logging, Cloud Monitoring, and VPC integration provide security and observability.

**Recommendation**: Deploy Memorystore for Redis (high-availability mode with automatic failover). Set session TTL to 8-24 hours (depending on security requirements), cart TTL to 30 minutes to 2 hours. Enable RDB snapshots for durability. Use Redis connection pooling from GKE services. Monitor hit/miss rates and memory utilization via Cloud Monitoring. Start with a standard instance (2-5 GB) and auto-scale as sessions grow. Implement graceful degradation: if Redis becomes unavailable, degrade to shorter session TTL (force re-login) rather than serving stale data. Test failover scenarios quarterly to ensure automatic recovery works as expected.
