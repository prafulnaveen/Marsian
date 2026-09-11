# ADR-006 — Choice of BigQuery for Sensor Data Storage and Analytics

## Context

The estate system collects sensor telemetry data from multiple domains:

**Ride Monitoring System:**
- 40 rides with multiple sensors per ride
- Usage counters (riders per cycle, total cycles)
- Wear/vibration sensors (structural and mechanical stress indicators)
- Safety interlock sensors (restraint status, e-stop triggers)
- Data generation: ~100-500 readings per ride per day (4,000-20,000 total ride sensor events/day)

**Animal Health Monitoring System:**
- 55 enclosures with multiple sensors per enclosure
- Health sensors (weight, temperature, water quality)
- Smart feeders (food dispensed, consumption rates)
- Vision cameras (population counting, behavior analysis)
- Data generation: ~200-1,000 readings per enclosure per day (11,000-55,000 total animal sensor events/day)

**Data Characteristics:**
- **Time-Series Nature**: Sensor data is inherently sequential, timestamped measurements
- **High Volume**: 15,000-75,000 sensor events/day from day one; growth to 50,000-300,000 events/day as system scales
- **Immutable**: Historical data never updates; only new readings append
- **Long Retention**: Multi-year retention for trend analysis, anomaly detection, and compliance
- **Complex Analytics**: Aggregations (average temperature per day per enclosure), joins (ride popularity by zone), anomaly detection
- **Real-Time Ingestion**: Dataflow processes streams; data written to storage within seconds of ingestion

**Access Patterns:**
- **Write-Heavy**: Continuous append-only writes (sensors publish continuously)
- **Read-Heavy Analytics**: Complex queries for dashboards, reports, and historical analysis
- **Batch Analytics**: Weekly/monthly aggregations for trend analysis
- **Real-Time Dashboards**: Looker Studio dashboards with 5-minute refresh intervals
- **Ad-Hoc Queries**: Operations teams querying for specific incident investigations
- **Low Latency Not Required**: Queries expected to complete in seconds, not milliseconds

**Analytical Queries:**
- "What is the average temperature in Enclosure 5 over the last week?"
- "Show ride usage trends by ride type over the last month"
- "Which animals have unusual feeding behavior compared to historical average?"
- "Peak visitor flow by zone during peak hours"
- "Correlation between ride vibration patterns and maintenance visits"

## Decision

**We will use Google BigQuery as the data warehouse for sensor data storage and analytics.**

BigQuery provides a fully managed, serverless data warehouse optimized for analytical queries on large datasets. It scales seamlessly from gigabytes to petabytes, offers SQL for complex analysis, integrates natively with GCP Dataflow for streaming ingestion, and provides cost-effective storage with near-instantaneous query performance.

## Key Differentiators

- **Serverless Architecture**  
  BigQuery requires no cluster management, capacity planning, or infrastructure setup. Queries scale automatically from analyzing kilobytes to petabytes. Teams write SQL; BigQuery handles distribution, parallelization, and optimization. Eliminates operational overhead compared to self-managed data warehouses.

- **Optimized for Analytics**  
  BigQuery's columnar storage format (Dremel) is optimized for OLAP (Online Analytical Processing). Queries scanning specific columns are 100x faster than row-oriented databases. For analytics queries filtering millions of rows by a few columns, columnar storage is transformative.

- **Streaming Ingestion**  
  BigQuery supports streaming inserts: Dataflow publishes sensor events directly to BigQuery with sub-second latency. No intermediate file staging, no batch import delays. Dashboards see data within seconds of sensor publication.

- **Massive Scale**  
  BigQuery handles unlimited dataset sizes. As the estate grows from 40 rides to 400+ attractions, sensor data volume grows proportionally. BigQuery's query performance remains constant regardless of dataset size—a query on 1 year of data runs as fast as on 10 years.

- **SQL for Complex Analysis**  
  BigQuery uses standard SQL with extensions (window functions, ARRAY operations, nested/repeated fields). Operations teams write familiar SQL queries without learning specialized query languages. Complex analytics (correlations, time-series analysis, anomaly detection) are expressible in SQL.

