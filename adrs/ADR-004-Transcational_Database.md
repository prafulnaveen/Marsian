# ADR-004 — Choice of Cloud SQL for Transactional Database

## Context

The ticketing system is the operational core of the estate platform, handling:

**Transactional Requirements:**
- Customer orders and ticket purchases
- Payment processing and reconciliation
- Ticket generation and validity tracking
- Entry log recording (scan times, zones, gates)
- Loyalty program point tracking
- Family pass group management
- Promotion application and validation

**Data Model Characteristics:**
- Highly relational schema with 11+ entities (Customer, Order, OrderItem, Ticket, Payment, Loyalty, FamilyPassGroup, TicketType, Promotion, EntryLog, Zone)
- Complex foreign key relationships and constraints
- ACID transaction requirements (payment orders must be atomically consistent—partial orders are unacceptable)
- Normalized structure with multiple dependent entities per order

**Operational Requirements:**
- Support 5,000-15,000 daily visitors; potential 3x growth
- High availability; downtime directly impacts revenue
- Multiple microservices accessing simultaneously (Booking, Issuance, Validation, Loyalty, Reporting)
- Strong data consistency requirements (no double-charging, no duplicate tickets issued)
- Real-time transaction processing during peak hours (gate scannings, online bookings)
- Backup and disaster recovery capabilities

**Platform Context:**
- Deployed on GKE within GCP ecosystem
- Integration with Pub/Sub for event streaming (when tickets are issued, orders placed)
- Integration with BigQuery for analytics and reporting
- Cloud Storage for receipt/ticket exports

## Decision

**We will use Google Cloud SQL (PostgreSQL) as the transactional database for the ticketing system.**

Cloud SQL provides a managed, fully relational database that guarantees ACID compliance, supports complex queries and schema constraints, and integrates seamlessly with GCP services—all without the operational burden of self-management.

## Key Differentiators

- **Full ACID Compliance**  
  Cloud SQL (PostgreSQL) provides strong ACID guarantees essential for financial transactions. Payments cannot be partially processed; orders must be atomic. PostgreSQL's transaction isolation levels prevent anomalies like phantom reads, lost updates, and dirty reads—critical for a ticketing system where double-charging or duplicate ticket issuance is unacceptable.

- **Relational Schema Support**  
  The ticketing system has a normalized, relational data model with 11+ entities and complex relationships. SQL's ability to enforce referential integrity, unique constraints, and cascading actions ensures data consistency. NoSQL alternatives require encoding these constraints in application logic, introducing bugs and inconsistencies.

- **Complex Query Patterns**  
  The system requires complex queries: finding all tickets for an order with promotion details, tracking entry logs by customer and zone, identifying loyalty tier changes, generating sales reports. SQL's expressive query language handles these efficiently; NoSQL approaches require multiple queries and client-side joins.

- **Managed Service (Reduced Operational Burden)**  
  Cloud SQL is fully managed: automatic backups, point-in-time recovery, automatic patching, replication, and failover. The team avoids cluster management, replication setup, and infrastructure maintenance—allowing focus on application features rather than database operations.

- **High Availability Options**  
  Cloud SQL offers:
  - Automatic failover to high-availability replicas
  - Read replicas for distributing query load
  - Multi-zone deployment for zone-level resilience
  - Automated backups with 35-day retention
  These HA capabilities ensure the ticketing system remains available during regional disruptions or maintenance.

- **PostgreSQL Ecosystem**  
  PostgreSQL is the most feature-rich open-source relational database: advanced data types (JSON, arrays, ranges), full-text search, window functions, recursive CTEs, and extensions (PostGIS, pgcrypto). This richness enables sophisticated application logic to be expressed in the database rather than scattered across services.

- **Seamless GCP Integration**  
  Cloud SQL integrates natively with:
  - Cloud IAM for access control and role-based permissions
  - Workload Identity for keyless service authentication from GKE
  - Cloud Logging and Cloud Monitoring for observability
  - Cloud SQL Proxy for secure, encrypted connections
  - BigQuery (via Cloud Storage exports or federated queries) for analytics
  No middleware translation layers required.

- **Cost Predictability**  
  Cloud SQL pricing is transparent and predictable: machine type, storage, backups, and replication are clearly itemized. Auto-scaling features (storage auto-expansion) prevent surprise outages. Unlike serverless options with unpredictable consumption patterns, SQL databases scale with query complexity, not request count.

