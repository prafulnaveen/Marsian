# ADR-009 — Choice of Looker Studio for Dashboarding and Visualization

## Context

The estate system requires dashboards for multiple stakeholder groups:

**Estate Management Dashboards:**
- Daily visitor volume and forecasts
- Revenue by ticket type and family pass
- Seasonal trends (peak/off-season patterns)
- Loyalty program engagement and retention metrics
- Revenue forecasts and goal tracking

**Operations Dashboards:**
- Ride utilization and popularity by attraction type
- Ride maintenance alerts and wear trends
- Animal health status (temperature, feeding, population)
- Animal behavioral anomalies and feeding issues
- Enclosure environmental conditions
- Staff on-call scheduling and alerts

**Real-Time Monitoring Dashboards:**
- Live visitor footfall by zone/ride (for capacity management)
- Real-time ride status (operational/maintenance/closed)
- Animal health critical alerts (immediate notification needed)
- System health (data ingestion lag, stream processing latency)

**Data Source:**
- BigQuery (sensor data, visitor analytics, historical trends)
- Cloud SQL (transactional data, orders, payments)
- Pub/Sub (real-time events, streaming updates)
- Dataflow (stream processing results)

**Requirements:**
- Real-time data visualization (refresh every 5-10 minutes)
- Multi-user access control (different stakeholders see different dashboards)
- Mobile-responsive design (staff viewing on tablets/phones)
- Drill-down capability (click on ride to see detailed data)
- Alert capability (highlight anomalies, threshold breaches)
- No coding required for dashboard updates (non-technical staff can adjust)
- Low cost at scale (100+ concurrent users)

## Decision

**We will use Google Looker Studio as the primary dashboarding and visualization platform.**

Looker Studio provides native BigQuery integration, real-time querying, multi-user access control, mobile-responsive dashboards, and zero-cost-to-scale architecture—ideal for this analytics use case.

## Key Differentiators

- **Native BigQuery Integration**  
  Looker Studio connects directly to BigQuery without intermediaries. Queries execute in BigQuery's powerful engine; results stream to dashboards in real-time. No data export, no intermediate data movement. Looker Studio is natively optimized for BigQuery's columnar performance.

- **Real-Time Data Refresh**  
  Dashboards refresh every 5-10 minutes by default; can be configured for more frequent updates. Real-time visitor footfall and animal health alerts surface immediately—critical for operations teams making real-time decisions.

- **Zero Scaling Cost**  
  Looker Studio pricing is per-viewer (free for view-only, $12/month for editors). Adding 100 users doesn't increase infrastructure cost. Dashboards scale to thousands of concurrent viewers without degradation.

- **Mobile-Responsive Design**  
  Looker Studio dashboards are fully responsive; work seamlessly on phones and tablets. Estate staff viewing on iPads during rounds see properly formatted dashboards.

- **Multi-User Access Control**  
  Looker Studio integrates with Google IAM for granular access control. Can share dashboards with specific users, teams, or public. Different roles see different data (ride ops see ride dashboards, keepers see animal health).

- **GCP-Native Authentication**  
  Looker Studio users authenticate via Google accounts; integrates with Cloud IAM and Workspace. No separate user management system.

- **Flexible Data Visualization**  
  Looker Studio provides 20+ chart types (scorecards, time-series, heatmaps, tables, maps, etc.). Easily create custom visualizations for different use cases.

- **Drill-Down and Filtering**  
  Dashboards support interactive filters (date range, zone, animal type, etc.). Click on chart elements to drill down (click "Ride A" to see hourly breakdown).

- **Derived Metrics**  
  Define calculated fields directly in Looker Studio (e.g., "temperature anomaly = abs(current_temp - average_temp) > 2*std_dev"). No need to pre-compute in BigQuery.

- **Alerting Capabilities**  
  Looker Studio conditionally formats charts (highlight cells when thresholds breached). Can add email alerts (via scheduled reports) when metrics exceed thresholds.

- **Scheduled Report Export**  
  Generate PDF/email reports on schedule (daily sales report, weekly animal health summary) sent to stakeholders automatically.

- **No Code Required**  
  Non-technical staff can create new charts by dragging and dropping fields; no SQL required. Makes dashboards maintainable by operations team, not just data engineers.

- **Embedded Dashboards**  
  Embed Looker Studio dashboards directly into web applications (estate management portal, staff admin console). Users see integrated visualizations without leaving the app.

## Alternatives Considered

- **Tableau**  
  Enterprise business intelligence platform. However:
  - **High Cost**: Tableau pricing is per-user (~$70/month); 100 users = $7,000/month. Looker Studio 100 users = $1,200/month. 6x more expensive.
  - **Overkill for This Use Case**: Tableau is designed for complex analytical workflows. Estate dashboards are simpler; Tableau's power is underutilized.
  - **Longer Setup Time**: Tableau requires more configuration and enterprise IT involvement. Looker Studio dashboards created in hours, not days.
  - **Cloud Lock-in**: Tableau Cloud is available but less integrated with GCP than Looker Studio.

