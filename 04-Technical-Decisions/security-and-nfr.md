# Security, audit and non-functional design

## Identity and access

### Customers
- Auth0 authenticates customers using OIDC/OAuth 2.0.
- API Management / backend validates JWT signature, issuer, audience, expiry and scopes/claims.
- Backend maps Auth0 subject to Contact and checks Company Membership and role before company-level actions.
- Do not expose Dataverse service credentials to browsers or mobile clients.

### Internal staff
- Microsoft Entra ID authenticates staff.
- Admin Backoffice uses least-privilege Dataverse security roles / teams.
- Separate duties for KYB reviewer, approver and administrator where business policy requires it.
- Restrict sensitive financing and document fields to authorized roles.

### Service-to-service
- Use Entra ID application identities / service principals for Dataverse access.
- Prefer managed identity for Azure-hosted workers where supported.
- Store third-party credentials and certificates in Azure Key Vault.
- Use managed identity and least-privilege RBAC for Service Bus.

## Data protection
- TLS for all network communication.
- Encrypt storage at rest using platform encryption and approved key management.
- Store documents in private Blob Storage; use short-lived scoped access and malware scanning.
- Avoid placing full documents, national IDs or unnecessary personal/financial data in Service Bus messages and logs.
- Apply data retention, deletion and residency policies after compliance review.

## Audit
- Enable Dataverse auditing for critical tables and sensitive columns.
- Keep Onboarding Review and Status History records as business audit evidence.
- Capture reviewer, decision, reason, timestamp and source system.
- Restrict deletion and modification of historical decision records.

## Availability and performance
- Keep customer-facing registration and storefront paths independent from downstream notification and analytics consumers.
- Use queue buffering for spikes and bulk catalog synchronization.
- Apply API timeouts, bounded retries and circuit breakers in integration services.
- Use paging, batching and alternate-key upserts for Dataverse data loads.
- Load test against agreed volumes and Dataverse service protection limits before production rollout.

## Operational monitoring
- Correlation ID across all components.
- Application Insights / Azure Monitor distributed tracing.
- Dashboards and alerts for API availability, latency, error rates, Service Bus queue depth, dead-letter messages, outbox backlog and synchronization lag.
- Document replay/reprocessing runbooks and assign operational ownership.
