# ADR-012 — Choice of gRPC and REST APIs for Inter-Service Communication

## Context

The estate system comprises 30+ microservices that communicate with each other:

**Service Interaction Patterns:**
- **Booking Service** → Payment Service: Authorize payment
- **Booking Service** → Ticket Issuance Service: Generate tickets
- **Ticket Issuance Service** → Notification Service: Send confirmations
- **Entry Validation Service** → Cloud SQL: Verify ticket
- **Loyalty Service** → Cloud SQL: Update points
- **Reporting Service** → BigQuery: Query sales data
- **Monitoring Services** → Alert Service: Send notifications

**Communication Requirements:**
- **Low Latency**: API Gateway to microservice calls must complete <100ms (user-facing latency critical)
- **High Throughput**: 100+ API calls/second during peak hours
- **Reliability**: Failed calls must retry; timeouts managed gracefully
- **Monitoring**: Track request/response times, error rates, latencies
- **Versioning**: Deploy new service versions without breaking existing clients
- **Load Balancing**: Distribute requests across service replicas
- **Type Safety**: Prevent mismatched request/response types between services

**Scale Requirements:**
- 50+ pods across GKE cluster
- 100+ concurrent requests to individual services
- Cross-service call chains (e.g., API Gateway → Booking → Payment → Bank)
- Future multi-region expansion

**Integration Points:**
- External payment gateway (REST only)
- External notification providers (REST/SMTP)
- Internal Pub/Sub events (asynchronous)
- BigQuery analytics (queries)

## Decision

**We will use gRPC as the primary inter-service communication protocol (internal microservices) and REST APIs as the secondary protocol (public clients and external integrations).**

This hybrid approach provides:
- **gRPC for Microservices**: Low-latency, strongly-typed, efficient binary protocol for internal service-to-service communication
- **REST for Public APIs**: Familiar, HTTP-based, browser-friendly for external clients (web/mobile apps, third-party integrations)
- **Best of Both Worlds**: Performance benefits of gRPC internally; accessibility of REST externally

## Key Differentiators

### gRPC
- **Binary Protocol Efficiency**: gRPC uses Protocol Buffers (binary serialization), 7x smaller than JSON. Reduces bandwidth and latency.
- **Strongly-Typed**: Protocol Buffers generate type-safe client libraries. Compiler catches interface mismatches at compile-time, not runtime.
- **Multiplexing**: gRPC uses HTTP/2 multiplexing; multiple concurrent requests over single connection. Lower overhead than REST's request-per-connection model.
- **Bi-Directional Streaming**: gRPC supports client-streaming, server-streaming, and full-duplex bidirectional communication. Impossible with REST.
- **Performance**: gRPC latency typically <10ms over LAN; REST latency 20-50ms. Critical for low-latency requirements.
- **Language Agnostic**: gRPC works across Go, Python, Node.js, Java, etc. Polyglot microservices supported natively.

### REST APIs
- **Simplicity**: REST is simple, human-readable. Easy to debug via curl/Postman. No special tools needed.
- **Browser-Friendly**: REST calls work directly from JavaScript; no special gRPC client needed. Suited for web/mobile clients.
- **Cache-Friendly**: REST GET requests can be cached by proxies/CDNs. gRPC POST requests non-cacheable.
- **Widespread Adoption**: Every developer knows REST. Ecosystem tools (API gateways, documentation generators) mature.
- **External Integration**: Payment gateways, notification services, third-party APIs typically offer REST. Easier integration than gRPC.
- **Flexible**: REST loose coupling allows different versions running simultaneously. gRPC strict typing means versions must align.

## Alternatives Considered

- **REST APIs Only (No gRPC)**  
  Using REST for all service-to-service communication:
  - **Latency Overhead**: REST over HTTP/1.1 incurs overhead per request. Service chains (API Gateway → Booking → Payment → Bank) accumulate latency. At scale, becomes noticeable.
  - **Bandwidth**: JSON serialization verbose; at 100+ requests/second, bandwidth cost increases.
  - **Coupling Risk**: JSON's flexibility enables loose typing; request/response mismatches difficult to catch.
  - **Not Optimal**: REST is fine for external APIs; sub-optimal for internal service-to-service where performance matters.

