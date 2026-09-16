# ADR-011 — Choice of Google Cloud Identity and Firebase Authentication for Identity & Authentication

## Context

The estate system requires identity and authentication for multiple user types:

**User Categories:**
- Visitors: One-time or repeat visitors booking tickets via web/mobile app
- Staff: Ground staff (ride operators, animal keepers, queue management) and office staff
- Administrators: Estate management, system admins with elevated privileges
- External: Payment gateway, third-party integrations

**Authentication Requirements:**
- Visitor login (optional for guest checkout; optional for loyalty login)
- Staff login with multi-factor authentication (MFA)
- Admin login with privileged access controls
- Session management (tokens, expiration, revocation)
- Single Sign-On (SSO) for staff using corporate credentials
- OAuth2 / OpenID Connect support for third-party integrations
- PCI-DSS compliance (no plaintext passwords, secure password storage)

**Authorization Requirements:**
- Role-based access control (RBAC): visitor, staff, manager, admin
- Resource-level access: keepers see only their animals, ride ops see only their rides
- Time-based access: staff access restricted to shift hours
- Data access: read-only vs. read-write permissions

**Scale Requirements:**
- 5,000-15,000 visitors/day, each potentially creating account (100-300 registrations/day)
- 100-200 staff members with concurrent sessions
- Peak concurrent sessions: 1,000+ visitors + 50 staff
- Session timeout: 30 minutes for staff (security), 24 hours for visitors
- Password reset: 10-50 daily requests

**Integration Requirements:**
- Auth Service microservice in GKE validates tokens
- API Gateway routes authenticated requests
- Sessions stored in Redis (JWT tokens or session IDs)
- Audit logging of login attempts and privilege changes
- Integration with corporate SSO (if staff uses company Google/Microsoft accounts)

## Decision

**We will use Google Cloud Identity (for enterprise identity management) and Firebase Authentication (for user-facing authentication) integrated together.**

This combination provides:
- Google Cloud Identity for enterprise staff SSO and MFA
- Firebase Authentication for visitor accounts with social login options
- Unified authentication service without building custom auth system
- Seamless GCP IAM integration for staff
- PCI-DSS compliant (no password storage, secure token handling)
- Minimal operational burden

## Key Differentiators

### Google Cloud Identity
- **Enterprise SSO**: Integrates with corporate Google Workspace, Active Directory (via Cloud Identity Sync), and other enterprise identity providers
- **Multi-Factor Authentication**: Enforced MFA for staff accounts; SMS, TOTP, Security Keys supported
- **GCP IAM Integration**: Staff authentication flows directly into Cloud IAM permissions; authorized staff can access GCP resources (logs, dashboards, data)
- **Device Management**: Control which devices can access systems (mobile device compliance policies)
- **Zero-Trust Security**: Continuous device assessment; detect compromised devices and revoke access

### Firebase Authentication
- **Simple User Registration**: Visitors create accounts easily with email/password or social login (Google, Facebook)
- **Email Verification**: Automatic email confirmation flow prevents fake accounts
- **Password Reset**: Self-service password reset; no support ticket needed
- **Social Login**: Google, Facebook, GitHub logins reduce password fatigue
- **Passwordless Authentication**: Magic link authentication (email-based login) for improved security
- **Phone Authentication**: SMS-based login option for mobile visitors
- **Anonymous Authentication**: Guests can use system without creating account; convert to full account later

### Combined Approach Benefits
- Staff (100-200 users) authenticate via Cloud Identity + GCP IAM
- Visitors (5,000-15,000 users) authenticate via Firebase Authentication
- Unified JWT-based tokens; Auth Service validates both types
- Simplified user management: Cloud Identity for enterprise, Firebase for public
- Reduced operational burden: Both managed services (no password database, no salt/hash implementation)

## Alternatives Considered

- **Auth0**  
  Third-party authentication platform. However:
  - **Higher Cost**: Auth0 per-active-user pricing (~$30-100/month depending on volume). For 5,000+ users = $150-500/month.
  - **External Dependency**: Auth0 outages block login. No direct control over service.
  - **Vendor Lock-in**: Auth0-specific configuration; switching providers difficult.
  - **Not Necessary**: Google provides equivalent functionality at lower cost.

- **Okta**  
  Enterprise identity platform. However:
  - **Very High Cost**: Okta per-user pricing $1-3/month; 5,000+ users = $5,000-15,000/month. Extremely expensive for estate system.
  - **Enterprise-Focused**: Okta designed for large enterprises (10,000+ employees). Overkill for this scale.
  - **Complex Setup**: Okta deployments require months of configuration. Not suitable for MVP timeline.

