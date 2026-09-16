# ADR-003 — Choice of Google Kubernetes Engine (GKE) for Cloud Platform

## Context

The estate system architecture comprises multiple interconnected microservices across three domains:

1. **Ticketing System**: 15+ microservices (API Gateway, Auth, Booking, Payment, Issuance, Validation, Loyalty, Notification, Reporting, etc.)
2. **Ride Monitoring System**: Telemetry Ingestion, Pub/Sub, Dataflow streaming, BigQuery analytics, Operations API
3. **Animal Health Monitoring System**: Similar pattern with health/feeding/population monitoring, Dataflow processing, BigQuery storage, Operations API

**Key Requirements:**
- Support 5,000-15,000 daily visitors with potential 3x growth
- Real-time data processing for ride safety and animal health monitoring
- Multiple stateless and stateful services running concurrently
- Inter-service communication with multiple databases (transactional DB, cache, BigQuery)
- Independent service deployment and scaling
- High availability and fault tolerance
- Team autonomy for different service domains

**Infrastructure Constraints:**
- Budget available for managed Kubernetes infrastructure
- Need seamless integration with GCP services (Pub/Sub, BigQuery, Dataflow, IAM)
- Requirement for reliable orchestration across multiple service tiers

## Decision

**We will use Google Kubernetes Engine (GKE) as the primary container orchestration platform for deploying and managing microservices in the cloud.**

GKE provides a managed, enterprise-grade Kubernetes cluster that enables efficient deployment, scaling, and management of the 30+ microservices across the three system domains while maintaining high availability and integrating deeply with GCP's ecosystem.

## Key Differentiators

- **Native Kubernetes Support**  
  GKE is Google's managed Kubernetes service, offering native Kubernetes API and ecosystem. This allows the team to use industry-standard orchestration patterns, tools, and knowledge. Unlike proprietary solutions, Kubernetes skills are transferable across organizations and clouds.

- **Deep GCP Integration**  
  GKE integrates seamlessly with GCP services used in the architecture: Pub/Sub (for event streaming), BigQuery (for analytics), Dataflow (for stream processing), Cloud Storage, Cloud IAM, Stackdriver/Cloud Monitoring, and VPC networking. This integration reduces operational overhead and eliminates middleware translation layers.

- **Optimal for Microservices Architecture**  
  The system has 30+ interdependent microservices. Kubernetes excels at managing microservice lifecycles through:
  - Service discovery and load balancing between services
  - Rolling updates and canary deployments
  - Network policies for secure inter-service communication
  - Sidecar patterns for observability and security

- **Stateful and Stateless Service Support**  
  The architecture mixes stateless services (API Gateway, Auth, Booking) with stateful components (Operations API + DB, Caches). Kubernetes StatefulSets handle persistent workloads, while Deployments manage stateless services. This flexibility is superior to serverless-only approaches.

- **Resource Efficiency and Cost Control**  
  Kubernetes enables precise resource requests and limits, bin-packing containers efficiently. Node auto-scaling matches cluster size to actual demand, reducing costs. Workload prioritization ensures critical services (animal health alerts, safety monitoring) get resources during contention.

- **Multi-Team Scalability**  
  Different teams manage different service domains (ticketing team, ride monitoring team, animal health team). Kubernetes namespaces provide logical isolation, RBAC enables fine-grained access control, and Helm charts allow teams to manage their services independently while staying within enterprise governance.

- **Observability and Debugging**  
  GKE integrates with Google Cloud Logging and Cloud Monitoring for centralized observability. Kubernetes' built-in audit logging, events system, and pod/container logs provide comprehensive visibility into system behavior—essential for complex microservice debugging.

- **Automatic High Availability**  
  GKE automates cluster management: automatic node repair/replacement, rolling updates, pod rescheduling on node failures, and multi-zone deployment options. This reduces operational burden compared to managing cluster infrastructure manually.

- **Industry Standard and Future-Proof**  
  Kubernetes is the de facto standard for container orchestration. Investments in Kubernetes expertise, tooling, and processes remain valuable across different cloud providers and on-premises deployments. This reduces long-term vendor lock-in risk.

## Alternatives Considered

- **Cloud Run (Serverless Containers)**  
  Google Cloud Run is a fully serverless container platform optimized for stateless, event-driven workloads. However:
  - **Stateful Services**: Cloud Run instances are ephemeral; running stateful services (Operations API + DB) is inefficient
  - **Interconnected Services**: Cloud Run lacks native service discovery and load balancing between services; inter-service communication requires manual management
  - **Cold Starts**: Request latency spikes occur when instances scale from zero—problematic for ride safety alerts and animal health monitoring
  - **Resource Overprovisioning**: Always-on services require reserved capacity, negating serverless cost benefits
  - **Network Complexity**: Multiple Cloud Run services require API Gateway and ingress configuration; no built-in mesh networking
  - **Database Connection Pooling**: Each Cloud Run instance manages its own connections; connection pool sizing becomes complex at scale
  - **Unsuitable Scale Model**: Designed for request-based scaling; the background processing workloads (Dataflow coordination, analytics batching) are better suited to container orchestration

