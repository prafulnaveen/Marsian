# High Level Context

> This page describes the high level context of the problem. It lists down what all different services will be required to make the solution work. It breaks down the services into categories to be focused into.


The solution covers follow categories:

**People Layer** : They are vistiors, ground staff(queue mgmt team,animal keeper, ride operator etc), Back office staff(Data team, operation team etc), Management.

**Edge Layer** : These are sensors on gates, enclosures etc

**Cloud Layer** : This is the platform where all data ingestion, processing, notification, Dashboarding, website etc will run.

---



### High level intraction between these layers and the service running in these layers are depicted below.

---



```mermaid

flowchart TB

    %% ===== USERS / ACTORS =====
    subgraph USERS["People"]
        VISITOR["Visitor"]
        KEEPER["Animal Keeper"]
        RIDEOPS["Ride Operations Staff"]
        MGMT["Estate Management"]
    end

    %% ===== ON-ESTATE EDGE LAYER =====
    subgraph EDGE["On-Estate Edge Layer (patchy WiFi)"]
        RIDESENSOR["Ride Usage Sensors\n(per-ride counters, wear/safety sensors)"]
        ENCSENSOR["Enclosure Sensors\n(health, feeder, population/vision cameras)"]
        GATESENSOR["Gate / Zone Footfall Sensors"]
        MQTT["MQTT Broker / Edge Gateway\n(local buffering, store-and-forward)"]

        RIDESENSOR --> MQTT
        ENCSENSOR --> MQTT
        GATESENSOR --> MQTT
    end

    %% ===== CLOUD PLATFORM =====
    subgraph CLOUD["Cloud Platform"]
        INGEST["Data Ingestion Service\n(handles intermittent sync)"]
        DATAPLATFORM["Central Data Platform\n(storage + processing)"]

        TICKETING["Ticketing Service\n(standard & family passes)"]
        PAYMENT["Payment Gateway\n(external, PCI-compliant)"]
        LOYALTY["Loyalty / Returning-Visitor Service"]

        ANALYTICS["Visitor Analytics Service\n(footfall & popularity by zone/ride)"]
        ANIMALMON["Animal Monitoring Service\n(health, feeding, population)"]
        RIDEMON["Ride Monitoring Service\n(usage & safety trend tracking)"]

        ALERTING["Alerting & Notification Service"]
        DASHBOARD["Reporting & Dashboard Service"]

        INGEST --> DATAPLATFORM
        DATAPLATFORM --> ANALYTICS
        DATAPLATFORM --> ANIMALMON
        DATAPLATFORM --> RIDEMON

        TICKETING --> PAYMENT
        TICKETING --> LOYALTY
        TICKETING --> DATAPLATFORM

        ANIMALMON --> ALERTING
        RIDEMON --> ALERTING

        ANALYTICS --> DASHBOARD
        ANIMALMON --> DASHBOARD
        RIDEMON --> DASHBOARD
        LOYALTY --> DASHBOARD
    end

    %% ===== CONNECTIONS: EDGE -> CLOUD =====
    MQTT -->|"telemetry (synced when connectivity available)"| INGEST

    %% ===== CONNECTIONS: USERS -> SERVICES =====
    VISITOR -->|"buys tickets / family pass"| TICKETING
    VISITOR -->|"scans in at gate"| GATESENSOR

    KEEPER -->|"views alerts & health data"| DASHBOARD
    ALERTING -->|"health / feeding / population alerts"| KEEPER

    RIDEOPS -->|"views ride usage & safety data"| DASHBOARD
    ALERTING -->|"ride wear / safety alerts"| RIDEOPS

    MGMT -->|"views footfall, revenue & retention insights"| DASHBOARD

```