- **Keycloak**  
  Open-source identity platform. However:
  - **Self-Hosted Complexity**: Requires Keycloak deployment, database setup, SSL certificates, and maintenance.
  - **Infrastructure Burden**: Keycloak cluster management, scaling, and security patching required.
  - **Development Effort**: Keycloak integration more complex than Firebase; requires more custom code.
  - **Smaller Ecosystem**: Fewer pre-built integrations (payment providers, notifications) compared to Firebase.

- **Amazon Cognito (Comparison to AWS)**  
  AWS's authentication service. However:
  - **Cloud Lock-in**: Selecting AWS Cognito contradicts GCP platform choice (GKE, BigQuery). Multi-cloud complexity without benefit.
  - **Cross-Cloud Integration**: Cognito (AWS) with GKE (GCP) requires bridge configuration; adds latency and complexity.
  - **Different Ecosystem**: Different authentication flows, different token formats; harder to integrate across cloud providers.

- **Custom Auth System**  
  Building authentication in-house using bcrypt/scrypt:
  - **Security Risk**: Implementing authentication incorrectly introduces vulnerabilities (weak password hashing, timing attacks, token leakage).
  - **Compliance Burden**: PCI-DSS requires specific password policies, encryption, and audit logging. Implementing correctly requires security expertise.
  - **Maintenance Burden**: Managing password resets, MFA, account recovery, and breach response requires dedicated resources.
  - **Not Recommended**: Pre-built services exist; building in-house poor security ROI.

- **Azure AD**  
  Microsoft's identity platform. However:
  - **Platform Lock-in**: Microsoft ecosystem (Azure, Office 365) not aligned with GCP choice.
  - **Cross-Cloud Complexity**: Azure AD (Microsoft) to GKE (GCP) integration complex.
  - **Cost**: Azure AD similar cost to Okta for enterprise scenarios.

- **OAuth2/OIDC-Only (No Provider)**  
  Building auth using open standards only:
  - **Still Requires Provider**: OAuth2/OIDC are protocols, not implementations. Still need a provider to host auth (Google, Auth0, Okta, custom).
  - **Not an Alternative**: This is what Cloud Identity and Firebase provide.

## Why Firebase + Cloud Identity Are Better Than Alternatives

| Criterion | Firebase + Cloud Identity | Auth0 | Okta | Keycloak | Custom |
|-----------|--------------------------|-------|------|----------|--------|
| **Visitor Registration** | Excellent (Firebase) | Excellent | Good | Good | Complex |
| **Enterprise SSO** | Excellent (Cloud Identity) | Excellent | Excellent | Good | Complex |
| **MFA Support** | Excellent (both) | Excellent | Excellent | Good | Risky |
| **Social Login** | Excellent (Firebase) | Excellent | Good | Limited | Risky |
| **Cost (5K users)** | $50-100/mo | $150-500/mo | $5,000+/mo | $0 (self-hosted) | Unpredictable |
| **Setup Time** | Fast (days) | Medium (weeks) | Slow (months) | Medium (weeks) | Slow (months) |
| **Operational Burden** | Minimal (managed) | Minimal | Minimal | High (self-hosted) | Very High |
| **PCI Compliance** | Automatic | Automatic | Automatic | Manual | Risky |
| **GCP Integration** | Native | Limited | Limited | Limited | Limited |
| **Uptime SLA** | 99.99% | 99.99% | 99.99% | Self-dependent | Self-dependent |
| **Security Risk** | Low | Low | Low | Medium | High |
| **Scalability** | Unlimited (managed) | Auto-scaling | Licensed | Manual scaling | Unknown |

**Why Firebase + Cloud Identity win:**

1. **Cost-Efficient**: Firebase free tier covers 10K+ users; Cloud Identity scales affordably for staff. Auth0 and Okta prohibitively expensive.

2. **Unified GCP Integration**: Cloud Identity staff authentication flows directly to GCP IAM. Staff can access logs, dashboards, data with same credentials. No credential management.

3. **Visitor Experience**: Firebase's social login and passwordless authentication improve visitor signup experience. Higher conversion, fewer password resets.

4. **Enterprise Features**: Cloud Identity MFA, device management, and SSO satisfy enterprise security requirements for staff.

5. **Zero Operational Burden**: Fully managed services (no auth database, no password hashing, no infrastructure). Team focuses on business logic.

6. **PCI-DSS Compliance**: Google manages secure password storage, encryption, and compliance. No custom implementation risk.

