# ADR-013 — Choice of Google Cloud Dataflow for Stream Processing

## Context

The estate system generates continuous streams of data requiring real-time processing:

**Data Sources:**
- **Ride Sensors** (40 rides × 3 sensors/ride): 4,000-20,000 sensor events/day
- **Animal Sensors** (55 enclosures × 4 sensors/enclosure): 11,000-55,000 sensor events/day
- **Visitor Activity** (entry/exit events, gate scans): 5,000-15,000 entry events/day
- **Transaction Events** (orders, payments): 5,000-15,000 transactions/day
- **User Activity** (app clicks, page views): 50,000-150,000 events/day

**Processing Requirements:**
- **Real-Time Aggregations**: Calculate per-ride popularity (visitors/hour), per-enclosure average temperature (rolling 5-minute window)
- **Anomaly Detection**: Identify unusual sensor readings (temperature spike, vibration threshold breach, population change)
- **Threshold Alerts**: Trigger alerts when ride wear exceeds maintenance threshold or animal temperature deviates >2σ from average
- **Stream Enrichment**: Join sensor data with metadata (ride name, enclosure location, owner)
- **Windowed Analytics**: Calculate daily/hourly statistics (peak visitor hours, busiest rides, animal feeding compliance)
- **ML Predictions**: Predict maintenance needs based on vibration patterns; predict visitor behavior from historical trends

**Data Destinations:**
- **BigQuery**: Store processed events for analytics (100% of events)
- **Pub/Sub Topics**: Publish alerts for operations staff (anomalies, threshold breaches)
- **Pub/Sub for Notifications**: Trigger notification service for alerts
- **Cloud SQL**: Store critical alerts and incident records

**Processing Characteristics:**
- **High Throughput**: 50,000+ events/day = ~1 event/second average, 10-100 events/second during peak
- **Low Latency**: Alerts must surface within 1-5 minutes of anomaly (not batch overnight)
- **Complex Transformations**: Join streams, window calculations, ML scoring
- **Stateful Processing**: Maintain running averages, thresholds, and animal health baselines
- **At-Least-Once Semantics**: Avoid processing the same event twice; deduplication required

## Decision

**We will use Google Cloud Dataflow as the primary stream processing and ML engine.**

Cloud Dataflow provides managed, serverless stream processing that scales automatically, integrates natively with Pub/Sub and BigQuery, supports Apache Beam's unified batch/streaming model, and includes built-in ML capabilities—ideal for this workload.

## Key Differentiators

- **Serverless Architecture**  
  No cluster management, no capacity planning, no infrastructure provisioning. Dataflow scales automatically from zero to millions of events/second. Team writes pipeline code; Google handles distribution and scaling.

- **Unified Batch & Streaming**  
  Apache Beam supports both batch (historical data processing) and streaming (real-time events) with identical code. Can reprocess historical data using same pipeline; no duplication.

- **Native Pub/Sub Integration**  
  Dataflow reads directly from Pub/Sub (sensor events, transactions). No intermediate queuing or data export. Low-latency, high-throughput stream ingestion.

- **BigQuery Output**  
  Dataflow writes processed events directly to BigQuery with streaming inserts. Dashboard queries see data within seconds of processing. No batch ETL delays.

- **Windowing & Aggregations**  
  Apache Beam provides native windowing (tumbling, sliding, session). Compute per-ride popularity hourly, rolling temperature averages. Simpler than custom code.

- **Stateful Processing**  
  Beam's stateful DoFn enables maintaining state across events (animal health baselines, running averages). Complex analytics without external state store.

- **ML Integration**  
  Dataflow supports TensorFlow/scikit-learn models. Score events (e.g., "is this temperature anomalous?") within pipeline. ML results written directly to BigQuery.

- **Deduplication Support**  
  Beam's deduplication patterns prevent processing duplicate events (critical for financial correctness). Built-in, not an afterthought.