- **Multi-Service Access Pattern**  
  Multiple microservices access the same data (Booking reads Promotions, Validation reads Tickets, Loyalty reads Loyalty accounts). Cloud SQL's connection pooling and standard SQL interfaces enable efficient, concurrent access from multiple services without architectural complexity.

- **Data Consistency Guarantees**  
  Strong consistency ensures that after a ticket is issued, all subsequent reads see that ticket. Eventual consistency systems are inappropriate for ticketing: allowing gate scanners to read stale data could result in duplicate entries or rejected valid tickets.

## Alternatives Considered

- **Firestore (Cloud Native NoSQL Database)**  
  Google's managed NoSQL document database. However:
  - **No ACID Transactions Across Documents**: Firestore transactions are limited to a single document write or a small batch. Multi-document transactions (order + payment + ticket issuance) are not atomic, risking inconsistencies.
  - **No Referential Integrity**: No built-in foreign key support; relationships must be managed in application code, increasing complexity and bug surface.
  - **Complex Query Limitations**: Firestore queries are document-centric. Complex queries like "find all tickets for a customer with their promotions" require multiple queries and client-side joining, increasing latency and complexity.
  - **Cost at Scale**: Firestore charges per read/write operation. A single transaction (order creation) might generate 5-10 writes; peak traffic (15,000 visitors) could generate millions of operations, becoming expensive.
  - **Denormalization Required**: To avoid complex multi-document queries, data must be denormalized. Updating a promotion affects multiple order documents, requiring eventual consistency approaches prone to data anomalies.

- **Cloud Bigtable (Wide-Column NoSQL)**  
  Google's high-throughput, low-latency database. However:
  - **Not Designed for Transactions**: Bigtable is optimized for time-series and analytical workloads, not transactional consistency.
  - **Complex Application Logic**: No SQL queries; all business logic must be coded in application services, duplicating logic across services.
  - **Operational Complexity**: Bigtable requires expertise in column family design, row key design, and read/write hotspot management. Not ideal for standard transactional workloads.
  - **Overkill for Ticketing**: The ticketing system doesn't have Bigtable's use case (high-volume time-series data, sparse operations). Cloud SQL is a simpler, better fit.

- **Firestore in Datastore Mode**  
  Google's older managed NoSQL service. However:
  - **Limited to Weak Consistency**: Eventual consistency is not acceptable for ticketing transactions.
  - **Legacy Technology**: No longer the recommended Google NoSQL offering; newer features go to Firestore (native mode only).
  - **Query Limitations**: Similar to Firestore, lacks complex query support and referential integrity.

- **Spanner (Distributed Relational Database)**  
  Google's globally distributed SQL database with strong consistency. However:
  - **Overkill for Single-Region Deployment**: Spanner's main value is geographic distribution with strong consistency. The ticketing system is deployed in a single region (GCP region containing the estate). Single-region Spanner costs significantly more than Cloud SQL without additional benefits.
  - **Complexity**: Spanner has operational quirks (interleaving tables for performance, commit timestamp overhead). Cloud SQL is simpler.
  - **Cost**: Spanner nodes are expensive; 2-node minimum is $3,320/month. Cloud SQL PostgreSQL's smallest instance ($10/month) handles typical ticketing load.

