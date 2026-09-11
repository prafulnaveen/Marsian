# ADR-002 — Choice of MQTT Devices for IoT Data Collection

## Context

The estates are large, sprawling properties with patchy WiFi coverage. The system needs to:
- Monitor 40 amusement park rides for operational status and visitor popularity
- Track health and welfare of 200+ animals across 55 displays/enclosures
- Collect data from distributed sensors across the estate
- Send collected data reliably to cloud services for analytics and monitoring
- Support growth from 5,000 to 15,000+ daily visitors

Key constraints:
- **Unreliable network connectivity**: WiFi coverage is patchy throughout the estate
- **Budget availability**: Funds are allocated for MQTT-capable hardware devices
- **Geographic distribution**: Sensors need to be deployed across multiple locations
- **Real-time requirements**: Animal health monitoring and ride safety require timely data transmission
- **Limited local infrastructure**: No guaranteed on-premises server infrastructure in all areas

## Decision

**We will use MQTT (Message Queuing Telemetry Transport) devices as the primary protocol for collecting and transmitting sensor data from the estates to cloud services.**

MQTT is a lightweight, publish-subscribe messaging protocol specifically designed for IoT and machine-to-machine communication. It is ideal for this use case due to its ability to operate efficiently over unreliable networks with limited bandwidth.

## Key Differentiators

- **Lightweight Protocol**  
  MQTT has minimal overhead (2-byte fixed header) compared to HTTP/REST, making it ideal for bandwidth-constrained environments. This is critical given the patchy WiFi coverage on the estates.

- **Publish-Subscribe Architecture**  
  Sensors publish data to topics without needing to know about subscribers. This enables scalable, decoupled communication—ride sensors, animal monitoring systems, and ticketing systems can all publish independently without tight coupling.

- **Quality of Service (QoS) Levels**  
  MQTT offers three QoS levels (0, 1, 2) that guarantee message delivery even over unreliable connections. QoS 1 ensures at-least-once delivery, which is essential for critical animal health data and ride safety information.

- **Persistent Connections**  
  MQTT maintains persistent connections using keep-alive mechanisms, reducing reconnection overhead compared to REST APIs that require new HTTP connections for each request.

- **Offline Message Buffering**  
  MQTT brokers can queue messages when subscribers are temporarily offline, ensuring no data loss during WiFi disruptions—a frequent occurrence on the sprawling estate.

- **Low Power Consumption**  
  MQTT's efficient protocol design is suitable for battery-powered IoT devices, extending device lifecycle and reducing maintenance burden for remote sensors.

- **Mature Ecosystem**  
  Extensive tool support, libraries, and cloud platform integration (AWS IoT Core, Azure IoT Hub, Google Cloud IoT, etc.) are available for MQTT implementations.

## Alternatives Considered

- **HTTP/REST APIs**  
  Traditional RESTful services over HTTP. While familiar to developers, they require higher bandwidth, more overhead per request, and are inefficient for frequent sensor updates. Unreliable networks necessitate complex retry logic and connection management. Not optimized for IoT scenarios.

- **CoAP (Constrained Application Protocol)**  
  A lightweight alternative designed for constrained IoT devices. While CoAP is efficient, it lacks the mature ecosystem and cloud platform integration that MQTT provides. Fewer off-the-shelf hardware solutions are available for CoAP compared to MQTT.

- **WebSockets**  
  Full-duplex communication over HTTP. While reliable, WebSockets require more bandwidth than MQTT and don't provide the same message queuing and QoS guarantees. Better suited for real-time web applications than IoT deployments.

- **Zigbee**  
  A mesh networking protocol for low-power devices. While excellent for local wireless communication, Zigbee requires specialized hardware and mesh infrastructure. It's not designed for cloud integration and would add complexity to bridge data to cloud services.

- **LoRaWAN**  
  A long-range, low-power wide-area network protocol. LoRaWAN is ideal for sparse, wide-area deployments but requires dedicated gateway infrastructure and has limited bandwidth—inappropriate for the data volume needed for animal health monitoring and ride analytics.

- **Custom Binary Protocol over TCP/UDP**  
  Building a proprietary protocol would avoid dependency on standards but introduces significant development overhead, security risks, and maintenance burden. Eliminates access to mature tooling and community support.

## Why MQTT is Better Than Alternatives

