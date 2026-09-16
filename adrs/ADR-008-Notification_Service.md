# ADR-008 — Choice of Google Cloud Messaging and Twilio for Notification Service

## Context

The ticketing and operations systems require multi-channel notifications for:

**Ticketing Notifications:**
- Order confirmation emails/SMS upon successful payment
- Ticket delivery (QR code, e-ticket attachment)
- Payment receipts and invoices
- Password resets and account recovery
- Promotion and discount alerts
- Loyalty points updates

**Operations Notifications:**
- Animal health alerts (temperature anomalies, feeding issues, population changes) to keepers
- Ride safety alerts (vibration anomalies, wear warnings) to ride operators
- System alerts (data ingestion delays, threshold breaches) to management

**Notification Scale:**
- Email volume: 5,000-15,000 confirmations/day with potential 3x growth
- SMS volume: 1,000-5,000 time-sensitive alerts/day
- WhatsApp/Push notifications: 500-2,000 real-time alerts/day for operations staff
- Peak load: 100+ notifications/second during peak hours

**Requirements:**
- Multi-channel support: Email, SMS, WhatsApp, push notifications
- High delivery reliability (>99% delivery rate)
- Low latency (emails within 1 minute, SMS within 30 seconds, push within 5 seconds)
- Templating support (order confirmation, alert formats)
- Unsubscribe/opt-out management
- Delivery status tracking and retry logic
- Cost-effective at scale
- Integration with microservices (Notification Service in GKE)

## Decision

**We will use Google Cloud Messaging (Firebase Cloud Messaging / FCM) for push notifications combined with Twilio for SMS and WhatsApp notifications, and SendGrid for email notifications.**

This combination provides:
- Native Firebase integration for app push notifications
- Twilio's reliable SMS/WhatsApp delivery
- SendGrid's email deliverability expertise
- Cost efficiency across all channels
- Minimal operational overhead

## Key Differentiators

### Google Cloud Messaging (Firebase Cloud Messaging)
- **Native Push Notifications**: Integrated with Google ecosystem; low latency push to Android and iOS devices
- **Cross-Platform Support**: Unified API for Android, iOS, and web notifications
- **Built-In GCP Integration**: Seamless authentication via Service Accounts, IAM, and Workload Identity
- **Reliable Delivery**: Google's infrastructure ensures high delivery rates (>99%)
- **Cost-Effective**: Free tier up to 500M messages/month; pay-per-use after that (~$0.50 per 1M messages)

### Twilio
- **SMS Reliability**: Highest SMS delivery rates in industry (>99.5%); connected to 1000+ carriers globally
- **WhatsApp Integration**: Seamless WhatsApp Business API integration; customers respond via WhatsApp directly
- **Global Reach**: SMS/WhatsApp available in 180+ countries; local carrier relationships ensure reliability
- **Developer-Friendly**: Well-documented APIs; easy integration with Node.js, Python, Go
- **Programmable Intelligence**: Programmatic SMS routing, failover to alternative carriers
- **Cost**: Competitive SMS pricing (~$0.01-0.05 per SMS depending on volume); WhatsApp pricing varies by country

### SendGrid
- **Email Deliverability**: Specialized email platform with strong reputation management; high inbox placement rates (>95%)
- **Template Support**: Dynamic templating for order confirmations, alerts, and promotional emails
- **Compliance**: GDPR, CAN-SPAM compliance built-in; unsubscribe management automated
- **Analytics**: Bounce, click, open rate tracking; helps monitor delivery quality
- **Cost-Effective**: $14-100/month for small-to-medium volume; first 100 emails/day free
- **Integration**: Easy API integration; webhook support for delivery status updates

## Alternatives Considered

- **AWS Simple Notification Service (SNS)**  
  AWS's managed notification service. However:
  - **Platform Lock-in**: Contradicts GCP platform choice (GKE, BigQuery); multi-cloud complexity without benefit.
  - **Email Capability**: SNS's email delivery has reputation issues; lower inbox placement than SendGrid.
  - **SMS Available**: AWS SNS supports SMS but Twilio's carrier relationships provide better delivery rates.
  - **Missing WhatsApp**: SNS doesn't support WhatsApp; would need separate Twilio integration anyway.

