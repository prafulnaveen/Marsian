# Data Ingestion Pipeline

> This document describes a batch-first data flow from operational systems into the central Data Warehouse. Data is collected during the day and processed in a scheduled nightly window to control platform cost. Safety-critical alerts remain an explicit near-real-time exception.

---

## 1. Ingestion Architecture Overview

```mermaid
graph TB
    subgraph EDGE["On-Estate Edge Layer"]
        MQTT["MQTT Broker / Gateway"]
        RIDESENSOR["Ride Sensors"]
        ENCSENSOR["Enclosure Sensors"]
        GATESENSOR["Gate/Footfall Sensors"]
        TICKETGW["Ticketing System\n(local queue)"]
    end

    subgraph CLOUD_INGEST["Cloud Ingestion Layer"]
        INGESTAPI["Data Ingestion Service"]
      BATCH["Nightly Batch Uploader\n(compress, manifest, retry)"]
      PUBSUB["Google Cloud Pub/Sub\n(batch triggers, fan-out)"]
      ALERTPUBSUB["Pub/Sub Alert Topic\n(safety exceptions only)"]
      DATAFLOW["Batch Processor\n(GCP Dataflow)"]
    end

    subgraph DW["Central Data Warehouse"]
        BQ["BigQuery\n(structured data)"]
        GCS["Cloud Storage\n(raw data lake)"]
    end

    MQTT -->|"day buffer / alert exception"| INGESTAPI
    TICKETGW -->|"day batch / alert exception"| INGESTAPI
    RIDESENSOR -->|"usage, maintenance"| MQTT
    ENCSENSOR -->|"health, feeding, population"| MQTT
    GATESENSOR -->|"footfall"| MQTT
    
    INGESTAPI -->|"validate, stage batch"| BATCH
    BATCH -->|"nightly batch manifest"| PUBSUB
    PUBSUB --> DATAFLOW
    INGESTAPI -->|"critical alerts only"| ALERTPUBSUB
    ALERTPUBSUB -->|"alerting service"| DATAFLOW
    DATAFLOW -->|"transform"| BQ
    DATAFLOW -->|"archive"| GCS
```

---

## 2. Data Sources and Collection Methods

### 2.1 Ticketing System (High Priority, Online)

**Source**: Ticketing Service microservices via API

| Data Entity | Collection Method | Frequency | Volume | Transport |
|-------------|------------------|-----------|--------|-----------|
| **Orders** | Nightly export from transactional DB | Once per night | Daily volume | Cloud Storage batch |
| **Tickets** | Nightly export from transactional DB | Once per night | Daily volume | Cloud Storage batch |
| **Entry Logs** | Gate devices and ticketing DB day buffer | Once per night | Daily volume | Cloud Storage batch |
| **Payments** | Nightly export from payment ledger | Once per night | Daily volume | Cloud Storage batch |
| **Loyalty Events** | Nightly export from loyalty DB | Once per night | Daily volume | Cloud Storage batch |
| **Promotions** | Scheduled extract | Once per night | ~50 records | Cloud Storage batch |

**Schema**: Transactional DB (PostgreSQL) → nightly export → Data Ingestion Service → Pub/Sub batch trigger

**Latency Requirement**: Available in the warehouse by 05:00 next day. Ticketing remains operational in its transactional database during the day.

**Data Freshness**: Daily, after the nightly load

---

### 2.2 Ride Monitoring System (High Priority, Edge-based)

**Source**: Ride sensors via MQTT Broker on estate

| Data Entity | Collection Method | Frequency | Volume | Transport |
|-------------|------------------|-----------|--------|-----------|
| **Usage Logs** | Edge gateway day buffer | Once per night | Daily volume | Cloud Storage batch |
| **Sensor Readings** | Edge gateway day buffer | Once per night | Daily volume | Cloud Storage batch |
| **Maintenance Records** | Nightly export from Operations DB | Once per night | ~50-200/day | Cloud Storage batch |
| **Inspection Records** | Scheduled export | Once per night | ~40/week | Cloud Storage batch |
| **Safety Alerts** | MQTT → ingestion exception path | On threshold breach | Variable | Pub/Sub alert topic |

**Schema**: Sensor data (JSON) → local edge buffer → nightly compressed batch → Data Ingestion Service → Pub/Sub

**Latency Requirement**: Usage and sensor data available by 05:00 next day; critical safety alerts delivered near real time.

