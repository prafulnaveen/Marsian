# ADR-010 — Choice of Google Cloud Logging and Cloud Monitoring for Observability

## Context

The estate system comprises 30+ microservices generating continuous observability data:

**Logging Requirements:**
- Application logs from all microservices (booking, payment, validation, ticketing)
- Audit logs for compliance (payment processing, user access, data changes)
- Security logs (authentication failures, unauthorized access attempts, API abuse)
- Infrastructure logs (GKE events, node issues, container crashes)
- Sensor ingestion logs (MQTT events, Pub/Sub latency, Dataflow processing)
- Log volume: 10,000-100,000+ log entries per minute at peak load

**Monitoring Requirements:**
- Real-time alerts for critical events (payment failures, ride safety anomalies, animal health alerts)
- Performance metrics (API latency, database query times, cache hit rates)
- Resource utilization (GKE CPU/memory, BigQuery slot usage, Redis memory)
- Business metrics (ticket sales rate, payment success rate, visitor count)
- Custom metrics (ride uptime percentage, average visit duration, animal feeding completion rate)
- Dashboards for operations teams (system health, SLA tracking)

**Scale Requirements:**
- GKE cluster with 50+ pods (across 3 domains)
- BigQuery ingesting 50,000+ sensor events/day
- Cloud SQL with 10,000+ transactions/hour during peak
- Pub/Sub with 1,000+ messages/second
- 24/7 operation; no downtime tolerance
- Multi-region future expansion

**Data Sources:**
- Application logs (stdout/stderr from containers)
- Structured metrics (Prometheus format)
- Custom events (business metrics, anomalies)
- GCP service logs (GKE, Cloud SQL, Cloud Storage)

## Decision

**We will use Google Cloud Logging and Cloud Monitoring as the unified observability platform.**

Cloud Logging and Cloud Monitoring are fully managed, native GCP services that integrate seamlessly with GKE, Cloud SQL, BigQuery, and other GCP services. They provide centralized log aggregation, real-time alerting, and custom dashboards without infrastructure management.

## Key Differentiators

- **Native GCP Integration**  
  Cloud Logging and Cloud Monitoring collect logs and metrics from all GCP services automatically. No agent installation or configuration needed. GKE pods export logs natively; Cloud SQL exports metrics automatically. Unified collection across entire stack.

- **Automatic Log Collection from GKE**  
  GKE automatically routes container stdout/stderr to Cloud Logging. No application configuration needed; logging happens transparently. Logs available within seconds of emission.

- **Centralized Log Storage and Search**  
  All logs from 30+ microservices, infrastructure, and GCP services stored in central repository. Search logs by service, pod, customer, order ID, etc. Complex queries using GCP log filter syntax.

- **Real-Time Alerts**  
  Cloud Monitoring alert policies trigger immediately when metrics breach thresholds. Email, SMS, PagerDuty, Slack integrations enable incident response. Critical alerts (payment failures, ride safety issues) surfaced within seconds.

- **Custom Metrics**  
  Applications emit custom metrics (ticket sales rate, ride utilization percentage, animal temperature anomalies). Cloud Monitoring collects and visualizes custom metrics alongside infrastructure metrics.

- **Long-Term Log Retention**  
  Cloud Logging stores logs for 30 days by default; can be extended. Logs archived to Cloud Storage for long-term compliance and historical analysis.

- **Log-Based Metrics**  
  Derive metrics from logs automatically. Example: create metric "failed_payment_count" by counting logs containing "payment_error". Convert log data to metrics without code changes.

- **Grafana/Prometheus Compatibility**  
  Cloud Monitoring exports metrics in Prometheus format. Existing Prometheus/Grafana users can integrate with Cloud Monitoring. Migrate to Cloud Monitoring gradually without rip-and-replace.

- **SLA Monitoring**  
  Define SLIs (Service Level Indicators) and SLOs (Service Level Objectives). Cloud Monitoring calculates SLO compliance automatically. Track uptime percentage, error budgets.