- **AWS SES (Simple Email Service)**  
  Email-only service. However:
  - **Email Only**: Doesn't support SMS or WhatsApp; would need Twilio for additional channels anyway.
  - **Reputation Management**: SES has reputation issues; lower inbox placement than SendGrid (>90% vs. >95%).
  - **Setup Complexity**: Requires more configuration (DKIM, SPF) than SendGrid for optimal delivery.
  - **Cloud Lock-in**: Same multi-cloud issue as SNS.

- **Mailgun**  
  Email delivery platform. However:
  - **Email Only**: No SMS or WhatsApp; would need Twilio for additional channels.
  - **Developer-Friendly**: Good API, but specialized for email only.
  - **Reputation**: Strong deliverability but slightly lower than SendGrid for enterprise.
  - **Smaller Ecosystem**: Fewer integrations with GCP services.

- **OneSignal**  
  Mobile push notification platform. However:
  - **Push Notifications Only**: Doesn't support email or SMS; would need SendGrid + Twilio for complete solution.
  - **More Expensive**: OneSignal pricing (~$1-2 per 1M push notifications) higher than Firebase (free to $0.50 per 1M).
  - **Less Integration**: Lower integration depth with GCP ecosystem.

- **MessageBird**  
  Omnichannel messaging platform. However:
  - **Higher Cost**: SMS pricing (~$0.02-0.10) higher than Twilio; email deliverability not as strong as SendGrid.
  - **Smaller Carrier Network**: Fewer carrier connections than Twilio; slightly lower SMS delivery rates.
  - **Less Mature Ecosystem**: Fewer third-party integrations compared to Twilio.

- **Vonage (Nexmo)**  
  Communications platform. However:
  - **SMS Reliable**: Good SMS delivery but Twilio's carrier network stronger.
  - **Higher Cost**: SMS and voice higher than Twilio.
  - **Email Not Competitive**: No built-in email platform; SendGrid's deliverability better.

- **In-House Notification Service**  
  Building custom notification system:
  - **Complex Infrastructure**: Maintaining email/SMS infrastructure (MTA, carrier integrations, reputation management) is non-trivial.
  - **Compliance Burden**: GDPR, CAN-SPAM, TCPA regulations require expertise.
  - **Carrier Relationships**: Direct SMS delivery requires establishing relationships with carriers; costly and slow.
  - **Operational Risk**: Email reputation issues (blacklisting, bounce management) require constant monitoring.
  - **Not Recommended**: Pre-built services exist; building in-house is poor ROI.

## Why Firebase + Twilio + SendGrid is Better Than Alternatives

| Criterion | Firebase FCM | Twilio SMS | SendGrid Email | OneSignal | MessageBird | In-House |
|-----------|-------------|-----------|----------------|-----------|-----------|----------|
| **Push Notifications** | Excellent | N/A | N/A | Excellent | Fair | Complex |
| **SMS Delivery Rate** | N/A | 99.5%+ | N/A | N/A | ~98% | Difficult |
| **Email Deliverability** | N/A | N/A | >95% inbox | N/A | ~90% inbox | ~80-90% |
| **WhatsApp Support** | No | Yes (Twilio) | No | No | Limited | No |
| **Cost per Message** | $0-$0.50/1M | $0.01-0.05 | $0.1-0.15 | $1-2/1M | $0.02-0.10 | Unpredictable |
| **GCP Integration** | Native | Good (API) | Good (API) | Fair | Fair | Limited |
| **Setup Complexity** | Low | Medium | Low | Medium | Medium | High |
| **Compliance (GDPR/CAN-SPAM)** | Automated | Good (Twilio) | Excellent (SendGrid) | Good | Good | Manual |
| **Reliability (99%+ SLA)** | Yes | Yes | Yes | Yes | Yes | Risky |
| **Developer Experience** | Excellent | Excellent | Excellent | Good | Good | Complex |
| **Operational Burden** | Minimal | Minimal | Minimal | Minimal | Minimal | High |

**Why this combination wins:**

1. **Multi-Channel Coverage**: Firebase for push, Twilio for SMS/WhatsApp, SendGrid for email—each provider is best-in-class for their channel.

2. **Cost Efficiency**: Free Firebase tier, cheap Twilio SMS, affordable SendGrid email = lowest total cost across all channels.

3. **High Delivery Rates**: Firebase's push (Google infrastructure), Twilio's SMS (carrier network), SendGrid's email (reputation management) all optimized for reliability.

4. **GCP Native**: Firebase integrates directly with GKE authentication and Workload Identity; minimal operational overhead.