- **Cost-Effective Storage**  
  BigQuery charges per byte scanned, not per gigabyte stored. Historical data (1+ year old) rarely queried; can be archived to Cloud Storage at 1% of BigQuery cost. Active data (current year) remains queryable. As data ages, archive and reduce query cost.

- **Seamless Dataflow Integration**  
  Dataflow processes streaming sensor data, performs transformations, checks thresholds for alerts, and writes results to BigQuery—all in one unified pipeline. No separate storage systems to integrate; Dataflow and BigQuery are designed to work together.

- **Real-Time Dashboards**  
  BigQuery supports real-time dashboards via Looker Studio and other BI tools. Dashboards query live data within seconds of ingestion. Real-time health monitoring for animals and ride safety requires this capability; traditional batch data warehouses have 12-24 hour latency.

- **Built-In ML Capabilities**  
  BigQuery ML enables in-database machine learning: anomaly detection for unusual animal behavior, time-series forecasting for visitor trends, clustering similar animals for behavioral comparison. SQL-based ML models avoid moving data to separate ML platforms.

- **Nested and Repeated Fields**  
  BigQuery supports nested data structures (JSON-like), eliminating normalization overhead. A sensor reading can include nested metadata (sensor_id, location, calibration_date) without separate lookup tables and joins. Reduces query complexity and improves performance.

- **Automatic Partitioning and Clustering**  
  BigQuery automatically partitions large tables by date, enabling efficient time-based queries. Clustering on sensor_id/enclosure_id speeds up location-based analytics. These optimizations happen transparently without application logic.

- **Query Caching and Materialized Views**  
  Repeated queries (common dashboard queries) are cached; identical queries return instantly. Materialized views pre-compute expensive aggregations, making dashboards responsive.

- **Compliance and Data Residency**  
  BigQuery supports dataset location control (specify region), enabling compliance with data residency requirements. Audit logging tracks all queries and data access for compliance audits.

- **GCP Ecosystem Integration**  
  Seamless integration with Dataflow (streaming ingestion), Looker (visualization), Cloud Storage (data export), BigQuery ML, Data Studio dashboards, and Cloud IAM (access control).

## Alternatives Considered

- **Cloud SQL / PostgreSQL**  
  Relational database for transactional data. For sensor analytics:
  - **Scaling Limitations**: Handling 50,000+ daily sensor inserts requires significant database tuning. Write throughput becomes bottleneck.
  - **Analytical Query Performance**: Row-oriented storage means scanning all rows for analytical queries. Querying 1 year of 20M sensor readings to compute "average temperature" requires full table scan (~10-100 seconds). BigQuery scans only temperature column (~100x faster).
  - **Cost at Scale**: Cloud SQL pricing is per-instance; large instances for historical data are expensive. BigQuery's per-byte-scanned pricing is cheaper for large datasets.
  - **Normalization Overhead**: Sensor readings with metadata require multiple joins; each join slows queries.
  - **No Built-In Analytics**: ML, time-series functions, and window functions require expensive user-defined functions or data export.

- **Firestore**  
  Google's NoSQL document database:
  - **Cost Prohibitive**: Firestore charges per read/write operation. 50,000 daily sensor inserts + analytics queries (reading millions of documents) costs exponentially more than BigQuery.
  - **Query Limitations**: Firestore queries are document-centric; complex analytics (join enclosure data with sensor data) require multiple queries and client-side aggregation.
  - **No Aggregation**: Firestore has no GROUP BY, no SUM/AVG functions. Computing daily averages requires fetching all readings and aggregating in application code.
  - **Not Designed for Analytics**: Firestore is optimized for transactional reads/writes, not analytical queries.

