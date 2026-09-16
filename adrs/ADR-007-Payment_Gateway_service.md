# ADR-007 — Choice of Stripe/Adyen as External Payment Gateway Provider

## Context

The ticketing system processes payments for:
- Individual ticket purchases (standard and family passes)
- Repeat visitor memberships and loyalty programs
- Group bookings for large parties
- Promotional discounts and seasonal offers

**Payment Requirements:**
- Process 5,000-15,000 daily visitors with potential 3x growth
- Support multiple payment methods: credit/debit cards, digital wallets (Apple Pay, Google Pay), bank transfers
- Compliance with PCI-DSS (Payment Card Industry Data Security Standard)
- Fraud detection and prevention
- Recurring billing for season passes and membership programs
- Settlement and reconciliation reporting
- Multi-currency support if expanding to international visitors
- High availability (payment failures directly impact revenue)

**Security Requirements:**
- Payment Card Industry Data Security Standard (PCI-DSS) Level 1 compliance
- Tokenization to avoid storing raw card data
- End-to-end encryption
- 3D Secure / Strong Customer Authentication (SCA) support
- Fraud detection and chargeback protection
- Audit logging and compliance reporting

**Integration Requirements:**
- Payment Service microservice in GKE needs to integrate with payment provider
- Order/payment status updates to Cloud SQL
- Payment events published to Pub/Sub for analytics
- Webhook support for asynchronous payment status updates
- Webhook signature verification for security

**Scale Requirements:**
- Peak transaction rate: 100+ transactions/minute during peak hours
- 99.99% uptime requirement (payment downtime = revenue loss)
- Sub-3 second payment processing time (user experience critical)

## Decision

**We will use either Stripe or Adyen as the external Payment Gateway provider.**

Both Stripe and Adyen are enterprise-grade payment processors offering PCI-DSS compliance, global payment method support, fraud detection, and robust APIs for integration. The choice between them depends on specific regional, feature, and pricing requirements. A comparative analysis is provided below; either provider is acceptable for this system.

## Key Differentiators (Stripe vs. Adyen)

### Stripe
- **Developer Experience**: Stripe's API and documentation are industry-leading; the most developer-friendly payment platform. Onboarding and integration are fastest.
- **Global Payment Methods**: 135+ payment methods across 195+ countries; extensive coverage for diverse visitor demographics.
- **Fraud Detection**: Stripe Radar provides ML-powered fraud prevention. Chargeback protection via Stripe Disputes API.
- **Flexible Pricing**: Percentage-based (2.2% + $0.30 per card transaction) with transparent fees. Lower volumes can benefit from favourable interchange rates.
- **Ecosystem**: Stripe Billing for recurring charges (season passes), Stripe Terminal for in-person payments, Stripe Connect for marketplace scenarios.
- **Webhooks**: Robust webhook system with replay and debugging tools.
- **Settlement**: Daily or weekly payouts; configurable settlement schedules.

### Adyen
- **Enterprise Focus**: Designed for large enterprises; global reach with local presence in 190+ countries.
- **Omnichannel**: Unified platform for online, mobile, in-store, and recurring payments. If expanding to physical ticket booths (Adyen Terminal), single platform advantage.
- **Advanced Routing**: Intelligent routing optimizes payment success rates by trying multiple acquirers/payment methods.
- **Fraud Management**: Proprietary fraud scoring system with ML; historical data from 800B+ transactions.
- **Compliance**: Extensive compliance coverage (GDPR, PCI-DSS Level 1, local regulations).
- **Cost**: Interchange-optimized pricing; larger volumes benefit from negotiated rates. Higher base cost than Stripe for small volumes.
- **Reporting**: Detailed transaction and settlement reporting; strong analytics.

## Alternatives Considered

- **PayPal**  
  Mature payment processor with global reach. However:
  - **Higher Fees**: 3.49% + $0.49 per transaction for online payments; more expensive than Stripe/Adyen for high volume.
  - **Customer Friction**: PayPal redirects customers away from the booking site during checkout; reduces conversion rates.
  - **Limited Payment Methods**: Fewer payment methods than Stripe/Adyen; important for international visitor diversity.
  - **Developer Experience**: API is less elegant than Stripe; higher integration complexity.