**Data Freshness**: Daily for telemetry; near real time for safety alerts

---

### 2.3 Animal Health Monitoring System (High Priority, Edge-based)

**Source**: Enclosure sensors and cameras via MQTT Broker

| Data Entity | Collection Method | Frequency | Volume | Transport |
|-------------|------------------|-----------|--------|-----------|
| **Health Records** | Edge gateway day buffer | Once per night | Daily volume | Cloud Storage batch |
| **Feeding Logs** | Edge gateway day buffer | Once per night | ~300-500/day | Cloud Storage batch |
| **Population Counts** | Edge gateway day buffer | Once per night | Daily volume | Cloud Storage batch |
| **Vet Visits** | Nightly export from Operations DB | Once per night | ~10-50/day | Cloud Storage batch |
| **Health Alerts** | MQTT → ingestion exception path | On threshold breach | Variable | Pub/Sub alert topic |
| **Camera Feeds** | Camera stream (video) | Continuous 24/7 | ~500GB/day | Direct to GCS |

**Schema**: Sensor data (JSON) → local edge buffer → nightly compressed batch → Data Ingestion Service → Pub/Sub. Video remains in Cloud Storage and is processed in scheduled ML jobs.

**Latency Requirement**: Health, feeding, and population data available by 05:00 next day; critical health alerts delivered near real time.

**Data Freshness**: Daily for monitoring data; near real time for critical health alerts

---

### 2.4 Gate and Footfall Sensors (Medium Priority, Edge-based)

**Source**: Smart gate/zone sensors via MQTT

| Data Entity | Collection Method | Frequency | Volume | Transport |
|-------------|------------------|-----------|--------|-----------|
| **Entry Count** | Gate device day buffer | Once per night | Daily aggregate | Cloud Storage batch |
| **Exit Count** | Gate device day buffer | Once per night | Daily aggregate | Cloud Storage batch |
| **Zone Footfall** | Edge gateway day buffer | Once per night | Daily aggregate | Cloud Storage batch |

**Schema**: Sensor counters → local edge buffer → nightly compressed batch → Data Ingestion Service → Pub/Sub

**Latency Requirement**: Available by 05:00 next day

**Data Freshness**: Daily

---

### 2.5 External Data Sources (Low Priority, Pull-based)

| Data Source | Collection Method | Frequency | Volume | Transport |
|-------------|------------------|-----------|--------|-----------|
| **Weather Data** | API: OpenWeatherMap | Once per night | ~100 records/day | REST API batch extract |
| **School Holidays** | API: Government Education Data | Annual | ~250 records/year | REST API |
| **Public Holidays** | API: Local Government | Annual | ~50 records/year | REST API |
| **Promotions Calendar** | Spreadsheet sync | Weekly | ~50 records | Cloud Storage |

**Latency Requirement**: < 1 hour acceptable

**Data Freshness**: Daily/Weekly

---

## 3. Ingestion Pipeline Components

### 3.1 Data Ingestion Service

**Purpose**: Provide a controlled nightly batch-ingress boundary between operational/edge systems and Google Cloud Pub/Sub.

The Data Ingestion Service is not the warehouse and is not the primary long-term message store. It receives nightly files and safety-alert exceptions, applies a common event contract, and publishes a batch manifest to Pub/Sub. Pub/Sub triggers and fans out the batch workflow to Dataflow, alerting, ML feature processing, and other consumers.

**Responsibilities**:
- Accept nightly files from Cloud Storage, database exports, MQTT alert exceptions, and external API extracts
- Authenticate the source and authorize the event type
- Validate the payload against the versioned event schema
- Normalize different source formats into a canonical event envelope
- Add batch metadata such as `batch_id`, `source_system`, `extract_date`, `received_at`, and `schema_version`
- Deduplicate files and records using `batch_id` and source record identifiers
- Publish a valid batch manifest to the Google Cloud Pub/Sub topic
- Send invalid or repeatedly failing events to a Pub/Sub dead-letter topic
- Apply file validation, retry handling, and completeness checks at the ingress boundary

**Implementation**:
- **Technology**: Cloud Run or Cloud Functions + Google Cloud Pub/Sub
- **Language**: Python / Go
- **Schedule**: Nightly load window, for example 02:00-05:00 local time
- **Dead Letter Handling**: Failed records → Cloud Storage for manual review

