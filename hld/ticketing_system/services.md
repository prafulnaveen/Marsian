# Service

> This file list microservices running to facilitate the ticketing system.

**Different micoservices and their purpose is listed in table below**

| # | Area | Service Name | Service Description |
| --- | --- | --- | --- |
| 1 | Clients | Web/Mobile Booking App | Bookign tickets via web or mobile application. |
| 2 |   | Admin UI | Admin UI for admin funcationlaity - updating promotions, overriding normal processes etc |
| 3 |   | GateScan | Validating tikcets at the entry gates (main entry gates, attraction/ride/animal farm entry gates) |
| 4 | Api Gateway | Api Gateway service | Integrating all microservices, routing, rate limiting etc |
| 5 | Core components | Auth & Identity Service | Provies authentication and Validates the identity |
| 6 |   | Booking Service | Facilitates the booking of the tikcet for the estate. Provides various option of tikcet - individual, family, retunrn vistor etc. Manages carts and checkouts.. |
| 7 |   | Catalogue Service | Provide the catalogue of diffrenet tickets avaiable to purchase along with their price, ongoing promotions |
| 8 |   | Loyality Service | Provide the benefits and points related to loyality and repeat visit tracking |
| 9 |   | Payment Service | Interfaces with the external PCI complaint Payment Providers |
| 10 |   | Ticket Issuance service | Generates Qr/barcode, e-ticket |
| 11 |   | Notification Service | Notifies the client via mail/sms/whatapp |
| 12 |   | Entry Vallidation Service | Checks the validity of the tikcet presented at the gate |
| 13 |   | Reporting Service | Reporting on the daily, weekly Sales |
| 14 | Data Stores | Transactional DB | Database for storing transactional data |
| 15 |   | Cart and Session Cache | Fast db for quick lookup |

---

#### **Below diagram denotes the interaction between these services**

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
        CATALOG["Catalog\n(ticket types, family pass rules, promotions)"]
        BOOKING["Booking Service\n(cart, checkout orchestration)"]
        PAYMENT["Payment Service\n(wraps external payment gateway)"]
        ISSUANCE["Ticket Issuance Service\n(generates QR/barcode, e-ticket)"]
        VALIDATION["Entry Validation Service\n(scan verification, anti-fraud/reuse check)"]
        LOYALTY["Loyalty Service\n(points, repeat-visit tracking)"]
        NOTIFY["Notification Service\n(email/SMS confirmations, reminders)"]
        REPORTING["Sales Reporting"]
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
````