5. **Compliance Automated**: Each provider handles regulatory compliance (GDPR unsubscribe, CAN-SPAM lists, TCPA consent); team doesn't manage manually.

6. **Real-Time Operations**: WhatsApp via Twilio enables two-way communication with keepers (send alert, keeper responds with actions taken).

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Multi-Provider Complexity** | Three different providers (Firebase, Twilio, SendGrid) mean three integrations, three APIs, three dashboards | Implement Notification Service abstraction layer (publish to internal event bus), handle provider-specific logic internally, use unified logging for all channels |
| **Email Deliverability Issues** | SendGrid reputation drops due to high bounce rates or customer spam complaints; emails land in spam | Monitor bounce/complaint rates via SendGrid dashboards, implement list hygiene (remove bounces promptly), test email templates for spam keywords, maintain domain reputation (DKIM/SPF) |
| **SMS Delivery Failures** | Twilio's delivery might fail in certain regions or for specific carriers | Implement retry logic (queue failed SMSes, retry after 5 minutes), fall back to email if SMS fails, monitor delivery rates by region and carrier, investigate regional issues |
| **Push Notification Opt-Out** | Users disable push notifications; real-time alerts for ride operators not delivered | Provide multiple notification channels (SMS fallback for critical alerts), implement notification preference settings, educate users on importance of push for safety alerts |
| **Cost Overruns** | High notification volume during peak season (holidays, promotions) could exceed budget | Set monthly spend alerts in GCP/Twilio/SendGrid, implement rate limiting (maximum notifications per user per day), use batch notifications instead of individual messages, forecast volume based on visitor capacity |
| **Provider Outage** | SendGrid, Twilio, or Firebase service degradation blocks notifications | Implement graceful degradation (store failed notifications in queue, retry via Pub/Sub), maintain manual alert process via call/SMS, set up provider status page monitoring with alerts |
| **Phone Number Privacy** | Storing phone numbers for SMS requires compliance with data protection regulations | Encrypt phone numbers at rest in Cloud SQL, implement consent tracking (user opt-in for SMS), enable phone number deletion on request, audit access to phone number data |
| **Template Management** | Notification templates scattered across three providers become hard to maintain | Centralize templates in Cloud Storage or firestore, version control template changes, implement template versioning, test templates before deployment |
| **Bounce/Complaint Handling** | Email bounces and complaints must be processed; continued sending to bad addresses damages reputation | Monitor SendGrid webhooks for bounces, implement automatic unsubscribe for hard bounces, investigate soft bounces (retry), maintain bounce list separately from active subscribers |
| **Integration Testing Difficulty** | Testing notifications end-to-end requires test credentials from three providers | Use provider sandbox environments (Firebase test tokens, Twilio test numbers), implement mock notification service for unit tests, maintain test phone/email addresses for integration testing |

## Conclusion

The combination of Firebase Cloud Messaging + Twilio + SendGrid is optimal for multi-channel notifications because:

1. **Best-of-Breed**: Each provider is the industry leader for their channel (Firebase for push, Twilio for SMS/WhatsApp, SendGrid for email).

2. **High Delivery Rates**: Firebase (>99%), Twilio SMS (>99.5%), SendGrid email (>95% inbox) ensure notifications reach users reliably.

3. **Low Operational Burden**: Managed services eliminate need for in-house infrastructure, carrier relationships, reputation management, or compliance expertise.

4. **Cost-Efficient**: Firebase's free tier, Twilio's competitive SMS pricing, SendGrid's affordable email combine for lowest total cost.

5. **GCP Native Integration**: Firebase integrates directly with GKE, Workload Identity, and Cloud Logging; minimal setup.

6. **Real-Time Operations**: WhatsApp via Twilio enables two-way communication with estate staff for immediate action on alerts.

7. **Scalable**: All three providers scale to millions of messages/day without infrastructure changes.

**Recommendation**: 
- Implement Notification Service microservice as abstraction layer over three providers
- Use Firebase Cloud Messaging for in-app push notifications (all estate staff apps)
- Use Twilio for SMS alerts (time-sensitive) and WhatsApp (two-way communication with keepers)
- Use SendGrid for email (confirmations, receipts, promotions)
- Implement retry queue in Pub/Sub for failed notifications
- Monitor delivery rates and set alerts for anomalies
- Test all templates in sandbox before production deployment
- Implement cost monitoring and set monthly spend budgets per provider