- **InfluxDB (Time-Series Database)**  
  Specialized time-series database:
  - **Excellent for Metrics**: InfluxDB excels at storing metrics (CPU, memory, disk). For simple single-metric queries (CPU over time), it's superior to BigQuery.
  - **Limited Analytical Capability**: Complex cross-domain analytics (ride popularity correlated with animal health anomalies) are difficult; InfluxDB isn't designed for joins across different measurement domains.
  - **Cost**: InfluxDB Cloud pricing is per-write. High-volume sensor data becomes expensive.
  - **Self-Managed Complexity**: On-premises InfluxDB requires cluster setup, replication, and management.
  - **Limited ML/Advanced Analytics**: InfluxDB lacks built-in ML. Advanced features require exporting data to separate ML platforms.

- **TimescaleDB (PostgreSQL Extension)**  
  PostgreSQL optimized for time-series:
  - **Better than Raw PostgreSQL**: Compression and time-partitioning improve query performance for time-series. Still slower than columnar storage (BigQuery) for analytical queries.
  - **Still Limited Analytics**: Window functions and advanced analytics are possible but more complex than BigQuery SQL.
  - **Self-Managed**: Requires hosting on Compute Engine or similar; operational burden of PostgreSQL cluster management.
  - **Scaling Limits**: Vertical scaling (larger machines) required as data grows; no unlimited horizontal scaling like BigQuery.

- **Cloud Storage + Batch Processing**  
  Storing sensor data as CSV/JSON files in Cloud Storage, processing with Dataflow:
  - **High Latency**: Data isn't queryable immediately. Batch processing happens daily/weekly; queries work on stale data from previous batch run.
  - **Complex Querying**: No SQL interface; Dataflow pipelines required for every analytical question. Requires engineering effort for ad-hoc queries.
  - **Cost**: File-based processing with Dataflow for every query is more expensive than pre-built BigQuery.
  - **Unsuitable for Real-Time Dashboards**: 12-24 hour batch lag means dashboards show stale data. Real-time animal health monitoring is impossible.

- **Elasticsearch**  
  Distributed search and analytics engine:
  - **Search-Optimized**: Elasticsearch excels at full-text search (finding sensor logs mentioning "malfunction"). Not optimized for numerical analytics.
  - **Expensive for Large Datasets**: Elasticsearch storage cost grows with data volume. BigQuery compresses better; cheaper at scale.
  - **Lower Query Performance**: Aggregations on billions of rows slower than BigQuery's columnar processing.
  - **Operational Complexity**: Elasticsearch cluster setup, shard management, and replication require expertise.

- **AWS Redshift (Comparison to AWS)**  
  AWS's managed data warehouse:
  - **Cloud Lock-in**: Selecting AWS Redshift contradicts GCP platform choice (GKE, Cloud SQL). Multi-cloud complexity without benefit.
  - **Network Cost**: GKE (GCP) to Redshift (AWS) incurs cross-cloud data egress charges.
  - **Operational Overhead**: Redshift requires more cluster management than BigQuery's serverless model.

- **Apache Druid**  
  Time-series OLAP datastore:
  - **Self-Managed Complexity**: Druid clusters require setup, configuration, and management. Not a managed service.
  - **Smaller Ecosystem**: Fewer integrations with GCP services compared to BigQuery.
  - **Lower Adoption**: Fewer team members familiar with Druid compared to SQL and BigQuery.

- **Cassandra**  
  Distributed NoSQL database:
  - **Time-Series Capable**: Cassandra can store time-series data with good performance for simple queries.
  - **Complex Analytics**: Aggregations and analytical queries are difficult without denormalization.
  - **Operational Burden**: Cassandra cluster management, node balancing, and repair operations are complex.
  - **Cost**: Cassandra cluster (minimum 3 nodes) more expensive than BigQuery for analytical workloads.

## Why BigQuery is Better Than Alternatives