| Criterion | MQTT | HTTP/REST | CoAP | Zigbee | LoRaWAN |
|-----------|------|----------|------|--------|---------|
| **Bandwidth Efficiency** | Excellent | Poor | Good | Excellent | Good |
| **Unreliable Network Tolerance** | Excellent (QoS + buffering) | Fair | Good | Excellent | Excellent |
| **Cloud Integration** | Excellent | Excellent | Limited | Limited | Limited |
| **Ecosystem Maturity** | Excellent | Excellent | Good | Good | Good |
| **Hardware Availability** | Excellent (widespread) | Excellent | Fair | Good | Limited |
| **Learning Curve** | Low-Medium | Low | Medium | Medium-High | High |
| **Cost** | Low-Medium | Low | Low | Medium | High |
| **QoS Guarantees** | Yes (3 levels) | No | Basic | Limited | No |
| **Message Persistence** | Yes (broker) | No | No | No | Limited |
| **Scalability** | Excellent | Good | Good | Fair | Fair |

For this use case:
- **Network Reliability**: MQTT's QoS and message persistence handle patchy WiFi far better than HTTP/REST
- **Budget Alignment**: MQTT devices are widely available and affordable, matching the budget allocation
- **Data Volume**: Animal monitoring and ride popularity tracking generate frequent sensor updates; MQTT's efficiency matters
- **Cloud Integration**: All major cloud providers support MQTT natively
- **Operational Simplicity**: MQTT brokers handle delivery guarantees; no complex retry logic needed in sensor code

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Broker Dependency** | All data flows through a central MQTT broker; broker failure could disrupt data collection | Deploy broker with high availability (clustering/failover), implement health monitoring, have manual fallback procedures |
| **Network Bandwidth** | While MQTT is efficient, the sheer volume of sensor data (40 rides + 200+ animals) could saturate connectivity | Implement message filtering, aggregate data locally on edge devices before publishing, use higher QoS only for critical data |
| **Security Complexity** | MQTT supports TLS and authentication, but misconfigurations could expose sensor networks | Enforce TLS encryption, implement certificate management, use MQTT access control lists (ACLs), regular security audits |
| **Vendor Lock-in** | Relying on cloud provider's MQTT service could lock implementation to one platform | Use open MQTT broker (Mosquitto) for local infrastructure where possible, design abstraction layers for cloud integration |
| **Debugging Difficulty** | Asynchronous publish-subscribe model can make distributed debugging harder than request-response | Implement comprehensive logging and monitoring, use MQTT tools for message inspection, establish debugging protocols |
| **Latency Variance** | QoS 2 (exactly-once) incurs higher latency than QoS 0; trade-off between reliability and speed | Use QoS 1 for most data, QoS 2 only for critical events, implement latency monitoring |
| **Device Management at Scale** | Managing software updates, credentials, and configuration across many devices is complex | Implement device management platform (AWS IoT Jobs, Azure Device Management), standardize device images, automate provisioning |

## Conclusion

MQTT is the optimal choice for this distributed IoT system because it:

1. **Handles Poor Connectivity**: The patchy WiFi on the estates is a primary concern. MQTT's QoS levels and message queuing ensure data isn't lost during connectivity disruptions—HTTP/REST cannot guarantee this without significant additional complexity.

2. **Aligns with Budget**: MQTT-capable devices are widely available and affordable, directly matching the stated hardware budget allocation.

3. **Scales Efficiently**: With 40 rides, 200+ animals, and potential growth to 15,000 daily visitors, the system will generate high sensor data volume. MQTT's low-overhead, publish-subscribe model handles this better than request-response paradigms.

4. **Integrates with Cloud**: All major cloud providers (AWS IoT Core, Azure IoT Hub, Google Cloud IoT) have native MQTT support, enabling seamless data transit from edge devices to analytics platforms.

5. **Reduces Operational Burden**: MQTT brokers handle message delivery guarantees and persistence, eliminating the need for custom retry logic, circuit breakers, and complex error handling in sensor code.

6. **Future-Proof**: MQTT is an industry standard for IoT with a mature ecosystem. As the system grows or evolves, the technology investment remains valuable.

**Recommendation**: Proceed with MQTT device deployment. Prioritize selecting MQTT brokers with clustering/HA capabilities and implement comprehensive security controls from the outset.