- **Error Handling & Dead-Letter Queues**  
  Failed processing routes to dead-letter queues (Cloud Storage, Pub/Sub) for investigation. No silent failures; all events accounted for.

- **Monitoring & Metrics**  
  Cloud Dataflow integrates with Cloud Monitoring. Track throughput, latency, error rates, resource usage. Alert on pipeline anomalies.

- **Cost-Effective Autoscaling**  
  Dataflow scales workers based on backlog. Peak hours might use 100 workers; off-peak hours 5 workers. Pay only for workers running; no idle capacity.

- **Multi-Language Support**  
  Apache Beam supports Python, Go, Java. Use language appropriate for team (Python for data science, Go for performance).

## Alternatives Considered

- **Apache Kafka Streams**  
  Open-source stream processing on Kafka:
  - **Self-Managed Complexity**: Requires Kafka cluster (brokers, zookeepers, managers), stream processors, and monitoring. Operational burden.
  - **Scaling Management**: Manual scaling; adding processing capacity requires rebalancing. Dataflow auto-scales.
  - **Resource Efficiency**: Dataflow's autoscaling more efficient than fixed Kafka infrastructure.
  - **Not GCP-Native**: Kafka doesn't integrate natively with BigQuery, Pub/Sub. Requires custom bridges.
  - **Not Recommended for Managed Service**: If going self-managed, high operational cost.

- **Apache Spark Streaming**  
  Spark's micro-batch streaming:
  - **Micro-Batch Latency**: Spark processes micro-batches (typically 500ms-5s); higher latency than true streaming (Dataflow <100ms).
  - **Not True Streaming**: Fundamentally different from event-by-event streaming; anomaly alerts delayed by micro-batch interval.
  - **Self-Managed Complexity**: Requires Spark cluster setup, YARN/Kubernetes orchestration, and monitoring. Operational burden.
  - **Not Serverless**: Dataflow serverless; Spark requires infrastructure.

- **AWS Kinesis**  
  AWS's streaming service:
  - **Cloud Lock-in**: Contradicts GCP platform choice (GKE, BigQuery). Multi-cloud complexity without benefit.
  - **Kinesis → BigQuery Integration**: Data must export from Kinesis to BigQuery; adds latency and complexity. Dataflow writes directly.
  - **Cost Model**: Kinesis per-shard pricing less efficient than Dataflow's autoscaling.
  - **Not Recommended**: GCP-only systems should use GCP-native services.

- **Apache Flink**  
  Distributed stream processing:
  - **Similar to Spark**: Flink can run streaming workloads but requires Flink cluster management.
  - **Self-Managed Overhead**: Not a managed service; team responsible for infrastructure.
  - **Smaller Ecosystem**: Fewer integrations with GCP services compared to Dataflow.
  - **Not Recommended for Managed Approach**: If cloud-native approach desired, choose managed service (Dataflow).

- **Custom Event Processing Service**  
  Building stream processing in-house using Python/Go consumers:
  - **High Complexity**: Implementing windowing, deduplication, state management, error handling, scaling from scratch is 3-6 months of engineering.
  - **Maintenance Burden**: Bugs in custom streaming code cause data loss or duplicate processing. Financial impact for ticketing system.
  - **Scaling Challenges**: Manual consumer group management, rebalancing, and backpressure handling complex.
  - **Not Recommended**: Pre-built solutions (Dataflow) eliminate reinventing wheel.

- **Cloud Functions + Pub/Sub**  
  Trigger Cloud Functions on Pub/Sub events:
  - **Stateless Only**: Cloud Functions are stateless; complex stateful operations (running averages, baselines) not supported.
  - **Timeout Limits**: Cloud Functions 9-minute timeout; long-running processing not supported.
  - **Aggregation Difficulty**: Windowing and aggregations over millions of events difficult without state.
  - **Not Recommended for Complex Processing**: Cloud Functions suitable for simple transformations; Dataflow for complex analytics.

