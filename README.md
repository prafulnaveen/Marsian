# Project: Digital Transformation for the Grand Estate

# Overview

This project is a digital transformation initiative designed to revitalize a large, historic estate and turn it into a profitable, modern visitor attraction. The primary goal is to create a sustainable business model by monetizing the estate's unique assets, including a historic amusement park and an exotic animal collection. The project aims to significantly increase visitor engagement and daily attendance by leveraging data-driven insights to enhance the visitor experience and optimize operations.

# Key Features

The proposed digital solution will include the following key features:

<table><tbody><tr><td>Feature</td><td>Description</td></tr><tr><td>Online Ticketing System</td><td>A secure and user-friendly platform for customers to purchase individual or family tickets online, helping to streamline visitor entry.</td></tr><tr><td>Ride Monitoring System</td><td>An IoT-based system to monitor and record &nbsp;the ride usage, maintance requirement.</td></tr><tr><td>Animal Health Monitoring System</td><td>An IoT-based system to track the health and well-being of the exotic animal collection, with alerts for staff to ensure animal welfare.</td></tr><tr><td><p>AI based Usecase&nbsp;</p><ul><li>Demand Forcaseting</li><li>Dynamic Pricing&nbsp;</li><li>Lead Generation</li><li>Vision Based Population Count</li><li>Behaviour Stress Recognization</li><li>Ride Maintainence Prediction</li><li>Ride Level Deamand Forcasting</li><li>Marketing content Generation</li><li>Staffing Optimization</li></ul></td><td><p>ML and AI based usecases for&nbsp;</p><ul><li>frocasting the sale of tickets for different section of estate.</li><li>Providing the pricing strategy for profitability of the estate</li><li>Genrating lead to increase the ticket sale and footfall.</li><li>Counting the population of aquatic species.</li><li>Health monitoing of the animal based on behaviour/stress.</li><li>Ride Maintenence Prediction</li><li>Ride Level Demand&nbsp;</li><li>Marketing content generation for better reach</li><li>Better Staffing strategy</li></ul></td></tr><tr><td>Dare Warehousing</td><td><p>A strategy to ingest, process and consume the dara with focus on&nbsp;</p><ul><li>Single Source of Truth</li><li>ML Requirements</li><li>Analytical and Reporting Requirements</li><li>Data Quality</li><li>Product Strategy</li></ul></td></tr><tr><td>Reporting and Analytics Dashbaord</td><td>A centralized dashboard to visualize visitor data and animal health metrics, empowering staff to make informed, data-driven decisions on investments and improvements.</td></tr></tbody></table>

# Technical Considerations

The system is designed to be robust and scalable, with a focus on reliability in a challenging operational environment.

**Scalability:** The architecture will be built to handle a threefold increase in daily visitor numbers over the next three years.

**Offline-First Capability:** Given the estate's patchy Wi-Fi, the system will utilize IoT devices that can store data locally and sync with the cloud once a connection is available, ensuring no data is lost.

**Phased Implementation:** The project will be rolled out in phases, starting with the most critical features to manage costs and reduce risk. It will leverage pay-as-you-go cloud services for a cost-effective and scalable solution.

# Repository Structure

```
.
├── problem_statement.md
├── README.md
├── adrs/
│   ├── ADR-000-Template.md
│   ├── ADR-001-Cloud Provider.md
│   ├── ADR-002-Choice of MQTT devices.md
│   ├── ADR-003-GKE_Usages.md
│   ├── ADR-004-Transcational_Database.md
│   ├── ADR-005-Session_Cart_Service.md
│   ├── ADR-006-Sensor_Data_Storage.md
│   ├── ADR-007-Payment_Gateway_service.md
│   ├── ADR-008-Notification_Service.md
│   ├── ADR-009-Dashboarding_And_Visualization.md
│   ├── ADR-010-Logging_And_Monitoring.md
│   ├── ADR-011-Identity_And_Authentication.md
│   ├── ADR-012-Inter_Service_Communication.md
│   ├── ADR-013-Stream_Processing.md
│   ├── ADR-014-CICD_Tools.md
│   ├── ADR-015-Content_Delivery.md
│   ├── ADR-016-MachineLearning_Jobs.md
│   └── readme.md
├── hld/
│   ├── 0_context_layer_diagram.md
│   ├── readme.md
│   ├── 1_ticketing_system/
│   │   ├── infra.md
│   │   ├── schema.md
│   │   └── services.md
│   ├── 2_ride_monitoring_system/
│   │   ├── infra.md
│   │   ├── schema.md
│   │   └── services.md
│   ├── 3_animal_health_monitoring_system/
│   │   ├── infra.md
│   │   ├── schema.md
│   │   └── services.md
│   ├── 4_ML_implementation/
│   │   ├── 4.1_Demand_Forecasting.md
│   │   ├── 4.2_Dynamic_Pricing.md
│   │   ├── 4.3_Lead_Generation.md
│   │   ├── 4.4_Visison_Based_Population_Counting.md
│   │   ├── 4.5_Behaviour_Stress_recognizition.md
│   │   ├── 4.6_Ride_Maintenence_Prediction.md
│   │   ├── 4.7_Ride_Level_Demand_Prediction.md
│   │   ├── 4.8_Marketing_Content_Generation.md
│   │   └── 4.9_Staffing_Optimization.md
│   └── 5_DataWarehouse/
│       ├── consumption.md
│       ├── data_products.md
│       └── ingestion.md
├── requirements/
│   ├── 1_Business_Goal_and_Drivers.md
│   ├── 2_Business_Challenges.md
│   ├── 3_FR.md
│   ├── 4_NFR.md
│   └── 5_Risk_and_Mitigations.md
└── video/
   └── readme.md
```