- **gRPC Only (No REST)**  
  Using gRPC for all service-to-service and external APIs:
  - **External Integration Difficult**: Payment gateways, third-party APIs typically don't support gRPC. Would require custom REST wrappers (defeating performance gain).
  - **Client Complexity**: Web/mobile clients must use gRPC-web (not fully browser-compatible). More complex client setup.
  - **Debugging Difficulty**: gRPC binary protocol hard to debug (curl doesn't work). Requires grpcurl or other specialized tools.
  - **Ecosystem Maturity**: REST ecosystem more mature; fewer edge cases and bugs in REST implementations vs. gRPC.

- **Message Queue (RabbitMQ / Kafka) Only**  
  Async communication via message queues for all inter-service calls:
  - **Not Real-Time**: Booking Service queues payment authorization; waits for async response. Introduces latency and complexity.
  - **Eventual Consistency**: Message queue doesn't guarantee immediate delivery; creates inconsistency windows.
  - **Operational Complexity**: Requires message queue infrastructure (RabbitMQ, Kafka cluster setup and monitoring).
  - **Wrong Tool**: Message queues excel at event streaming (payment events, alerts) but are poor for request-response patterns.
  - **Use as Complement**: Message queues should supplement (not replace) REST/gRPC for async events (Pub/Sub already chosen for events).

- **SOAP (Simple Object Access Protocol)**  
  XML-based RPC protocol:
  - **Legacy Technology**: SOAP was popular 15+ years ago; modern industry shifted to REST/gRPC.
  - **Verbosity**: XML serialization verbose; larger than JSON or Protocol Buffers.
  - **Complexity**: SOAP WSDLs complex to work with; tooling not as mature as REST/gRPC.
  - **Not Recommended**: SOAP rarely chosen for new projects.

- **JSON-RPC**  
  Lightweight JSON-based RPC protocol:
  - **No Performance Advantage over REST**: JSON serialization same as REST; no latency/bandwidth benefit.
  - **No Strong Typing**: JSON-RPC lacks strict typing; same loose coupling issues as REST.
  - **Smaller Ecosystem**: Fewer tools and frameworks compared to REST or gRPC.
  - **Not Recommended**: REST accomplishes same goals with larger ecosystem.

- **Apache Thrift**  
  Binary RPC framework:
  - **Similar to gRPC**: Thrift (binary) provides same benefits as gRPC (performance, typing).
  - **Less Popular**: gRPC has stronger industry adoption; larger community and ecosystem.
  - **Less Mature for Kubernetes**: gRPC has better Kubernetes/container orchestration integration.
  - **Recommend gRPC Over Thrift**: Both similar; gRPC preferred for new projects.

## Why Hybrid gRPC + REST is Better Than Alternatives

| Criterion | gRPC + REST | REST Only | gRPC Only | Kafka Only | SOAP |
|-----------|-------------|-----------|-----------|-----------|------|
| **Service-to-Service Latency** | Excellent (gRPC) | Fair (REST) | Excellent | Poor (async) | Poor |
| **External API Integration** | Excellent (REST) | Excellent | Poor (gRPC-web) | Poor | Fair |
| **Type Safety** | Excellent (gRPC) | Fair (JSON) | Excellent | Poor | Good (XML) |
| **Debugging Ease** | Good (REST) | Excellent | Poor (binary) | Medium | Poor |
| **Bandwidth Efficiency** | Excellent (gRPC) | Good (JSON) | Excellent | Medium | Poor (XML) |
| **Mobile Client Support** | Good (both) | Excellent | Limited (gRPC-web) | N/A | Limited |
| **Caching Capability** | Good (REST) | Good (HTTP caching) | None (POST) | None | Limited |
| **Streaming Support** | Excellent (gRPC) | Limited (webhooks) | Excellent | Excellent | Limited |
| **Ecosystem Maturity** | Excellent (both) | Excellent | Very Good | Very Good | Mature but old |
| **Learning Curve** | Medium (both) | Low (REST familiar) | Medium (gRPC new) | Medium | High |

**Why hybrid wins:**

1. **Performance Optimized**: gRPC for internal service-to-service where latency critical (Booking → Payment <10ms). REST for external where simplicity matters.

2. **Type Safety Internal**: gRPC's Protocol Buffers catch interface mismatches at compile time. Prevents runtime errors in critical payment flows.

3. **External Integration**: REST for payment gateway, notifications, third-party APIs. gRPC wouldn't work anyway; REST mandatory.

4. **Scalability**: gRPC's HTTP/2 multiplexing handles 100+ concurrent requests efficiently. REST requires more connections.

5. **Debugging**: REST APIs debuggable via curl; gRPC debuggable via grpcurl. Both reasonable options.

6. **Ecosystem**: gRPC mature in cloud-native (Kubernetes); REST mature everywhere. Both excellent.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Complexity of Dual Protocols** | Supporting both gRPC and REST increases framework/library choices; potential for inconsistency | Establish standards (gRPC for service-to-service, REST for external), document clearly, implement shared validation libraries |
| **Protocol Mismatch Bugs** | Developers might use wrong protocol (gRPC where REST needed); causes integration failures | Code review checklist ("Is this an internal or external service?"), linting rules to enforce protocol choice |
| **Load Balancing Complexity** | gRPC requires HTTP/2-aware load balancing; standard HTTP load balancers might not work | Use GKE's native load balancing (supports HTTP/2), test gRPC traffic under load, monitor connection distribution |
| **gRPC Debugging Difficulty** | Binary protocol hard to debug vs. REST's human-readable JSON | Distribute grpcurl for debugging, use grpc-web for browser debugging, implement detailed gRPC logs |
| **Version Incompatibility** | gRPC client/server version mismatch harder to detect than REST | Implement semantic versioning for Protocol Buffers, test backward compatibility before deployment, use proto versioning strategy |
| **Middleware Compatibility** | Some middleware (load balancers, proxies, firewalls) might not support gRPC/HTTP2 | Test middleware with gRPC traffic, configure firewall rules for HTTP/2, use gRPC-compatible load balancers |
| **Monitoring Complexity** | Monitoring gRPC traffic different than REST (no standard HTTP metrics); requires specialized tooling | Use gRPC-aware monitoring (Envoy proxy, Istio service mesh), emit custom metrics to Cloud Monitoring |
| **Team Learning Curve** | Team unfamiliar with gRPC might struggle with Protocol Buffers, go generate commands, etc. | Provide gRPC training, maintain gRPC code examples, start with simple services before complex ones |
| **Connection Pool Exhaustion** | gRPC maintains persistent connections; too many clients/servers could exhaust connection limits | Implement connection pooling, set conservative connection limits, use load balancing to distribute connections |
| **Cascading Timeout Issues** | Service chain latency compounds (API Gateway → A → B → C). Timeout tuning complex | Implement timeout strategy per service, use circuit breakers, test under load to determine optimal timeouts |

## Conclusion

The hybrid gRPC + REST approach is optimal because:

1. **Performance-Optimized**: gRPC for internal services where latency critical (<10ms). REST's 20-50ms overhead acceptable for external integrations.

2. **Type Safety**: gRPC's Protocol Buffers catch interface mismatches at compile-time. Prevents bugs in critical payment/ticketing flows.

3. **External Integration**: REST for payment gateways, third-party APIs (they don't support gRPC anyway).

4. **Scalability**: gRPC HTTP/2 multiplexing handles high concurrent load. Fewer connections = lower resource usage.

5. **Debuggability**: REST APIs curl-friendly for quick debugging. gRPC has grpcurl alternative.

6. **Ecosystem**: Both have mature ecosystems. gRPC excellent in Kubernetes; REST excellent everywhere.

**Recommendation**:
- **Service-to-Service (Internal)**: Use gRPC for Booking ↔ Payment, Booking ↔ Issuance, Issuance ↔ Notification, Validation ↔ DB queries
- **External Clients**: Use REST for web/mobile apps accessing API Gateway
- **Third-Party Integrations**: Use REST for payment gateway (Stripe), notification providers (Twilio, SendGrid)
- **Async Events**: Use Pub/Sub for events (payment completed, ticket issued, animal alert)
- **Protocol Buffer Versioning**: Use backward-compatible versioning; support 2 protocol versions during deployment
- **Load Balancing**: Configure GKE to route gRPC traffic via HTTP/2-aware load balancer
- **Monitoring**: Export gRPC metrics to Cloud Monitoring; alert on latency/error spikes
- **Testing**: Test gRPC service chains under peak load; measure end-to-end latency
- **Documentation**: Document protocol choice per service endpoint; provide client library setup instructions
- **Migration Path**: Start REST-only; migrate internal services to gRPC as needed (performance bottleneck proven)
