# Non-Functional Requirements

This document contains the non-functional requirement for the to be solution


| NFR# | Area                                 | Non Functional Requiremets                                                                                                                                                                                                          | Comments |
| ---- | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| 1    | Cost-efficiency                      | Given the whole initiative is about profitability, the solution itself shouldn't be so expensive it undermines the profit goal.                                                                                                     |          |
| 2    | Scalability                          | The system must be able to handle a threefold increase in visitor numbers (from 5,000 to 15,000 daily visitors) within three years without performance degradation.                                                                 |          |
| 3    | Offline Capability                   | Given the patchy Wi-Fi, the system must be able to function reliably in areas with poor or no connectivity. Data collected by on-site devices should be stored locally and synced to the cloud when a connection becomes available. |          |
| 4    | Usability                            | The system should be easy for park staff to use for monitoring and for visitors to use for purchasing tickets.                                                                                                                      |          |
| 5    | Reliability                          | The animal health monitoring components must be highly reliable, as system failure could directly impact animal welfare.                                                                                                            |          |
| 6    | Low Power and MQTT Device Protection | MQTT sensors spread across a large estate are likely hard to service frequently, so battery/power efficiency matters                                                                                                                |          |
| 7    | Regulatory Complaince                | Animal welfare regulations and ride safety standards likely impose reporting/audit-trail requirements                                                                                                                               |          |
| 8    | Security and PCI Compliance          | The ticketing and payment system must be secure to protect customer data and financial transactions.                                                                                                                                |          |