- **Power BI (Microsoft)**  
  Microsoft's business intelligence platform. However:
  - **Platform Lock-in**: Microsoft ecosystem (Azure, Excel, Active Directory) not aligned with GCP platform choice.
  - **Cross-Cloud Complexity**: GKE (GCP) to Power BI (Azure/Microsoft) adds cross-cloud latency and complexity.
  - **Cost**: Power BI Pro per user (~$10/month) cheaper than Tableau but more expensive than Looker Studio for large user bases (100 users = $1,000/month vs. $1,200 for Looker Studio Premium).
  - **Learning Curve**: Teams familiar with GCP would need Microsoft ecosystem training.

- **Apache Superset**  
  Open-source visualization platform. However:
  - **Self-Hosted Complexity**: Requires Superset cluster deployment, maintenance, upgrades, and monitoring. Operational burden.
  - **Limited BigQuery Integration**: Superset connects to BigQuery but optimization less mature than Looker Studio (native Google products).
  - **Mobile Experience**: Mobile responsiveness not as polished as Looker Studio.
  - **Community Support**: Smaller community; fewer third-party integrations than commercial platforms.
  - **Scalability**: Hosting thousands of dashboard viewers requires Superset infrastructure scaling; Looker Studio scales infinitely without extra cost.

- **Metabase**  
  Open-source analytics platform. However:
  - **Similar to Superset**: Same self-hosted complexity, less mature BigQuery integration, smaller community.
  - **Simpler Setup**: Easier to set up than Superset, but still requires hosting and maintenance.
  - **Not Recommended**: Benefits of self-hosting don't justify operational burden for this use case.

- **Amazon QuickSight**  
  AWS business intelligence service. However:
  - **Cloud Lock-in**: Contradicts GCP platform choice (GKE, BigQuery); multi-cloud complexity without benefit.
  - **Data Movement**: BigQuery (GCP) to QuickSight (AWS) requires data export; adds latency and cost.
  - **Cost**: QuickSight per-user pricing similar to Power BI; more expensive than Looker Studio.
  - **Network Overhead**: Cross-cloud data transfer incurs egress costs.

- **Grafana**  
  Open-source monitoring and visualization. However:
  - **Monitoring-Focused**: Grafana optimized for infrastructure metrics and time-series monitoring. Better fit for Datadog/Prometheus data than BigQuery analytics.
  - **Analytics Capability**: Less suited for business analytics queries (revenue, customer segmentation) compared to Looker Studio.
  - **Self-Hosted**: Same operational burden as Superset/Metabase.
  - **BigQuery Integration**: Limited compared to Looker Studio.

- **Qlik Sense**  
  Associative analytics platform. However:
  - **High Cost**: Qlik pricing per-user (~$20-40/month); very expensive for large user bases.
  - **Complex Licensing**: Licensing models confusing; hard to predict cost scaling.
  - **Steep Learning Curve**: Qlik's data model and associative engine require specialized training.
  - **Overkill**: Qlik's advanced features (associative navigation, data discovery) underutilized for straightforward estate dashboards.

- **In-House Custom Dashboards**  
  Building dashboards using React/Vue.js + BigQuery API:
  - **High Development Cost**: Significant engineering effort to build responsive, mobile-friendly dashboards with real-time updates.
  - **Maintenance Burden**: Updates to dashboards require code changes and testing. Non-technical staff can't adjust without engineering support.
  - **Scaling Complexity**: Caching strategies, connection pooling, and real-time update mechanisms add complexity.
  - **Not Recommended**: Pre-built solutions (Looker Studio) are purpose-built for this; custom development poor ROI.

## Why Looker Studio is Better Than Alternatives

| Criterion | Looker Studio | Tableau | Power BI | Superset | QuickSight |
|-----------|---------------|---------|----------|----------|-----------|
| **BigQuery Native Integration** | Excellent | Good | Fair (via connector) | Fair | Poor (requires export) |
| **Real-Time Refresh** | 5-10 min (native) | 1 hour (default) | 1 hour (default) | Configurable | 1 hour (default) |
| **Cost (100 users)** | $1,200/mo | $7,000+/mo | $1,000/mo | $0 (self-hosted) | $1,500+/mo |
| **Mobile Responsiveness** | Excellent | Good | Good | Fair | Good |
| **Ease of Use** | Excellent (drag-drop) | Good (learning curve) | Good (learning curve) | Fair (technical) | Good |
| **Multi-User Access Control** | Excellent (GCP IAM) | Excellent | Excellent | Good | Fair |
| **Setup Complexity** | Low (hours) | High (days-weeks) | Medium (days) | Medium (self-host) | Medium (hours) |
| **Scalability to 1000+ Users** | Excellent (cost flat) | Complex (licensing) | Complex (licensing) | Requires infrastructure | Complex (licensing) |
| **Alerting Capability** | Good (conditional formatting, scheduled emails) | Excellent (rich alerts) | Excellent (rich alerts) | Fair (limited) | Good |
| **GCP Platform Alignment** | Native | No | No | Neutral | No |
| **Embedded in Custom Apps** | Supported | Supported | Supported | Supported | Supported |