**Key Logic**:
```
For each nightly source file:
  1. Validate the manifest, checksum, schema, and extract date
  2. Check that the expected source file has arrived
  3. Check for duplicate batch_id or source record identifiers
  4. Enrich with batch metadata (received_at, source_system, schema_version)
  5. Stage the file in Cloud Storage and publish a batch manifest to Pub/Sub
  6. Log the batch outcome to the audit trail
  7. Handle failures → retry queue or dead-letter topic
```

---

### 3.2 Google Cloud Pub/Sub Topics and Subscriptions

**Topic Organization** (by system and entity):

| Topic Name | Retention | Use Case |
|-----------|-----------|----------|
| **estate-ticketing-orders** | 30 days | Order events |
| **estate-ticketing-entries** | 90 days | Gate entry scans |
| **estate-ticketing-payments** | 30 days | Payment events |
| **estate-rides-usage** | 90 days | Ride usage logs |
| **estate-rides-sensors** | 7 days | Sensor readings (high volume) |
| **estate-rides-alerts** | 30 days | Safety alerts |
| **estate-animals-health** | 90 days | Health records |
| **estate-animals-feeding** | 90 days | Feeding logs |
| **estate-animals-population** | 30 days | Population counts |
| **estate-animals-alerts** | 30 days | Health alerts |
| **estate-sensors-footfall** | 30 days | Gate/zone footfall |

Each topic has independent Pub/Sub subscriptions, allowing each consumer to process the same batch manifest independently:
- `nightly-batch-processor-sub` → Dataflow and BigQuery batch writer
- `safety-alerts-sub` → Alerting service for critical exceptions
- `nightly-ml-features-sub` → ML feature processing after the batch load
- `reporting-sub` → Batch reporting and dashboard extracts

Subscriptions use acknowledgment deadlines, retry policies, and dead-letter topics for messages that cannot be processed. The nightly batch manifest is small and cheap to replay; Cloud Storage holds the batch files, while BigQuery remains the system of record for historical analytics.

---

### 3.3 Batch Processing (GCP Dataflow)

**Purpose**: Transform, aggregate, and enrich the completed nightly batch. Dataflow is started by the Pub/Sub batch manifest and stops when the batch succeeds or fails.

**Processing Pipelines**:

**Pipeline 1: Entry Log Processing**
```
Input: nightly batch manifest from `estate-ticketing-entries` via `nightly-batch-processor-sub`
  ↓
Window: batch extract for the previous operating day
  ↓
Transform:
  - Parse entry timestamp and gate location
  - Map ticket_id to zone_id (lookup from zone table)
  - Derive hour_of_day, day_of_week, is_peak_hour
  ↓
Aggregate:
  - Count entries per gate per minute
  - Count entries per zone per minute
  ↓
Enrich:
  - Join with promotion data
  - Join with weather data (lookup)
  ↓
Output:
  - Write to BigQuery: entry_logs (raw)
  - Write to BigQuery: footfall_aggregates (1-min window)
  - Write daily aggregates to BigQuery; no real-time dashboard cache is required
```

**Pipeline 2: Ride Usage Processing**
```
Input: nightly batch manifest from `estate-rides-usage` and `estate-rides-sensors`
  ↓
Window: batch extract for the previous operating day
  ↓
Transform:
  - Parse sensor readings (usage count, cycle time, rider count)
  - Calculate utilization = rider_count / capacity
  - Detect anomalies (unusual patterns)
  ↓
Aggregate:
  - Sum usage per ride per interval
  - Calculate avg rider count per cycle
  - Track operational hours
  ↓
Output:
  - Write to BigQuery: ride_usage_logs (raw)
  - Write to BigQuery: ride_summary_5min (aggregated)
  - Publish alerts on threshold breach
```

**Pipeline 3: Animal Health Processing**
```
Input: nightly batch manifest from `estate-animals-health` and `estate-animals-feeding`
  ↓
Window: batch extract for the previous operating day
  ↓
Transform:
  - Parse health metrics (weight, temp, vital signs)
  - Parse feeding logs (qty dispensed, qty consumed %)
  - Compare against normal ranges per species
  - Detect anomalies (e.g., weight drop > 5%)
  ↓
Aggregate:
  - Average health metrics per enclosure per interval
  - Feeding efficiency per enclosure per day
  ↓
Output:
  - Write to BigQuery: health_records (raw)
  - Write to BigQuery: health_summary_5min (aggregated)
  - Publish health alerts to Operations API
```