- **Audit Logging**  
  Cloud Audit Logs track all API calls, data access, and permission changes in GCP. Compliance audits see who accessed what data, when, from where.

- **Incident Response Integration**  
  Alert policies integrate with PagerDuty, Opsgenie, Slack. Incidents escalate automatically; team notified within seconds of threshold breach.

- **Cost Predictability**  
  Cloud Logging pricing per GB ingested; Cloud Monitoring pricing per metric series. Transparent, scalable pricing without surprise spikes.

- **No Infrastructure Management**  
  Fully managed service; no cluster setup, no agent management, no log rotation, no disk space management. Team focuses on observability configuration, not infrastructure.

## Alternatives Considered

- **ELK Stack (Elasticsearch, Logstash, Kibana)**  
  Open-source log aggregation platform. However:
  - **Self-Hosted Complexity**: Requires Elasticsearch cluster deployment, Logstash configuration, and Kibana setup. Operational burden.
  - **Infrastructure Scaling**: As log volume grows (10,000+ logs/minute), Elasticsearch cluster must scale. Managing sharding, replication, and node balancing complex.
  - **Cost at Scale**: Elasticsearch licensing and infrastructure cost grows significantly. Self-hosted Elasticsearch for 100K logs/minute can cost $2,000+/month.
  - **Maintenance Burden**: Elasticsearch upgrades, security patches, performance tuning require expertise.
  - **GCP Integration**: Limited compared to Cloud Logging's native integration with GKE, Cloud SQL.

- **Splunk**  
  Enterprise log management platform. However:
  - **Very High Cost**: Splunk per-GB ingestion pricing ($6-12/GB) extremely expensive for high-volume log streams. 100K logs/minute = ~150GB/month = $900-1,800/month.
  - **Overkill for This Use Case**: Splunk designed for complex enterprise security and compliance scenarios. Estate system needs simpler observability.
  - **Long Setup Time**: Splunk deployments typically take weeks; significant upfront configuration and training.
  - **Licensing Complexity**: Splunk licensing confusing; cost surprises common.

- **DataDog**  
  SaaS monitoring platform. However:
  - **High Cost**: DataDog per-host + per-metric pricing becomes expensive at scale. 50+ GKE pods + 100+ metrics = $1,500+/month.
  - **Multi-Cloud Focus**: DataDog optimized for multi-cloud (AWS, Azure, GCP). GCP-only users pay premium for multi-cloud features.
  - **Overkill**: DataDog powerful but designed for complex SRE workflows. Estate system needs simpler observability.
  - **Not GCP Native**: Additional agent/configuration needed compared to Cloud Logging's automatic collection.

- **New Relic**  
  Full-stack observability platform. However:
  - **Complex Pricing**: New Relic's pricing model confusing; cost unpredictable as usage grows.
  - **High Cost**: Similar to DataDog; per-host and per-metric pricing expensive for large deployments.
  - **Not GCP Native**: Less integrated with GCP services compared to Cloud Logging/Monitoring.

- **Prometheus + Grafana (Self-Hosted)**  
  Open-source monitoring stack. However:
  - **No Log Aggregation**: Prometheus designed for metrics only; doesn't aggregate application logs. Requires separate ELK/Splunk for logs.
  - **Limited Retention**: Prometheus stores metrics locally; retention typically 15 days. Long-term trend analysis requires external storage.
  - **Self-Hosted Complexity**: Requires Prometheus cluster, Grafana deployment, and alertmanager configuration.
  - **Manual Scaling**: Adding retention, increasing scrape frequency, or adding dashboards requires manual infrastructure scaling.
  - **Missing Audit Logs**: Prometheus/Grafana don't capture audit logs (API access, permission changes) required for compliance.

