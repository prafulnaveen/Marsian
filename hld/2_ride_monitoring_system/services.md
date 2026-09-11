# Services

> This file contains various microservices needed to operate the ride monitoring system.

## Service Inventory

| # | Area | Service Name | Usages |
|---|------|--------------|--------|
| 1 | Sensors | Usage Counters | Track riders per cycle and total cycle counts |
| 2 | Sensors | Wear / Vibration Sensors | Monitor structural and mechanical stress indicators |
| 3 | Sensors | Safety Interlock Sensors | Monitor restraint status and e-stop triggers |
| 4 | Core | Telemetry Ingestion Service | Normalize and aggregate sensor data streams |
| 5 | Core | Pub/Sub | Stream telemetry events to downstream processors |
| 6 | Core | Dataflow | Process streams, write to BigQuery, check thresholds, emit alerts |
| 7 | Core | BigQuery | Persist historical count, vibration, and temperature data |
| 8 | Core | Looker Studio | Create trend dashboards for monitoring |



**Below diagram shows the services and connections between them.**

---

```mermaid
flowchart TB

    subgraph SENSORS["Ride Sensors (per ride)"]
        USAGESENSOR["Usage Counters\n(riders per cycle, cycle count)"]
        WEARSENSOR["Wear / Vibration Sensors\n(structural & mechanical stress)"]
        SAFETYSENSOR["Safety Interlock Sensors\n(restraints, e-stops)"]
    end

    subgraph CORE["Ride Monitoring"]
        INGEST["Telemetry Ingestion Service\n(normalizes sensor)"]
        PUBSUB["Pub/Sub\n(telemetry ingestion)"]
        DATAFLOW["Dataflow\n(stream processing:\nBigQuery writes + threshold/anomaly checks + alert emit)"]
        BQ["BigQuery\n(count, vibration, temperature)"]
        LOOKER["Looker Studio\n(trend dashboards)"]

        OPSAPI["Operations API + DB\n( Usages records,\ncurrent status, alert notifications)"]
    end

    subgraph CLIENTS["Client Channels"]
        RIDEOPSUI["Ride Ops Mobile/Web App"]
        MAINTUI["Maintenance Technician UI"]
        ADMINUI["Estate Admin Console"]
    end

    USAGESENSOR --> INGEST
    WEARSENSOR --> INGEST
    SAFETYSENSOR --> INGEST
    INGEST --> PUBSUB --> DATAFLOW
    DATAFLOW --> BQ --> LOOKER
    DATAFLOW -->|"alert on threshold breach"| OPSAPI
    
    OPSAPI -->|"push notification"| ADMINUI
    ADMINUI -->|"view records, log visits"| OPSAPI

    OPSAPI -->|"push notification"| RIDEOPSUI
    RIDEOPSUI -->|"view records, log usages"| OPSAPI

    OPSAPI -->|"push notification"| MAINTUI
    MAINTUI -->|"view records, log  maintence visits"| OPSAPI
```