---

### 3.4 Storage Layer

#### BigQuery (Structured Data Warehouse)

**Raw Data Tables** (immutable, append-only):
- `raw.entry_logs` - individual gate scans
- `raw.ride_usage_logs` - per-cycle usage
- `raw.ride_sensor_readings` - sensor telemetry
- `raw.health_records` - animal health snapshots
- `raw.feeding_logs` - feeding events
- `raw.population_counts` - population observations
- `raw.maintenance_records` - maintenance events
- `raw.orders` - ticketing orders

**Processed/Aggregated Tables** (optimized for analytics):
- `processed.entry_logs_hourly` - hourly entry aggregates
- `processed.footfall_by_zone` - hourly footfall per zone
- `processed.ride_summary_daily` - daily ride usage summary
- `processed.ride_summary_hourly` - hourly ride summary
- `processed.health_summary_daily` - daily health metrics per enclosure
- `processed.feeding_efficiency_daily` - daily feeding rates
- `processed.population_by_enclosure` - latest population per enclosure
- `processed.visitor_journey` - session-level visitor paths

**Dimensional Tables** (reference data):
- `dim.zones` - zone/location reference
- `dim.rides` - ride reference with capacity, status
- `dim.enclosures` - enclosure reference
- `dim.species` - animal species reference
- `dim.ticket_types` - ticket catalog
- `dim.employees` - keeper/staff reference
- `dim.calendar` - calendar dimensions (date, day of week, holiday flags, school term)
- `dim.weather` - historical weather data

#### Cloud Storage (Raw Data Lake)

**Buckets**:
- `gs://estate-data-raw/` - raw sensor data (daily partitions)
- `gs://estate-data-raw/camera-feeds/` - video files (encrypted, access-controlled)
- `gs://estate-data-backups/` - BigQuery table backups (weekly)
- `gs://estate-data-exports/` - data exports for external systems

**Data Format**: Parquet (columnar, compressed) or JSON Lines (batch friendly)

**Retention**: 
- Raw sensor data: 2 years
- Video feeds: 30 days (with option to archive high-value clips)
- Backups: 1 year

---

## 4. Data Quality and Validation

### 4.1 Schema Registry

**Purpose**: Central schema management for all data topics

**Implementation**: Versioned JSON/Avro/Protobuf contracts managed with source control and enforced by the Data Ingestion Service. Pub/Sub topic and subscription configuration is managed as infrastructure-as-code.

**Schemas Managed**:
```json
Topic: ticketing.entries
{
  "type": "record",
  "name": "GateEntry",
  "fields": [
    {"name": "entry_id", "type": "string"},
    {"name": "ticket_id", "type": "string"},
    {"name": "gate_id", "type": "string"},
    {"name": "zone_id", "type": "string"},
    {"name": "scan_time", "type": "string", "logicalType": "timestamp-millis"},
    {"name": "entry_valid", "type": "boolean"},
    {"name": "source_system", "type": "string"}
  ]
}
```

### 4.2 Data Quality Checks

**Ingestion-time Checks**:
- Schema validation (Avro/Protobuf)
- Required fields presence
- Data type validation
- Range validation (e.g., rider_count <= ride_capacity)
- Timestamp reasonableness (within ±5 min of current time)

**Processing-time Checks** (GCP Dataflow):
- Duplicate detection (within 5-min window)
- Outlier detection (statistical)
- Missing value imputation for key metrics
- Referential integrity (e.g., zone_id exists in dim.zones)
- Completeness: all expected records arrived within SLA

**Data Quality Metrics** (tracked in BigQuery):
- `data_quality.ingestion_latency` - time to reach warehouse
- `data_quality.missing_values` - % of null values per column
- `data_quality.duplicate_records` - count of duplicates detected
- `data_quality.anomalies` - count of outliers detected

---

## 5. Handling Edge Connectivity

### 5.1 Intermittent Connectivity Pattern

**Challenge**: WiFi coverage on estate is patchy

**Solution**: Local buffering and store-and-forward