- **AWS CloudWatch (Comparison to AWS)**  
  AWS's monitoring service. However:
  - **Cloud Lock-in**: Selecting AWS CloudWatch contradicts GCP platform choice (GKE, Cloud SQL, BigQuery). Multi-cloud complexity without benefit.
  - **Cross-Cloud Data Transfer**: GKE (GCP) sending logs to CloudWatch (AWS) incurs data egress cost and latency.
  - **Operational Inconsistency**: Different authentication models (AWS IAM vs. GCP IAM), different metric formats, different tooling. Team must learn two platforms.

- **Azure Monitor (Comparison to Azure)**  
  Azure's observability platform. However:
  - **Same Issues as AWS CloudWatch**: Platform lock-in, cross-cloud complexity, operational inconsistency.

- **Jaeger / Distributed Tracing Only**  
  Distributed tracing platform. However:
  - **Tracing Only**: Jaeger designed for request tracing, not general log aggregation or metrics collection.
  - **Incomplete Observability**: Doesn't capture infrastructure metrics, business metrics, or system-level events.
  - **Should Complement, Not Replace**: Use Jaeger for detailed request tracing in addition to Cloud Logging/Monitoring.

- **Dynatrace**  
  Full-stack application monitoring. However:
  - **Very High Cost**: Dynatrace per-host pricing among the most expensive. Prohibitive for 50+ GKE pods.
  - **Enterprise-Focused**: Designed for large enterprises; pricing/features overkill for mid-market system.

## Why Cloud Logging & Monitoring Are Better Than Alternatives

| Criterion | Cloud Logging/Monitoring | ELK Stack | Splunk | DataDog | Prometheus+Grafana |
|-----------|------------------------|-----------|--------|---------|-------------------|
| **Log Aggregation** | Excellent | Excellent | Excellent | Very Good | None (metrics only) |
| **Metrics Collection** | Excellent | Limited | Excellent | Excellent | Excellent |
| **GCP Integration** | Native | Limited | Limited | Fair | Limited |
| **Setup Complexity** | Low (hours) | High (days) | Very High (weeks) | Medium (days) | Medium (days) |
| **Infrastructure Burden** | None (managed) | High | High | None | Medium (self-hosted) |
| **Cost (100K logs/min + 50 pods)** | $300-500/mo | $500-1000/mo | $1500+/mo | $1500+/mo | $200-400/mo (self-hosted) |
| **Real-Time Alerts** | Excellent (seconds) | Good (minutes) | Good (minutes) | Excellent | Fair (requires config) |
| **Long-Term Retention** | Excellent (30+ days) | Good (configurable) | Excellent | Excellent | Poor (15 days local) |
| **Audit Logging** | Excellent (Cloud Audit) | Limited | Good | Good | None |
| **Query Language** | GCP Log Filter | Kibana Query | SPL | PromQL + UI | PromQL |
| **Mobile Alerts** | Excellent (integration) | Fair | Good | Excellent | Fair |
| **Scalability** | Unlimited (managed) | Complex (cluster) | Complex (licensing) | Automatic | Requires manual scaling |

**Why Cloud Logging & Monitoring win for estate system:**

1. **Native GCP Integration**: Automatic log collection from GKE, Cloud SQL, Cloud Storage, Pub/Sub. No agent installation, no configuration. Zero-touch observability.

2. **Unified Platform**: Logs and metrics in single platform. No context switching between multiple tools. Correlate logs and metrics easily.

3. **Real-Time Alerts**: Payment failures, ride safety anomalies, animal health alerts surface within seconds. Critical for operations.

4. **Cost-Efficient**: Fixed pricing per GB logs + per metric series scales predictably. No per-pod or per-host licensing surprises.

5. **Compliance**: Cloud Audit Logs track all API calls and data access. Meets regulatory requirements without separate compliance tool.

6. **No Infrastructure Management**: Fully managed service. Team focuses on observability strategy, not infrastructure.

