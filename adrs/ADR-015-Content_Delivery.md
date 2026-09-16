# ADR-015 — Choice of Google Cloud CDN for Content Delivery

## Context

The estate system serves web and mobile clients with static and dynamic content:

**Content Types:**
- **Static Web Assets**: HTML, CSS, JavaScript, images (tickets, promotions, animal photos)
- **Mobile App Assets**: App binaries, updates, resources
- **API Responses**: JSON responses from microservices
- **Media Files**: Photos/videos of animals and rides, promotional videos
- **Receipts/Invoices**: PDF receipts, email attachments

**User Distribution:**
- **Geographic Spread**: Visitors from across UK/Europe (if expanding internationally)
- **Peak Load**: 5,000-15,000 daily visitors with spiky traffic (weekends peak, weekdays lower)
- **Mobile-Heavy**: 80% of traffic via mobile app (WiFi on estate unreliable)
- **International Growth**: Potential expansion to other countries in future

**Performance Requirements:**
- **Low Latency**: Web/app load time <2s (user experience critical)
- **High Availability**: Content delivery 99.99% uptime (no lost revenue from site downtime)
- **Bandwidth Efficiency**: Minimize data transfer (mobile data plans expensive for visitors)
- **Cache Efficiency**: Frequently accessed content (animal photos, ride info) cached to reduce origin load

**Content Delivery Requirements:**
- Serve static assets from edge locations near users
- Cache control (different TTL for different content types)
- Cache invalidation (instantly purge outdated content)
- HTTPS/TLS for all content (security)
- DDoS protection (protect against attacks)
- Custom domain support (theestates.co.uk)

## Decision

**We will use Google Cloud CDN for content delivery, integrated with Cloud Load Balancer and Cloud Armor.**

Cloud CDN distributes content across Google's global edge network, integrates natively with GCP services, provides DDoS protection via Cloud Armor, and scales automatically to handle peak traffic—ideal for this use case.

## Key Differentiators

- **Global Edge Network**  
  Google operates edge locations across 140+ countries. Content cached at edges nearest users. Visitors in London, Paris, Berlin each served from nearby edge (low latency).

- **Native GCP Integration**  
  Cloud CDN integrates with Cloud Load Balancer, Cloud Armor, Cloud Storage, GKE. No separate infrastructure; unified management within GCP console.

- **Automatic Scaling**  
  CDN scales automatically to handle traffic spikes. Peak traffic (15,000 visitors during weekend) handled without degradation or manual intervention.

- **Cache Control Flexibility**  
  Set cache TTL per path (API responses 0s, HTML 5m, static assets 1 day, media files 30 days). Granular caching strategy.

- **Cache Invalidation**  
  Instantly purge cached content by URL pattern. Deploy new animal photo; invalidate cache; users see new version immediately.

- **Signed URLs**  
  Generate signed URLs for protected content (invoices, tickets). Content accessible only via signed URL; no unauthorized access.

- **DDoS Protection (Cloud Armor)**  
  Cloud Armor integrates with Cloud CDN to protect against DDoS, WAF attacks. Rules-based protection (block by country, rate limit by IP).

- **Compression**  
  Automatic compression (gzip, brotli) for text content. JavaScript/CSS/HTML reduced 60-80%; faster download.

- **HTTP/2 and HTTP/3**  
  Modern protocols (HTTP/2, HTTP/3) enabled by default. Multiplexing reduces round trips; faster loading.

- **Access Logs & Metrics**  
  Cloud CDN exports metrics to Cloud Monitoring. Track cache hit rate, latency, error rate. Optimize caching strategy based on data.

- **Cost-Effective**  
  CDN reduces bandwidth from origin (Cloud Load Balancer); only cache misses hit origin. Savings scale with traffic.

- **No Origin Dependency**  
  Content cached at edges; origin can be slow/unreliable; CDN serves cached content to users. Isolates origin performance from user experience.

## Alternatives Considered

- **Cloudflare**  
  Third-party CDN and DDoS protection service:
  - **External Dependency**: Cloudflare outages block site. Limited control over Cloudflare infrastructure.
  - **Cost**: Cloudflare pricing variable ($20/month - $200+/month depending on volume). GCP CDN cheaper for GCP-hosted content.
  - **Not GCP Native**: Cloudflare requires nameserver delegation to Cloudflare (DNS change). GCP CDN integrates within GCP console.
  - **Suitable Alternative**: Cloudflare viable but adds vendor; Cloud CDN preferred for GCP-hosted systems.