7. **Seamless Integration**: Firebase SDKs for web/mobile; Cloud Identity integrates with GCP IAM. Minimal code needed.

## Risks & Trade-offs

| Risk Area | Description | Mitigation |
|-----------|-------------|-----------|
| **Google Account Dependency** | Staff authentication depends on Google/Corporate identity provider; provider outage blocks staff login | Test failover to secondary authentication (backup password), implement offline mode for critical operations, maintain contact with identity provider |
| **Firebase Token Expiration** | Firebase tokens expire (1 hour default); expired tokens must refresh; refresh failure logs users out | Implement token refresh in Auth Service transparently, use refresh tokens to extend sessions, educate team on token lifecycle |
| **Email Verification Spam** | Firebase sends email verification emails; high volume could trigger spam filters | Configure sending domain (DKIM, SPF), send verification emails from branded address, test email delivery in staging |
| **Social Login Account Linking** | Users might create account with email, then later try to login via Google (different accounts). Confusion and duplicate accounts possible | Implement account linking (user can link Google account to existing email account), detect duplicate accounts during signup |
| **MFA Bypass Risk** | MFA disabled by careless staff or lost authenticators; MFA not enforced uniformly | Enforce MFA via Cloud Identity policies (non-optional), provide recovery codes, test MFA enforcement before production |
| **JWT Token Validation** | Auth Service must validate Firebase and Cloud Identity tokens; invalid signature/expiration should reject request | Implement proper token validation library, test expired/invalid tokens, cache token validation results for performance |
| **Session Hijacking** | JWT tokens in URLs or logs could be intercepted; tokens must be protected | Use HTTPS only (no HTTP), store tokens in secure cookies (httpOnly, Secure flags), implement token rotation |
| **Rate Limiting Auth Requests** | Brute-force attacks could attempt thousands of login attempts; Firebase might rate-limit legitimate users | Implement client-side rate limiting (prevent multiple attempts), use CAPTCHA for suspicious login patterns, monitor failed login rate |
| **Staff Credential Compromise** | Staff password compromise or stolen device could leak access; no way to revoke single credential | Implement device management (revoke access from compromised device), enforce re-authentication for sensitive operations, audit staff login history |
| **Multi-Organization Support** | If estate expands to multiple locations, separate authentication/authorization per location challenging | Design RBAC with location/region attributes, use Cloud Identity hierarchies, separate Firebase projects per location if needed |

## Conclusion

Firebase Authentication + Google Cloud Identity are the optimal identity platforms for the estate system because:

1. **Cost-Efficient**: Firebase free tier + Cloud Identity fixed pricing much cheaper than Auth0 ($150+/mo) or Okta ($5,000+/mo) at this scale.

2. **Dual-Mode Authentication**: Firebase for public visitors (simple, passwordless, social), Cloud Identity for enterprise staff (MFA, SSO, device management). Best of both worlds.

3. **GCP-Native Integration**: Staff Cloud Identity accounts automatically integrate with GCP IAM. Staff access logs, dashboards, data with same credentials. No separate user management.

4. **PCI-DSS Compliant**: Google manages secure password storage, encryption, and compliance. No custom implementation risk or compliance burden.

5. **Zero Operational Burden**: Fully managed services. No auth database, no password hashing, no infrastructure. Team focuses on app features.

6. **Scalable & Reliable**: Both services 99.99% uptime SLA; auto-scale to millions of users. Handle peak traffic (15,000 daily visitors).

7. **Developer-Friendly**: Firebase SDKs for web/mobile; Cloud Identity integrates via GCP APIs. Minimal code for authentication.

8. **Security Integrated**: Passwordless login, social login, MFA, device management all built-in. No security corners cut to save cost.

**Recommendation**:
- Use Firebase Authentication for visitor accounts (email/password + Google/Facebook social login + passwordless magic links)
- Use Google Cloud Identity for staff accounts (corporate Google Workspace + Active Directory sync if needed)
- Implement Auth Service microservice that validates both Firebase and Cloud Identity tokens
- Store visitor/staff accounts in Cloud SQL (user_id, email, roles, last_login)
- Use Redis for session management (store JWT refresh tokens)
- Implement MFA enforcement via Cloud Identity policies
- Enable Cloud Audit Logging for all authentication events (login, logout, password change)
- Test failover scenarios (provider outage, token validation failure)
- Educate staff on password hygiene and MFA importance
- Implement token rotation and refresh strategies
- Monitor failed login attempts and block suspicious patterns via Cloud Monitoring alerts