| Criterion | BigQuery | PostgreSQL | InfluxDB | TimescaleDB | Cloud Storage | Elasticsearch | Druid |
|-----------|----------|-----------|----------|-------------|---------------|---------------|-------|
| **OLAP Query Speed** | Excellent (columnar) | Fair (row-oriented) | Good (for metrics) | Good (for time-series) | Poor (no index) | Fair | Good |
| **Scaling** | Unlimited (serverless) | Vertical | Horizontal | Vertical | Unlimited | Horizontal | Horizontal |
| **Analytical SQL** | Excellent (standard SQL) | Excellent | Limited | Good | None (batch) | Limited | Limited |
| **Streaming Ingestion** | Excellent (sub-second) | Fair (requires design) | Excellent | Good | Poor (batch) | Good | Fair |
| **Complex Joins** | Excellent | Excellent | Poor | Fair | Poor (batch) | Limited | Poor |
| **ML Capabilities** | Built-in (BigQuery ML) | User-defined functions | Limited | Limited | None | Limited | None |
| **Cost (50K events/day)** | $50-100/mo | $100-200/mo | $200-500/mo | $100-150/mo | $30-50/mo* | $200+/mo | $300+/mo |
| **Real-Time Dashboards** | Excellent | Good | Excellent | Good | Poor (stale data) | Good | Fair |
| **Operational Burden** | Minimal (serverless) | Medium | High (if self-managed) | Medium-High | Low | High | High |
| **Learning Curve** | Low (standard SQL) | Low | Medium | Medium | High (Dataflow) | Medium | High |
| **Compliance Features** | Excellent (audit, residency) | Good | Fair | Fair | Good | Fair | Fair |
| **GCP Integration** | Excellent (native) | Excellent | Fair | Fair | Excellent | Good | Fair |

*Cloud Storage cheaper but requires expensive Dataflow processing for every query.

**Why BigQuery wins for sensor data:**

1. **Columnar Storage for Analytics**: Sensor analytics queries scan specific columns (temperature, vibration). Columnar storage scans only needed columns, 100x faster than row-oriented databases for these queries.

2. **Unlimited Scale**: As the estate grows from 40 to 400+ attractions, sensor data volume grows 10x. BigQuery queries remain fast; traditional databases require expensive scaling.

3. **Streaming + Analytics Combined**: Dataflow publishes directly to BigQuery. No intermediate staging, no ETL delays. Dashboards see data within seconds. Traditional data warehouses require batch ETL (12-24 hour latency).

4. **SQL for Operations Teams**: Operations staff write SQL for ad-hoc investigations ("Why did temperature spike in Enclosure 5 yesterday?"). Standard SQL is familiar; no special training needed.

5. **Real-Time Dashboards**: Looker Studio dashboards query live BigQuery data. Real-time animal health alerts rely on recent data. Cloud Storage/batch processing can't provide this.

6. **Cost Efficiency at Scale**: BigQuery charges per byte scanned. Queries accessing 1% of data cost 1% of full scan. Historical data archived to Cloud Storage costs 99% less. Traditional databases charge per-instance regardless of query selectivity.

7. **Built-In Analytics**: BigQuery ML enables anomaly detection, forecasting, and clustering without moving data to separate ML platforms.