- **Manual BigQuery Inserts**  
  Insert events directly into BigQuery from Pub/Sub consumers:
  - **Not Stream Processing**: No intermediate aggregations, anomaly detection, or alerts. Just raw data storage.
  - **BigQuery Load Limits**: Streaming inserts have rate limits; high throughput might hit limits. Dataflow batches intelligently.
  - **No Anomaly Detection**: Alerts require BigQuery queries run on schedule; not real-time. Dataflow detects anomalies within pipeline.
  - **Missing ML**: No built-in ML scoring; would require separate service.

## Why Cloud Dataflow is Better Than Alternatives

| Criterion | Dataflow | Kafka Streams | Spark Streaming | Kinesis | Flink | Custom |
|-----------|----------|---------------|-----------------|---------|-------|--------|
| **Stream Processing** | Excellent | Excellent | Good (micro-batch) | Good | Excellent | Risky |
| **Serverless** | Yes | No | No | Partial | No | No |
| **Autoscaling** | Automatic | Manual | Manual | Per-shard | Manual | Complex |
| **Latency** | <100ms | <100ms | 500ms-5s | 100-500ms | <100ms | Unknown |
| **BigQuery Integration** | Native | Via connector | Via connector | Via Kinesis Firehose | Via connector | Manual |
| **Pub/Sub Integration** | Native | No | No | No | No | Manual |
| **ML Support** | Built-in | No | Yes | No | No | No |
| **Windowing** | Excellent | Excellent | Good | Good | Excellent | Complex |
| **State Management** | Excellent | Excellent | Limited | Limited | Excellent | Complex |
| **Operational Burden** | Minimal | High | High | Minimal | High | Very High |
| **Cost Predictability** | Excellent (pay-per-use) | Fixed (infrastructure) | Fixed (infrastructure) | Per-shard | Fixed (infrastructure) | Unknown |
| **Setup Complexity** | Low (hours) | High (days) | High (days) | Medium (hours) | High (days) | Very High |

**Why Dataflow wins for estate system:**

1. **Serverless Scaling**: Automatically scales from 1 to 1,000+ workers based on throughput. No capacity planning, no infrastructure management.

2. **Real-Time Anomaly Detection**: <100ms latency means animal health/ride safety anomalies trigger alerts within seconds. Kafka Streams/Flink similar, but operational burden higher.

3. **Native GCP Integration**: Pub/Sub ingestion, BigQuery output, Cloud Monitoring metrics. No custom bridges or data export/import.

4. **Unified Batch & Streaming**: Same code processes historical data (backfill) and real-time events. Reduces duplication and testing burden.

5. **Built-In ML**: TensorFlow/scikit-learn models score events within pipeline. Anomaly detection scores computed in-stream.

6. **Complexity vs. Managed Tradeoff**: Dataflow's serverless model eliminates Kafka/Flink cluster management overhead. Team focuses on pipeline logic, not infrastructure.