7. **Long-Term Retention**: 30+ day log retention in Cloud Logging; archive to Cloud Storage for years of historical analysis.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Cost Uncertainty** | High log volume could exceed budget if not monitored | Set monthly log ingestion budgets, implement log filtering to exclude verbose/low-value logs, monitor ingestion rate via dashboards, adjust sampling if needed |
| **Log Latency** | Logs might take 1-2 minutes to appear in Cloud Logging during peak ingestion | Accept brief latency for historical analysis; implement low-latency alerting via Pub/Sub for real-time events, test SLAs under peak load |
| **Log Retention Limits** | Free tier retains logs only 30 days; older logs require archival to Cloud Storage | Archive logs older than 30 days to Cloud Storage (cheaper), configure retention policy, test retrieval from Cloud Storage |
| **Query Performance Degradation** | Complex queries on very large log datasets (years of logs) could be slow | Use Cloud Logging's indexed fields for fast filtering, avoid full-text search on very large datasets, use Cloud Storage for cold log queries |
| **Alert Fatigue** | Too many alert policies could overwhelm team with notifications | Set thresholds conservatively (use p99 latency, not p50), disable non-critical alerts during maintenance windows, implement alert correlation to reduce duplicates |
| **Data Privacy** | Logs might contain sensitive data (customer emails, payment info, animal details) | Implement log redaction (remove PII before logging), encrypt sensitive logs, restrict log access via IAM, audit log access regularly |
| **Multi-Cloud Observation** | If future expansion to AWS/Azure, Cloud Logging doesn't observe non-GCP services | Plan for multi-cloud strategy early, implement application-level metrics export, maintain separate observability per cloud if needed |
| **Learning Curve for Log Queries** | GCP Log Filter syntax different from SQL or other query languages | Provide query templates, document common log filter patterns, train team on Log Filter syntax |
| **Alert Routing Complexity** | Multiple stakeholders (ops, dev, management) need different alerts via different channels | Use alert policy routing to send alerts to appropriate channels (Slack for dev team, email for management, PagerDuty for ops) |
| **Visualizing Complex Metrics** | Some complex dashboards might need custom visualization not supported by Cloud Monitoring UI | Use Looker Studio to create custom dashboards on top of Cloud Monitoring metrics, accept Cloud Monitoring's standard visualizations for infrastructure metrics |

## Conclusion

Cloud Logging and Cloud Monitoring are the optimal observability platforms for the estate system because:

1. **Native GCP Integration**: Automatic log/metric collection from GKE, Cloud SQL, Pub/Sub, BigQuery. Zero-configuration observability.

2. **Unified Platform**: Logs and metrics in single service. Correlate data easily without switching tools.

3. **Real-Time Alerts**: Payment failures, safety anomalies, and health alerts trigger within seconds. Critical for operations.

4. **Cost Predictable**: Per-GB and per-metric-series pricing scales linearly. No surprise licensing spikes.

5. **Compliance & Audit**: Cloud Audit Logs capture all API activity and data access for regulatory compliance.

6. **Long-Term Retention**: 30-day live retention + Cloud Storage archival enables years of historical analysis.

7. **Infrastructure-Free**: Fully managed. No cluster setup, scaling, or maintenance. Team focuses on observability strategy.

8. **Security Built-In**: Encryption at rest/in-transit, IAM-based access control, audit logging.

**Recommendation**: 
- Enable Cloud Logging auto-ingestion from GKE, Cloud SQL, Pub/Sub
- Create alert policies for critical business metrics (payment success rate, ride availability, animal health)
- Archive logs older than 30 days to Cloud Storage for long-term retention and compliance
- Build Cloud Monitoring dashboards for operations team (SLO tracking, SLA compliance)
- Integrate alerts with Slack (dev team), email (management), PagerDuty (on-call)
- Use BigQuery to analyze historical logs for trend analysis and anomaly detection
- Implement log redaction to remove PII before ingestion
- Set monthly log ingestion budgets and monitor against budget
- Test alert routing and incident response workflows monthly