- **Akamai**  
  Enterprise CDN service:
  - **Very High Cost**: Akamai enterprise pricing $1,000+/month minimum. Not suitable for mid-market traffic.
  - **Enterprise-Focused**: Designed for large enterprises; overkill for estate system.
  - **Complexity**: Akamai setup complex; months of configuration.
  - **Not Recommended**: Cloud CDN more suitable.

- **Amazon CloudFront**  
  AWS's CDN service:
  - **Cloud Lock-in**: Selecting AWS CloudFront contradicts GCP platform choice (GKE, BigQuery, Cloud SQL). Multi-cloud complexity without benefit.
  - **Cross-Cloud Integration**: GCP-hosted origin (Cloud Load Balancer) to CloudFront edge adds latency and complexity.
  - **Cost**: CloudFront pricing similar to Cloud CDN but less integrated with GCP.
  - **Not Recommended**: Cloud CDN better for GCP-hosted systems.

- **Azure CDN**  
  Microsoft's CDN service:
  - **Same Issues as CloudFront**: Platform lock-in, cross-cloud complexity, less integrated with GCP.

- **Self-Hosted Caching (Nginx, Varnish)**  
  Running cache proxy on Compute Engine:
  - **Self-Managed Complexity**: Requires Nginx/Varnish cluster setup, replication, failover. Operational burden.
  - **Limited Geographic Distribution**: Self-hosted cache in GCP region only. No global edge network like Cloud CDN.
  - **Scaling Complexity**: Adding cache capacity requires provisioning more Compute Engine instances.
  - **Not Recommended**: CDN services provide better geographic distribution.

- **No CDN (Origin-Only)**  
  Serving content directly from Cloud Load Balancer without CDN:
  - **Geographic Latency**: European visitors see 100-200ms latency to GCP origin. Noticeable for downloads.
  - **Origin Overload**: Peak traffic (15,000 visitors) overwhelms origin server. 50+ requests/second sustained.
  - **Bandwidth Cost**: All traffic hits origin; expensive data transfer charges.
  - **Not Suitable**: CDN necessary for performance and cost.

- **Amazon S3 + CloudFront**  
  Using AWS storage and CDN:
  - **Same Cloud Lock-in**: Contradicts GCP platform choice. Multi-cloud complexity.
  - **Cross-Cloud Transfer**: GCP origin to S3 to CloudFront adds latency and egress cost.
  - **Not Recommended**: Cloud CDN with Cloud Storage better for GCP.

## Why Cloud CDN is Better Than Alternatives

| Criterion | Cloud CDN | Cloudflare | CloudFront | Self-Hosted | No CDN |
|-----------|-----------|-----------|-----------|-------------|---------|
| **Global Edge Network** | 140+ countries | 180+ countries | 450+ edge locations | Single region | Single region |
| **GCP Integration** | Native | Via DNS delegation | No | Limited | N/A |
| **Cache TTL Control** | Per-path | Yes | Yes | Yes | N/A |
| **DDoS Protection** | Cloud Armor | Built-in | Via Shield | Custom | Custom |
| **Cost (1TB/month traffic)** | $50-100/mo | $20-50/mo | $50-100/mo | $100+/mo | $0 (high origin cost) |
| **Setup Complexity** | Low (hours) | Medium (hours) | Low (hours) | High (days) | None |
| **Operational Burden** | Minimal (managed) | Minimal | Minimal | High (self-managed) | None (high origin load) |
| **Origin Decoupling** | Excellent | Excellent | Excellent | Moderate | None |
| **Performance (user latency)** | <50ms (edge) | <50ms (edge) | <50ms (edge) | 100-200ms (origin) | 100-200ms (origin) |
| **Reliability** | 99.95% SLA | 99.99% SLA | 99.99% SLA | Self-dependent | Self-dependent |
| **Cache Invalidation** | Instant (per URL) | Instant | Instant (15min default) | Instant | N/A |

**Why Cloud CDN wins:**

1. **Native GCP Integration**: Single console for CDN, Load Balancer, Cloud Armor, Cloud Storage. No external vendor.

2. **Global Edge Network**: 140+ countries means visitors worldwide see <50ms latency. Geographic redundancy improves resilience.

3. **DDoS Protection**: Cloud Armor integrates directly; rules-based protection. No separate service to manage.

4. **Automatic Scaling**: Traffic spikes handled automatically. Peak weekends don't require infrastructure provisioning.

5. **Cache Efficiency**: Granular TTL control, instant invalidation, compression, signed URLs. Flexible caching strategy.