- **Google App Engine (Standard or Flex)**  
  Google App Engine provides higher-level abstraction than Kubernetes:
  - **Less Control**: Less flexibility for custom container images, dependencies, and runtime configurations
  - **Service Isolation**: Inter-service communication is harder than in Kubernetes; no service mesh, limited routing options
  - **Stateful Limitations**: App Engine Flex can run stateful services but offers less control than Kubernetes StatefulSets
  - **Operational Overhead**: App Engine management plane adds abstraction layer without commensurate benefit for microservice-heavy architecture
  - **Cost Predictability**: App Engine pricing is less transparent for heterogeneous workloads (some services receive few requests, others many)

- **Compute Engine (VMs) with Manual Orchestration**  
  Managing container orchestration manually on Compute Engine VMs:
  - **Operational Burden**: Manual deployment, scaling, health checking, rolling updates, and failure recovery
  - **No Service Discovery**: Requires manual service registry (e.g., Consul, etcd) or complex DNS management
  - **Poor Resource Utilization**: Manual bin-packing is inefficient; servers often under-utilized
  - **Scaling Complexity**: Adding instances during traffic spikes requires custom logic, load balancer reconfiguration, and testing
  - **Team Friction**: Teams managing individual services would duplicate DevOps work; no standardized deployment patterns

- **ECS (Amazon Elastic Container Service)**  
  AWS's proprietary container orchestration platform:
  - **Cloud Lock-in**: Deep integration with AWS services makes future multi-cloud strategies difficult
  - **Limited Portability**: ECS-specific configurations don't transfer to other Kubernetes platforms
  - **Team Knowledge**: Team expertise in GCP and Kubernetes wouldn't transfer as easily to ECS
  - **Cross-cloud Growth**: If the company grows to multi-cloud, ECS expertise becomes obsolete

- **On-Premises Kubernetes**  
  Self-hosting Kubernetes in data centers:
  - **Operational Complexity**: Full responsibility for cluster provisioning, upgrades, security patching, and maintenance
  - **Infrastructure Cost**: Capital expense for servers, networking, storage, and redundancy
  - **Patchy WiFi Constraint**: On-premises infrastructure doesn't solve the estate's WiFi connectivity issues; still requires cloud platform for IoT data ingestion
  - **Staffing**: Requires dedicated Kubernetes operations team (cluster admins, security specialists)
  - **Scalability**: Manual capacity planning for growth to 15,000+ daily visitors

- **Hybrid Approach (Kubernetes + Cloud Run)**  
  Splitting workloads: Kubernetes for complex services, Cloud Run for simple, stateless functions:
  - **Operational Complexity**: Managing two orchestration platforms introduces dual expertise requirements
  - **Consistency**: Different deployment patterns, monitoring, logging, and alerting across platforms
  - **Service Interdependence**: Inter-platform service communication adds complexity (API calls instead of native service mesh)
  - **Cost**: Savings from Cloud Run's simplicity offset by operational overhead of dual-platform management

## Why GKE is Better Than Alternatives

| Criterion | GKE | Cloud Run | App Engine | Compute Engine | ECS |
|-----------|-----|-----------|-----------|-----------------|-----|
| **Microservice Orchestration** | Excellent | Fair | Fair | Poor | Good |
| **Service Discovery & Load Balancing** | Built-in | Manual | Limited | Manual | Good |
| **Stateful Service Support** | Excellent | Poor | Fair | Excellent | Good |
| **GCP Integration** | Excellent | Excellent | Excellent | Good | N/A |
| **Cold Start Latency** | None (persistent) | High | Low | None | Low |
| **Resource Efficiency** | Excellent | Fair (overprovisioning) | Good | Poor | Good |
| **Network Flexibility** | Excellent (mesh, policies) | Limited | Limited | Flexible | Good |
| **Multi-Team Isolation** | Excellent (namespaces, RBAC) | Limited | Limited | Manual | Good |
| **Observability** | Excellent | Good | Good | Manual | Good |
| **Cost Predictability** | Good | Fair (cold starts) | Good | Poor | Good |
| **Skill Transferability** | High (Kubernetes) | Low (proprietary) | Medium | Medium | Low |
| **Vendor Lock-in** | Low (K8s portable) | High (GCP-only) | Medium | Medium | High |
| **Scaling Complexity** | Automatic | Automatic | Semi-auto | Manual | Automatic |

**Why GKE wins for this architecture:**

1. **Microservice Complexity**: 30+ services require orchestration that handles service discovery, load balancing, rolling updates, and network policies. Cloud Run's serverless model isn't designed for this interconnectedness.