- **Square**  
  Payment processor with strong in-person (POS) presence. However:
  - **UK/International Limited**: Square's coverage outside USA is limited. For estate in UK/Europe, local payment methods limited.
  - **Focus on Retail**: Optimized for retail/restaurants; less suitable for event ticketing.
  - **Higher Online Fees**: 2.9% + $0.30 per transaction; same as Stripe but no recurring billing optimization.
  - **Webhook Reliability**: Historically weaker webhook support compared to Stripe/Adyen.

- **Worldpay**  
  Large payment processor owned by FIS. However:
  - **Enterprise Only**: Worldpay targets large enterprises; minimum transaction volumes and annual fees apply.
  - **Complex Integration**: Requires more setup than Stripe/Adyen; longer integration timeline.
  - **Pricing**: Enterprise model makes cost comparison difficult; typically higher for mid-market volumes.

- **2Checkout (Verifone)**  
  Global payment processor. However:
  - **Lower Adoption**: Fewer merchants using 2Checkout; less mature ecosystem and community support.
  - **Documentation**: API documentation less comprehensive than Stripe/Adyen.
  - **Reputation**: Less established reputation than Stripe/Adyen in the ticketing/events industry.

- **In-House Payment Processing**  
  Building custom payment processing using Stripe/Adyen as sole acquirer:
  - **Compliance Risk**: Implementing PCI-DSS compliance in-house is error-prone. Compliance breaches result in data loss, fines, and reputation damage.
  - **Operational Burden**: Maintaining payment processing infrastructure, handling disputes, managing chargebacks all require specialized expertise.
  - **Unrealistic for MVP**: Proper payment infrastructure is 3-6 months of development alone. Not recommended unless business is mature.

- **Cryptocurrency Payments**  
  Bitcoin, Ethereum, stablecoins:
  - **Limited Acceptance**: Estate visitors unlikely to pay with cryptocurrency. Adoption among general population still <5%.
  - **Volatility**: Price fluctuations make revenue forecasting difficult.
  - **Regulatory Uncertainty**: Crypto regulations still evolving; compliance unclear.
  - **Not Suitable**: Inappropriate for traditional ticketing business.

## Why Stripe/Adyen Are Better Than Alternatives

| Criterion | Stripe | Adyen | PayPal | Square | Worldpay |
|-----------|--------|-------|--------|--------|----------|
| **Transaction Fees** | 2.2% + $0.30 | ~2.5% + acquirer fees | 3.49% + $0.49 | 2.9% + $0.30 | Negotiated (higher) |
| **Payment Methods** | 135+ (global) | 190+ (global) | Limited (35+) | Moderate (40+) | Good (100+) |
| **PCI Compliance** | Level 1 managed | Level 1 managed | Level 1 managed | Level 1 managed | Level 1 managed |
| **Developer Experience** | Excellent | Very Good | Good | Good | Fair |
| **API Documentation** | Excellent | Very Good | Good | Good | Fair |
| **Fraud Detection** | Excellent (Radar) | Excellent (proprietary) | Good | Good | Good |
| **Webhook Support** | Excellent | Very Good | Good | Fair | Fair |
| **Recurring Billing** | Excellent (Stripe Billing) | Good | Moderate | Limited | Good |
| **Settlement Speed** | 1-2 days | 1-2 days | 1-2 days | 1-2 days | 1-3 days |
| **Global Reach** | 195+ countries | 190+ countries | 200+ countries | Limited (50+ countries) | Global but complex |
| **Integration Time** | Fast (days) | Medium (weeks) | Medium (weeks) | Fast (days) | Slow (months) |
| **Enterprise Support** | Good | Excellent | Moderate | Limited | Excellent |
| **Cost at 5K txns/day** | ~$800-1200/mo | ~$1000-1500/mo | ~$1400-1800/mo | ~$1000/mo | $2000+/mo |

**Recommendation for this project**: **Stripe is recommended for initial MVP phase** due to:
1. Fastest integration time (days vs. weeks)
2. Most developer-friendly documentation
3. Appropriate pricing for mid-market volume
4. Strong fraud detection and chargeback protection
5. Excellent recurring billing for loyalty programs