6. **Cost-Effective**: CDN reduces origin bandwidth; savings scale with traffic volume. Cheaper than self-hosted alternative.

7. **Performance**: Edge caching means <50ms latency for geographic users. Direct origin access would be 100-200ms.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Cache Staleness** | Cached content might be outdated; users see stale data if TTL too long | Set appropriate TTL (static assets 1 day, HTML 5min, API 0s), implement cache invalidation for critical updates, test cache behavior before production |
| **Cache Bypass Risk** | Users might access origin directly (bypass CDN) via IP address; CDN protection bypassed | Restrict firewall to allow CDN IPs only, use Cloud Armor to block direct-to-origin requests, use signed URLs for sensitive content |
| **Cost at High Traffic** | CDN egress charges could exceed budget if traffic unexpectedly high | Monitor bandwidth usage via Cloud Monitoring, set alerts on estimated egress cost, optimize compression, set cache TTL to reduce misses |
| **Origin Failure Impact** | CDN serves cached content; if origin down and cache expires, CDN returns 503 error | Set generous cache TTL for error pages, implement origin health monitoring, use Cloud CDN's negative cache (cache errors briefly), set up monitoring/alerts |
| **Geographic Compliance** | Some regions require data residency (content must stay in-country); CDN edge locations might violate rules | Identify data residency requirements early, use Cloud CDN restrictions (disable caching for specific regions if needed), store sensitive data separately |
| **Debugging Complexity** | CDN caching can mask origin issues (slow origin still shows fast CDN response); debugging origin performance difficult | Bypass CDN during debugging (add X-Bypass-Cache header), monitor origin metrics separately, implement cache-busting for testing |
| **SSL Certificate Management** | CDN edge certificates might expire; certificate renewal required across 140+ edge locations | Google manages certificates automatically; no manual renewal needed, but monitor expiration dates via Cloud Monitoring |
| **Cache Invalidation Latency** | Invalidating cached content takes 15 minutes to propagate across all edges | Accept 15-minute delay for non-critical updates, use signed URLs for immediate updates (new URL per version), implement versioning strategy |
| **DDoS Attack Complexity** | Large DDoS attacks might bypass CDN; origin still needs protection | Enable Cloud Armor rules, implement rate limiting, set traffic quotas, collaborate with Google DDoS team during attacks |
| **Third-Party Content Risks** | CDN caching third-party content (ads, analytics, promotional content) might have privacy implications | Review privacy policies for cached third-party content, implement cookie stripping for tracking prevention, use Content Security Policy headers |

## Conclusion

Cloud CDN is the optimal content delivery platform for the estate system because:

1. **Global Edge Network**: 140+ countries means low latency for worldwide users. Competitive advantage vs. origin-only.

2. **Native GCP Integration**: Seamless integration with Cloud Load Balancer, Cloud Armor, Cloud Storage, GKE. Single platform, unified management.

3. **Automatic Scaling**: Traffic spikes handled without manual intervention. Peak weekends (15,000 visitors) handled transparently.

4. **DDoS Protection**: Cloud Armor rules-based protection integrated directly with CDN. Block attacks at edge, not origin.

5. **Cache Flexibility**: Granular TTL, instant invalidation, compression, signed URLs. Fine-grained control over caching strategy.

6. **Cost-Efficiency**: CDN reduces origin bandwidth; savings scale with traffic. Cheaper than self-hosted caching or no CDN.

7. **Performance Impact**: Edge caching means <50ms latency for geographic users. vs. 100-200ms for direct origin. Significant UX improvement.

8. **Operational Simplicity**: Fully managed. No infrastructure setup, scaling, or maintenance. Google handles edge management.

**Recommendation**:
- Enable Cloud CDN on Cloud Load Balancer in front of GKE cluster
- Configure cache TTL by content type:
  - Static assets (JS, CSS, images): 1 day (86400s)
  - HTML pages: 5 minutes (300s)
  - API responses: 0 seconds (no cache)
  - Media files (photos, videos): 30 days
- Enable compression (gzip, brotli) for text content
- Use Cloud Armor to block DDoS, WAF attacks
- Implement cache invalidation strategy (versioning + invalidation for critical updates)
- Monitor cache hit rate via Cloud Monitoring (target >70%)
- Test cache behavior in staging before production
- Implement origin health monitoring (alert if origin down)
- Use signed URLs for sensitive content (tickets, invoices)
- Enable HTTP/2 and HTTP/3 for modern browsers
- Set up cost monitoring; alert on bandwidth spikes
- Document cache strategy in operations runbook
- Test disaster recovery (origin failure; CDN still serves cached content)