**Why Looker Studio wins for estate dashboards:**

1. **Native BigQuery Integration**: Looker Studio and BigQuery are both Google products. Tightest integration, best performance, no data movement.

2. **Cost-Efficient at Scale**: Fixed per-viewer cost means 100 users costs same as 10 users. Tableau/Power BI become expensive quickly.

3. **Real-Time Data**: 5-10 minute refresh reflects current conditions. Visitor footfall, animal health, ride status visible immediately for operational decisions.

4. **GCP Ecosystem Fit**: Seamless authentication via Cloud IAM, Workspace integration, zero additional infrastructure.

5. **Non-Technical Authorship**: Drag-and-drop interface means operations staff can create/update dashboards without data engineering support.

6. **Mobile Experience**: Responsive design means iPad-carrying keepers see properly formatted dashboards in field.

7. **Zero Scaling Cost**: Adding users doesn't increase infrastructure cost; Looker Studio handles scaling.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Limited Advanced Analytics** | Looker Studio isn't designed for complex statistical modeling or machine learning workflows | Use BigQuery ML for ML models, visualize results in Looker Studio dashboards, accept Looker Studio limitation as visualization layer (not analysis layer) |
| **Real-Time Latency** | BigQuery queries 5-10 minute refresh slower than Pub/Sub streaming for ultra-low latency requirements | For critical real-time alerts (animal critical temperature), implement separate alert system (Dataflow thresholds → Pub/Sub → alert service), use Looker Studio for trend/historical analysis |
| **Complex Custom Calculations** | Some domain-specific metrics difficult to express in Looker Studio's calculated fields | Pre-compute complex metrics in BigQuery materialized views, reference pre-computed fields in Looker Studio |
| **Data Access Control Limitations** | Looker Studio's row-level security limited; can't easily restrict "keepers see only their own animals" at row level | Implement row-level security in BigQuery itself (create separate tables by role), share appropriate table with user's Looker Studio dashboard |
| **Formatting Limitations** | Looker Studio's visual customization limited compared to custom React dashboards | Accept Looker Studio's standard look/feel, use custom branding (logos, colors) where supported, build custom React dashboard for highly specialized needs |
| **Dashboard Performance at Scale** | Complex dashboards with many charts querying large tables can be slow | Optimize BigQuery queries (use materialized views, precompute aggregations), limit dashboard to 10-12 key charts (avoid dashboards with 50+ charts) |
| **Version Control** | Looker Studio dashboards not easily version controlled; changes not tracked in Git | Document dashboard schema separately (which tables, metrics, calculated fields), implement change log manually, maintain backups |
| **Learning Curve for SQL** | Some power users need to write SQL for custom metrics; Looker Studio's SQL editor is basic | Provide SQL training, maintain library of common SQL snippets, encourage use of BigQuery for complex queries |
| **Export Limitations** | Downloading dashboard data limited to CSV; no direct integration with external analytics tools | Implement Pub/Sub export for custom pipelines if needed, use BigQuery as source for external tools, accept Looker Studio as final destination |
| **Branding Options** | White-labeling limited for embedded dashboards; appears as "Powered by Google" | Accept Looker Studio branding, use custom domain for professional appearance, implement custom wrapper if branding critical |

## Conclusion

Looker Studio is the optimal dashboarding platform for the estate system because:

1. **Native BigQuery Integration**: Real-time queries on BigQuery data without data movement or export. Best performance and lowest latency.

2. **Cost-Efficient at Scale**: Per-viewer pricing model means adding 100 users costs same as 10 users. Flat scaling cost makes enterprise dashboards affordable.

3. **GCP-Native**: Authentication via Cloud IAM, no separate user management, seamless Workspace integration.

4. **Real-Time Data**: 5-10 minute refresh ensures dashboards reflect current conditions for operational decision-making.

5. **Non-Technical Authorship**: Drag-and-drop interface enables non-technical staff to create/update dashboards without data engineer involvement.

6. **Mobile-Responsive**: Dashboards work seamlessly on tablets/phones, enabling field staff (keepers, ride ops) to view live data during rounds.

7. **Multi-User Access Control**: Fine-grained sharing and IAM integration enables different stakeholders to see relevant data only.

8. **Zero Infrastructure Overhead**: Fully managed service; no cluster management, scaling, or maintenance required.

**Recommendation**: 
- Build core management dashboards (revenue, visitor trends, loyalty) for executive team
- Build operations dashboards (ride status, animal health) for ride ops and animal keepers
- Build real-time monitoring dashboard for System Operations Center (live alerts)
- Use BigQuery materialized views for expensive recurring queries
- Implement BigQuery row-level security for data access control
- Refresh dashboards every 5-10 minutes for operational freshness
- Use Looker Studio's scheduled reports for daily/weekly email summaries
- Embed dashboards in admin console for integrated experience
- Train operations staff on Looker Studio's drag-drop interface for self-service dashboard updates