**Architecture**:
```
Edge Device (Ride/Animal Sensor)
  ↓ (collects data throughout the day)
Local SQLite / Parquet buffer
  ↓ (nightly upload when WiFi is available)
Cloud Storage landing bucket
  ↓ (manifest published once per source batch)
Data Ingestion Service (validation, checksum, retry)
  ↓
Google Cloud Pub/Sub
```

**MQTT Configuration**:
- QoS Level: 1 (at-least-once delivery)
- Session Persistence: Enabled
- Message Batching: one compressed file per source per operating day
- Upload Window: 02:00-05:00 local time
- Reconnection: exponential backoff during the nightly upload window

**Data Recovery**:
- If disconnected during the upload window: retain the batch locally and retry at the next window
- If local storage reaches 80%: alert operations and prioritize the oldest batch
- Periodic reconciliation: compare source manifests with warehouse load manifests

---

## 6. Ingestion SLAs and Guarantees

| Data Source | Target Latency | Data Freshness | Availability |
|-----------|----------------|----------------|--------------|
| **Ticketing (Entry Logs)** | By 05:00 next day | Daily | 99.5% |
| **Ticketing (Orders)** | By 05:00 next day | Daily | 99.5% |
| **Ride Usage and Sensors** | By 05:00 next day | Daily | 99% |
| **Animal Health and Feeding** | By 05:00 next day | Daily | 99% |
| **Gate Footfall** | By 05:00 next day | Daily | 99% |
| **Critical Safety/Health Alerts** | < 1 minute | Exception path | 99.9% |
| **External Data** | By 05:00 next day | Daily/weekly | 99% |

**Recovery Procedure**:
1. If SLA breach detected: Alert data engineering team
2. Trigger manual reprocessing of missed data
3. Reconcile with source systems (APIs, local backups)
4. Update dimension tables and rerun affected ML jobs

---

## 7. Ingestion Configuration and Monitoring

### 7.1 Configuration Management

**Environment Variables**:
- `BQ_DATASET_ID` - BigQuery dataset
- `GCS_BUCKET` - Cloud Storage bucket for raw data
- `PUBSUB_PROJECT_ID` - Google Cloud project containing Pub/Sub topics
- `PUBSUB_TOPIC_PREFIX` - Prefix for estate topics
- `PUBSUB_DEAD_LETTER_TOPIC` - Dead-letter topic for failed events
- `BATCH_MAX_FILE_SIZE_MB` - Maximum compressed source file size
- `NIGHTLY_LOAD_START` - Start of the batch upload window
- `NIGHTLY_LOAD_END` - End of the batch upload window

### 7.2 Monitoring and Alerting

**Key Metrics** (via Cloud Monitoring / Stackdriver):
- `ingestion_lag` - seconds behind source
- `batch_file_rate` - files received per source and load window
- `processing_latency` - end-to-end latency
- `dead_letter_queue_size` - failed records
- `pubsub_oldest_unacked_message_age` - age of the oldest unacknowledged message
- `pubsub_dead_letter_count` - messages routed to dead-letter topics

**Alerts**:
- Nightly batch not received by 03:00
- Error rate > 1%
- Dead-letter topic contains any batch manifest
- Batch completeness check fails

---

## 8. Example: Complete Entry Log Flow

```
1. Visitor scans QR code at gate
  ↓
2. Gate Scanner Device validates the ticket locally and appends the entry to its day buffer
  ↓
3. At the nightly close, the ticketing system exports the day's entry logs
  ↓
4. Batch file is compressed, checksummed, and uploaded to Cloud Storage
  ↓
5. Data Ingestion Service validates the manifest and deduplicates the batch
  ↓
6. Publish the batch manifest to `estate-ticketing-entries` Pub/Sub topic
  ↓
7. Nightly Dataflow batch job processes the previous day's data
  - Enrich: zone_id lookup
  - Aggregate: footfall by zone and hour
  - Detect: data-quality and volume anomalies
  ↓
8. Write outputs:
  a) BigQuery raw.entry_logs (raw batch records)
  b) BigQuery processed.entry_logs_hourly (daily backfill)
  c) BigQuery processed.visitor_daily_summary
  d) Publish a safety alert only if a configured exception is detected
  ↓
9. Dashboards and ML feature tables refresh after the nightly load
```

---

This batch-first ingestion pipeline reduces platform and network costs by avoiding continuous streaming. It still provides reliable historical data for reporting and ML models, while safety-critical ride and animal alerts use a separate near-real-time Pub/Sub exception path.
