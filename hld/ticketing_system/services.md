# Service

> This file list microservices running to facilitate the ticketing system.

Different micoservices and their purpose is listed in table below



| # | Service Name                    | Service Description                            |
| - | ------------------------------- | ---------------------------------------------- |
| 1 | Auth and Visitor Identity Check | For Authenticating the logged in user or Guest |
|   |                                 |                                                |



---



#### Below diagram denotes the interaction between these services

---



```mermaid

flowchart TB

    subgraph CLIENTS["Client Channels"]
        WEBAPP["Web / Mobile Booking App"]
        GATESCAN["Gate Scanner Device\n(QR / barcode reader)"]
        ADMINUI["Staff Admin Console"]
    end

    subgraph EDGE_ENTRY["Entry Point"]
        APIGW["API Gateway\n(auth, rate limiting, routing)"]
    end

    subgraph CORE["Ticketing Service - Core Components"]
        AUTH["Auth & Visitor Identity Service\n(login, guest checkout)"]
        CATALOG["Catalog & Pricing Service\n(ticket types, family pass rules, promotions)"]
        BOOKING["Order / Booking Service\n(cart, checkout orchestration)"]
        PAYMENT["Payment Service\n(wraps external payment gateway)"]
        ISSUANCE["Ticket Issuance Service\n(generates QR/barcode, e-ticket)"]
        VALIDATION["Entry Validation Service\n(scan verification, anti-fraud/reuse check)"]
        LOYALTY["Loyalty & Returning-Visitor Service\n(points, repeat-visit tracking)"]
        NOTIFY["Notification Service\n(email/SMS confirmations, reminders)"]
        REPORTING["Sales & Attendance Reporting"]
    end

    subgraph EXTERNAL["External / Shared Platform"]
        PAYGW["External Payment Gateway\n(Stripe/Adyen - PCI scope)"]
        DATAPLATFORM["Central Data Platform\n(feeds visitor analytics)"]
    end

    subgraph STORE["Data Stores"]
        TXNDB["Transactional DB\n(orders, tickets, payments)"]
        CACHE["Cart / Session Cache"]
    end

    WEBAPP --> APIGW
    ADMINUI --> APIGW
    GATESCAN --> APIGW

    APIGW --> AUTH
    APIGW --> CATALOG
    APIGW --> BOOKING
    APIGW --> VALIDATION
    APIGW --> REPORTING

    AUTH --> LOYALTY
    BOOKING --> CATALOG
    BOOKING --> CACHE
    BOOKING --> PAYMENT
    PAYMENT --> PAYGW
    PAYMENT --> BOOKING
    BOOKING --> ISSUANCE
    ISSUANCE --> TXNDB
    ISSUANCE --> NOTIFY

    VALIDATION --> TXNDB
    VALIDATION --> LOYALTY

    BOOKING --> TXNDB
    LOYALTY --> TXNDB

    TXNDB --> REPORTING
    REPORTING --> DATAPLATFORM
    VALIDATION -->|"entry scan events"| DATAPLATFORM



```