- **AlloyDB for PostgreSQL**  
  Google's high-performance managed PostgreSQL (4x faster than standard PostgreSQL). However:
  - **Premium Cost**: Starts at $6.50/hour (vs. Cloud SQL's $0.10-0.50/hour for smaller instances). Only justified if benchmark evidence shows performance bottlenecks, which ticketing read patterns don't exhibit.
  - **Overkill for Initial Scale**: Suitable after load testing reveals CPU/IO bottlenecks. Start with Cloud SQL; upgrade to AlloyDB if performance testing shows need.
  - **Not Fundamentally Different**: Same PostgreSQL semantics; same operational benefits; just higher cost and performance.

- **Self-Managed PostgreSQL on Compute Engine (VMs)**  
  Running PostgreSQL in GCE instances. However:
  - **Full Operational Burden**: Team responsible for patching, backup/recovery, monitoring, replication setup, and failover orchestration.
  - **Reduced Reliability**: Self-managed setup has higher downtime risk. Database crashes require manual intervention; backup failures aren't automatically detected.
  - **Staffing Costs**: Requires dedicated database administrator; Cloud SQL eliminates this role.
  - **No Automatic HA**: Manual setup of replication, failover, and load balancing increases complexity and risk. Cloud SQL's automatic failover ensures 99.95% uptime SLA.
  - **Backup Complexity**: Manual snapshot scheduling, testing, and retention policy enforcement.

- **Amazon RDS (Comparison to AWS)**  
  AWS's managed relational database. However:
  - **Cloud Lock-in**: Selecting AWS for the database contradicts the choice of GCP for compute (GKE). Multi-cloud complexity without corresponding benefit.
  - **Network Overhead**: GKE (GCP) to RDS (AWS) incurs cross-cloud latency and data egress costs.
  - **Operational Inconsistency**: Different auth models (AWS IAM vs. GCP IAM), different monitoring (CloudWatch vs. Cloud Monitoring), different tooling. Team must learn two database platforms.

- **DynamoDB (Comparison to AWS)**  
  AWS's managed NoSQL database. However:
  - **Same Issues as Firestore**: No ACID transactions across items, no referential integrity, complex queries require multiple operations.
  - **Same Lock-in Issues**: AWS-only, inconsistent with GCP platform choice.

## Why Cloud SQL is Better Than Alternatives

| Criterion | Cloud SQL | Firestore | Bigtable | Spanner | AlloyDB | Self-Managed | DynamoDB |
|-----------|-----------|-----------|----------|---------|---------|--------------|----------|
| **ACID Compliance** | Excellent | Poor (single-doc only) | None | Excellent | Excellent | Excellent | Poor |
| **Referential Integrity** | Excellent | None | None | Excellent | Excellent | Excellent | None |
| **Complex Query Support** | Excellent | Fair | Poor | Excellent | Excellent | Excellent | Poor |
| **Operational Burden** | Minimal (managed) | Minimal | Minimal | Minimal | Minimal | High (self-managed) | Minimal |
| **Multi-Service Access** | Excellent | Fair (eventual consistency) | Fair | Excellent | Excellent | Excellent | Fair |
| **Data Consistency** | Strong | Eventual | Eventual | Strong | Strong | Strong | Eventual |
| **Cost Predictability** | Excellent | Fair (per-operation) | Good | Poor (expensive) | Poor (expensive) | Moderate | Fair (per-operation) |
| **HA & Failover** | Automatic | Automatic | Automatic | Automatic | Automatic | Manual | Automatic |
| **GCP Integration** | Excellent | Excellent | Excellent | Excellent | Excellent | Good | N/A |
| **Schema Flexibility** | Limited (schema-on-write) | High (schema-less) | High (schema-less) | Limited | Limited | Limited | High |
| **Expertise Required** | Low-Medium | Medium | High | High | Medium | High | Medium |
| **Scaling Model** | Vertical + Read Replicas | Horizontal (automatic) | Horizontal (manual) | Horizontal (automatic) | Vertical + Read Replicas | Manual | Horizontal (automatic) |

**Why Cloud SQL wins for transactional ticketing:**

1. **ACID is Non-Negotiable**: Payment processing requires atomicity. A Firestore transaction can't atomically update both the order and payment status; manual error handling introduces bugs. Cloud SQL's multi-row transactions guarantee consistency.

2. **Relational Integrity**: The 11-entity schema has complex relationships. Firestore would require duplicating relationships in application code (e.g., storing ticket details in order document for fast access, then manually syncing when tickets update). This duplication is a maintenance nightmare. Cloud SQL's foreign keys prevent inconsistency at the database level.

3. **Revenue-Critical System**: Ticketing is direct revenue. Bugs (double-charging, duplicate tickets, lost orders) cost money. Cloud SQL's strong consistency and schema enforcement reduce bug surface. Firestore's eventual consistency introduces an entire class of race condition bugs.

4. **Query Complexity**: Reports like "revenue by promotion by zone" require joins across multiple entities. Cloud SQL handles this efficiently; Firestore requires fetching orders, then promotions, then zone mappings—multiple round trips, poor performance.

5. **Cost Control**: At 15,000 daily visitors, generating 30,000-50,000 transactions/day, Firestore's per-operation pricing becomes expensive. Cloud SQL's fixed instance pricing provides predictability.

6. **No Geographical Distribution Needed**: Spanner adds cost and complexity. The estate is in one location; a single-region database is appropriate. Upgrade to Spanner only if the business expands to multiple estates in different countries.

7. **Operational Simplicity**: Cloud SQL's managed service eliminates backup, replication, and failover overhead. The team focuses on application features, not database administration.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Vertical Scaling Limit** | Cloud SQL scales primarily vertically (larger machines); at extreme scale (100,000+ TPS), vertical scaling hits limits | Plan for read replicas as traffic grows; consider horizontal sharding by customer region if needed; use Spanner only after load testing proves bottleneck |
| **Connection Pool Exhaustion** | Multiple microservices (Booking, Validation, Loyalty) competing for connections can exhaust pool at peak load | Use Cloud SQL Proxy to manage connection pooling centrally, set conservative connection limits per service, implement connection timeout handling |
| **Backup Size Growth** | Database backups grow with ticket/entry log volume; storage costs increase | Implement data retention policies (archive old entry logs to BigQuery), set backup retention to minimal necessary window, use incremental backups |
| **Query Performance Degradation** | Without proper indexing, queries against years of entry logs and ticket data slow down | Monitor slow query log, create composite indexes on frequently used filter combinations (customer_id + date, zone_id + scan_time), periodically vacuum/analyze |
| **Data Consistency During Failover** | Automatic failover could cause brief inconsistencies or duplicate reads if not handled in application | Implement retry logic with idempotency keys for critical operations, use connection retry with backoff, document failover behavior for teams |
| **Cost at Low Usage** | Cloud SQL has minimum pricing; underutilized instance still costs money | Use shared-core instances for development/staging, promote to standard instances for production, monitor utilization and right-size as demand grows |
| **Platform Lock-in** | Cloud SQL is GCP-specific; moving to another database requires schema redesign | Avoid GCP-specific features (e.g., Cloud SQL proxy, IAM-based auth) if multi-cloud is anticipated; use standard PostgreSQL features to maximize portability |
| **Compliance and Data Residency** | Certain jurisdictions require data residency; Cloud SQL may not have regions matching requirements | Check data residency requirements early; select appropriate GCP region; use Cloud SQL encryption at rest and in transit for sensitive data (payment info) |

## Conclusion

Cloud SQL (PostgreSQL) is the optimal choice for the ticketing system's transactional database because:

1. **ACID Guarantees**: Multi-row transactions ensure payment orders, ticket generation, and loyalty updates are atomic. This eliminates an entire class of race condition bugs, critical for revenue-processing systems.

2. **Relational Integrity**: The complex 11-entity schema with foreign key relationships is best expressed in SQL. Cloud SQL's constraints and referential integrity prevent data anomalies that would require fixing in application code.

3. **Complex Queries**: Reports, analytics, and operational queries (revenue by promotion, entry logs by zone, customer order history) are naturally expressed in SQL. Cloud SQL handles these efficiently; NoSQL alternatives require multiple round trips.

4. **Managed Operations**: Cloud SQL eliminates database administration—no patching, backup management, replication setup, or failover orchestration. The team focuses on application development.

5. **High Availability**: Automatic failover, read replicas, multi-zone deployment, and backups provide 99.95% uptime SLA without operational overhead.

6. **GCP Ecosystem Integration**: Workload Identity, Cloud Logging, BigQuery integration, and Cloud IAM provide seamless, secure integration with the rest of the platform.

7. **Cost Efficiency**: Transparent, predictable pricing for the expected transaction volume (5,000-15,000 visitors/day). No per-operation surprises like Firestore.

8. **Single Platform Simplicity**: Deploying on GKE (GCP) with Cloud SQL (GCP) provides operational consistency in monitoring, logging, authentication, and team expertise.

**Recommendation**: Deploy on Cloud SQL with PostgreSQL 14+. Enable automatic backups with 7-day retention, configure high availability with 2+ replicas, use Cloud SQL Proxy from GKE for secure connections, and enable audit logging for compliance. Start with a standard instance (db-custom-2-7680 or similar) and monitor CPU/memory utilization; scale to larger instances or read replicas as traffic grows. Migrate entry logs to BigQuery after 1 year for long-term analytics and to control Cloud SQL storage costs.