7. **Cost-Efficient Autoscaling**: Pay only for workers used. 10-worker average during peak, 1 worker off-peak. No idle capacity charges.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Pipeline Complexity** | Complex Beam pipelines (multiple stages, state, windowing) can be difficult to debug | Start with simple pipelines (single-stage transformation); add complexity incrementally, test each stage independently via unit tests, implement detailed logging |
| **Late Data Handling** | Events arrive late (delayed by network); must decide whether to include in windows or drop | Use allowed lateness parameter to include late events (e.g., 1-hour grace period), implement side-outputs for very late data, accept some data loss for operational alerts |
| **Cost Uncertainty** | Dataflow autoscaling could create unexpectedly high bills if pipelines inefficient | Monitor pipeline resource usage via Cloud Monitoring, set quotas on worker count, optimize pipelines before scale testing, implement cost alerts |
| **State Storage Overhead** | Maintaining state (animal baselines, temperature averages) uses Dataflow state backend; state backend latency adds overhead | Use efficient state storage (local state backend preferred), avoid excessive state per-window, size state backend appropriately, benchmark state access patterns |
| **Duplicate Event Risk** | Exactly-once semantics hard to guarantee in distributed streaming; duplicates possible | Implement deduplication logic (event_id deduplication window), use BigQuery's deduplication (deduplicate_rows), track duplicate count metrics |
| **Monitoring Complexity** | Beam pipelines span multiple stages; understanding which stage failed/slow requires good instrumentation | Implement counters per transform (elements processed, errors), use custom metrics, set alerts on lag/error rates, maintain pipeline documentation |
| **Version Upgrades** | Apache Beam version upgrades might introduce breaking changes; careful testing required | Test Beam upgrades in staging environment, maintain backward-compatible pipeline code, pin Beam version in production |
| **Data Skew** | Some sensor types (popular rides) generate far more data; parallelism becomes uneven; some workers overloaded | Implement custom partitioning/grouping to balance load, use Dataflow autoscaling aggressively for skewed data, monitor worker CPU distribution |
| **Exactly-Once vs. At-Least-Once Tradeoff** | Dataflow defaults to at-least-once; exactly-once requires careful design | For non-idempotent operations (billing, inventory), implement deduplication keys, for analytics (aggregations), accept at-least-once and deduplicate in BigQuery |
| **Cold Start Latency** | New Dataflow job startup takes 1-2 minutes; not suitable for ultra-low-latency scenarios | Keep Dataflow job running continuously (don't restart), use job autoscaling instead of stop/start, accept 1-2 minute lag for initial pipeline deployment |

## Conclusion

Google Cloud Dataflow is the optimal stream processing platform for the estate system because:

1. **Serverless Autoscaling**: Automatically scales to handle 50,000+ daily events efficiently. No infrastructure management.

2. **Real-Time Processing**: <100ms latency means anomalies trigger alerts immediately. Critical for safety and operational response.

3. **Native GCP Integration**: Pub/Sub ingestion, BigQuery output, Cloud Monitoring metrics. No custom bridges or data export/import overhead.

4. **Complex Transformations**: Windowing, stateful operations, deduplication, joins all built-in. Simpler than custom code.

5. **ML Integration**: TensorFlow/scikit-learn models score events within pipeline. Anomaly detection computed in-stream, not post-hoc.

6. **Cost-Efficient**: Autoscaling means 10-worker average peak, 1 worker off-peak. Pay only for resources used.

7. **No Operational Burden**: Fully managed. No cluster setup, patching, monitoring infrastructure. Team focuses on pipeline logic.

**Recommendation**:
- Deploy Dataflow pipelines for each domain:
  - **Ride Monitoring Pipeline**: Read ride sensors from Pub/Sub, compute per-ride hourly popularity, detect wear anomalies, write to BigQuery, publish critical alerts to Pub/Sub
  - **Animal Health Pipeline**: Read animal sensors, compute per-enclosure rolling averages, detect temperature/feeding anomalies, write to BigQuery, publish health alerts
  - **Visitor Analytics Pipeline**: Read entry/exit events, compute footfall by zone/hour, write to BigQuery for dashboards
  - **Payment Pipeline**: Read payment events, deduplicate, compute transaction success rate, write to BigQuery
- Use Apache Beam Python SDK (matches data science team expertise)
- Implement deduplication for all pipelines (event_id windowed deduplication)
- Set allowed lateness to 1 hour (accept late events within window)
- Monitor pipeline lag, error rates, and worker CPU via Cloud Monitoring
- Test pipelines with production-volume data before going live
- Use BigQuery as destination; Pub/Sub for real-time alerts
- Implement anomaly detection via Beam's stateful DoFn (ML models, baselines)
- Run pipelines continuously; scale workers via autoscaling, not stop/start
- Set cost alerts (max workers, max estimated cost/hour)