**Upgrade to Adyen if**: Business scales significantly (50K+ daily transactions), needs omnichannel payment support (in-person kiosks), or operates in multiple regions requiring local payment methods optimization.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Vendor Lock-in** | Switching payment providers mid-production is difficult; years of transaction history locked in provider | Design Payment Service abstraction layer to isolate provider specifics, maintain provider-agnostic schemas in Cloud SQL, document migration path if needed, evaluate provider annually |
| **Webhook Reliability** | Payment provider webhooks can be delayed, duplicated, or lost; Payment Service might miss status updates | Implement idempotent webhook handlers (payment_id deduplication), reconcile payment status with provider API periodically (e.g., hourly), implement retry logic with exponential backoff, log all webhook events for debugging |
| **PCI Compliance Complexity** | PCI-DSS requirements evolve; compliance violations result in fines and data loss | Never store raw card data; always tokenize via provider's hosted forms, use provider-managed vault for tokens, implement annual PCI compliance audits, educate team on security best practices |
| **Fraud False Positives** | Fraud detection system might block legitimate transactions; estate visitors experience payment rejections | Configure fraud threshold conservatively (favor false negatives over false positives), implement manual review queue for borderline transactions, enable customer support override for legitimate blocks, monitor false positive rate |
| **Payment Processing Latency** | Provider API latency (100-500ms) adds to checkout time; slow checkouts reduce conversion | Implement client-side optimistic UI (show success before server confirmation), batch authorization checks if possible, use provider's async payment APIs where applicable, monitor p99 latency and alert on degradation |
| **Regional Payment Method Gaps** | If estate expands internationally, provider might not support required local payment methods | Research provider's payment method coverage in target regions before expansion, maintain fallback payment methods (bank transfer), negotiate with provider for region-specific methods if needed |
| **Chargeback Disputes** | Customer disputes or fraudulent chargebacks result in revenue loss | Implement strong order verification (confirmation emails, order ID in receipt), maintain detailed transaction logs for dispute evidence, respond to chargeback disputes within provider's window, implement Stripe Disputes API for automation |
| **Settlement Delay** | Funds settle 1-2 business days after transaction; cash flow impact | Plan working capital to cover settlement lag, forecast based on settlement schedules, use provider's early payouts feature if available (at higher cost) |
| **API Rate Limits** | Provider rate limits might throttle Payment Service during peak load | Implement client-side rate limiting before hitting provider limits, use payment service request queuing to smooth spikes, test load under peak conditions, maintain contact with provider for limit increases |
| **Provider Outage Risk** | Payment provider downtime blocks all checkout functionality | Implement graceful degradation (offline mode with email payment requests), maintain warm backup provider credentials, set up provider status monitoring with alerts, communicate outages to customers transparently |

## Conclusion

Stripe or Adyen are the optimal choices for external payment processing because:

1. **PCI Compliance Outsourced**: Payment providers manage PCI-DSS compliance; Payment Service never handles raw card data. Reduces security risk and compliance burden.

2. **Global Payment Methods**: 135+ (Stripe) or 190+ (Adyen) payment methods support diverse visitor demographics without custom integration per method.

3. **Fraud Detection**: ML-powered fraud prevention reduces chargeback losses and false declines. Proprietary models built on billions of transactions.

4. **High Availability**: 99.99%+ uptime SLA ensures checkout availability during peak visitor hours. No revenue loss from payment infrastructure failure.

5. **Recurring Billing**: Stripe Billing enables season passes and membership programs; Adyen supports similar workflows. Increases customer lifetime value.

6. **Settlement & Reporting**: Automated settlement, detailed transaction reporting, and reconciliation integration with Cloud SQL accounting.

7. **Webhook Integration**: Robust webhook system for real-time payment status updates to Core systems and analytics.

8. **Minimal Operational Overhead**: Outsourcing payment processing eliminates PCI compliance, security, and infrastructure management.

**Recommendation**: 
- **For MVP (0-50K daily transactions)**: Use **Stripe**. Fastest integration (1-2 weeks), most developer-friendly, appropriate pricing, excellent documentation.
- **Payment Service Implementation**: Create abstraction layer wrapping provider-specific APIs. Use provider tokens for recurring payments. Implement webhook handlers for payment status updates. Publish payment events to Pub/Sub for analytics. Reconcile provider transaction history with Cloud SQL daily.
- **Security Setup**: Enable TLS for all API communication, validate webhook signatures, store provider keys in Secret Manager, implement rate limiting and request logging.
- **Monitoring**: Track payment success rates, error rates by error type, processing latency, and webhook delivery lag. Alert on anomalies.
- **Upgrade Path**: If business scales to 50K+ daily transactions, evaluate Adyen for better interchange rates and omnichannel capabilities.