8. **No Operational Overhead**: Serverless model eliminates cluster management, patching, and infrastructure maintenance. Team focuses on analytics, not infrastructure.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Cost Uncertainty** | BigQuery charges per byte scanned; poorly written queries scanning entire table unexpectedly expensive | Use EXPLAIN queries to estimate cost, set query budget limits in BigQuery console, monitor monthly costs via billing dashboards, educate team on scan-efficient query patterns, use materialized views for expensive recurring queries |
| **Query Performance Degradation** | Very large tables (10+ years of data) can still slow down if queries not optimized | Implement time-based partitioning (by date), cluster on frequently filtered columns (sensor_id, enclosure_id), archive old data to Cloud Storage, regularly analyze query patterns and create indexes |
| **Data Duplication Risk** | Streaming inserts without deduplication could create duplicate sensor readings | Implement idempotent writes with deduplication keys (sensor_id + timestamp), use Dataflow to deduplicate before writing, set up data quality checks for duplicate detection |
| **Late Arrivals** | Sensor data delayed by network issues arrives after partition window closed | Use table partitioning with 2-3 day grace period, implement late-arrival handling in Dataflow (backfill previous partition), set up monitoring for out-of-order data |
| **Data Residency Constraints** | Certain regions may not have BigQuery availability | Select appropriate GCP region during setup, encrypt data at rest with customer-managed keys if required for compliance, test data residency setup before production |
| **Query Complexity** | Complex multi-domain analytics (ride data + animal data + visitor footfall) require complex joins | Design star schema with fact and dimension tables, pre-compute common joins as materialized views, document query patterns and optimization strategies |
| **Access Control Complexity** | Multiple teams (ride ops, animal team, management) need different data access levels | Use BigQuery's dataset-level IAM, create separate datasets for sensitive data, implement row-level security for PII, audit data access via Cloud Logging |
| **Integration Testing Difficulty** | Testing Dataflow + BigQuery pipeline requires realistic data volume | Use Cloud Storage for test data, snapshot production data subset to staging environment, implement data quality assertions in Dataflow tests |
| **Cold Query Performance** | First query on large dataset is slow; re-queries cached but cold queries expensive | Warm up tables by running periodic queries, set appropriate cache TTL, accept higher latency for rarely-run queries, optimize query structure for first-run performance |
| **Data Schema Evolution** | Adding new sensor types (new column) to existing sensor data requires schema updates | Use BigQuery's schema auto-detection for new columns, implement backward-compatible schema changes, plan for versioning of sensor data structures, document schema evolution process |

## Conclusion

BigQuery is the optimal choice for sensor data storage and analytics because:

1. **Columnar Storage for Fast Analytics**: Sensor analytics queries naturally scan specific columns (temperature, vibration, feed amount). BigQuery's columnar storage is 100x faster than row-oriented databases for these access patterns.

2. **Seamless Dataflow Integration**: Real-time data pipeline (sensors → MQTT → Pub/Sub → Dataflow → BigQuery) enables dashboards to see data within seconds of ingestion—critical for real-time animal health monitoring and ride safety tracking.

3. **Unlimited Scaling**: As the estate grows from 40 to 400+ attractions, data volume grows 10x. BigQuery queries remain fast regardless of dataset size. Traditional databases require expensive infrastructure scaling.

4. **Cost Efficiency**: Per-byte-scanned pricing means queries accessing 1% of data cost 1%. Archive historical data to Cloud Storage at 99% cost reduction. Traditional databases charge per-instance regardless of query efficiency.

5. **SQL for Operations Teams**: Standard SQL enables operations staff to write ad-hoc analytical queries without engineering support. No specialized query languages or steep learning curves.

6. **Real-Time Dashboards**: Looker Studio dashboards on live BigQuery data enable real-time monitoring. Animal health anomalies and ride safety issues surfaced immediately; not after batch processing overnight.

7. **Built-In Analytics**: BigQuery ML enables anomaly detection, forecasting, and clustering in-database. No data export, no separate ML platforms.

8. **Serverless Operations**: No cluster management, patching, or infrastructure maintenance. Team focuses on data analysis and insights, not infrastructure.

9. **Multi-Domain Analytics**: SQL joins across ride monitoring, animal health, and visitor analytics enable sophisticated insights (ride popularity correlated with animal stress, visitor flow patterns).

10. **Compliance and Audit**: BigQuery's audit logging, encryption, and dataset-level IAM meet regulatory requirements.

**Recommendation**: Deploy BigQuery as the central sensor data warehouse. Configure Dataflow pipelines to stream sensor data from Pub/Sub to BigQuery with real-time ingestion (sub-second latency). Partition tables by date (daily partitions) and cluster by sensor_id/enclosure_id for query efficiency. Implement materialized views for expensive recurring queries (daily aggregations, moving averages). Archive data older than 1 year to Cloud Storage and create external BigQuery tables for cold analytics. Use BigQuery ML for anomaly detection and trend forecasting. Enable audit logging and row-level security for access control. Monitor query performance via Cloud Monitoring and optimize slow queries monthly. Set BigQuery cost alerts to prevent budget surprises from poorly optimized queries.