2. **Mixed Workload Types**: 
   - Stateless API services (API Gateway, Auth, Booking) ✓ GKE handles with Deployments
   - Stateful services (Operations API + DB, Cache) ✓ GKE handles with StatefulSets
   - Long-running processors (Dataflow coordination) ✓ GKE handles natively
   - Cloud Run only handles the first category efficiently

3. **Real-Time Safety Requirements**: Ride safety alerts and animal health monitoring can't afford Cloud Run's cold start latency. GKE maintains persistent services with predictable latency.

4. **GCP Ecosystem Fit**: Pub/Sub, BigQuery, Dataflow, and Cloud Logging integrate natively with GKE. Services deployed on GKE can use Workload Identity for secure, keyless authentication to other GCP services.

5. **Cost Efficiency**: The ticketing system will have relatively constant load (always need API Gateway, Auth running). Cloud Run would require reserved instances, eliminating cost savings. GKE's resource requests/limits and node auto-scaling provide better cost control for this workload profile.

6. **Team Autonomy**: Ticketing team, ride monitoring team, and animal health team can each own their services within GKE namespaces. RBAC policies prevent cross-team interference. Cloud Run offers no such namespace-like isolation for multi-team scenarios.

7. **Operational Consistency**: Single orchestration platform, single monitoring/logging stack, single deployment model. No cognitive overhead of managing Kubernetes + Cloud Run together.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Operational Complexity** | Kubernetes has a steep learning curve; misconfiguration can cause outages | Invest in team training, use managed GKE (removes cluster management burden), establish runbooks and SOP documentation, implement GitOps for standardized deployments |
| **Cost at Low Scale** | Kubernetes clusters have minimum resource overhead; underutilized clusters are expensive | Start with a single autopilot cluster, enable cluster autoscaling, monitor node utilization, consider consolidating low-traffic services onto shared cluster |
| **Over-Engineering Risk** | Kubernetes provides capabilities the initial system may not need | Begin with simple Deployments and StatefulSets; introduce advanced features (network policies, service mesh) as complexity grows |
| **Dependency on GCP Services** | Dataflow, BigQuery, Pub/Sub coupling makes multi-cloud difficult | Use abstraction layers for critical components (e.g., Kafka for Pub/Sub, define cloud-agnostic data pipeline interfaces); document migration paths |
| **Stateful Service Complexity** | Managing persistent volumes and databases in Kubernetes is complex | Use GCP Cloud SQL for managed databases instead of in-cluster databases; use managed cache (Cloud Memorystore) instead of in-cluster Redis |
| **Resource Contention** | Multiple services on shared cluster may compete for CPU/memory during peak load | Set appropriate resource requests/limits per service, use pod disruption budgets, implement horizontal pod autoscaling based on metrics |
| **Network Troubleshooting** | Kubernetes networking (DNS, service discovery, network policies) is complex to debug | Enable GKE logging and monitoring, use kubectl debugging tools (exec, port-forward, logs), maintain network policy documentation |
| **Version Compatibility** | Kubernetes version upgrades can introduce breaking changes | Enable GKE auto-upgrade on a maintenance window, test upgrades in staging cluster before production, monitor release notes |

## Conclusion

GKE is the optimal platform for this distributed, microservice-based architecture because:

1. **Best-Fit for Microservices**: The 30+ interdependent services require native service discovery, load balancing, and orchestration—Kubernetes's core strengths. Cloud Run's serverless model is optimized for single-function workloads, not service networks.

2. **Stateful Service Support**: Unlike Cloud Run, GKE natively supports both stateless and stateful services through Deployments and StatefulSets, eliminating the need for platform-specific workarounds for the Operations APIs and caching layers.

3. **Real-Time Responsiveness**: Persistent containers eliminate cold start latency, ensuring ride safety alerts and animal health monitoring respond immediately without serverless scaling delays.

4. **Deep GCP Integration**: Seamless integration with Pub/Sub, BigQuery, Dataflow, and Workload Identity enables secure, efficient data pipelines without middleware translation layers.

5. **Multi-Team Governance**: Kubernetes namespaces and RBAC provide the isolation and autonomy needed for separate teams (ticketing, ride monitoring, animal health) to manage their services independently while remaining within enterprise governance.

6. **Cost-Effective at Scale**: Node auto-scaling, resource request/limit precision, and workload prioritization provide predictable, efficient resource utilization as traffic grows to 15,000+ daily visitors.

7. **Future-Proof Investment**: Kubernetes expertise, tools, and patterns are transferable across cloud providers and on-premises deployments, reducing long-term vendor lock-in risk.

**Recommendation**: Deploy the platform on GKE using Autopilot mode to minimize cluster management overhead. Organize services into namespaces by business domain (ticketing, ride-monitoring, animal-health). Use managed GCP services (Cloud SQL for databases, Cloud Memorystore for caching, Pub/Sub for events) rather than running these in-cluster to further reduce operational burden. Establish a GitOps workflow (Flux/ArgoCD) for standardized, auditable deployments across the platform.
