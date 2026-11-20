# Integration matrix

| # | Source → Target | Purpose / data | Protocol | Pattern | Security |
|---|---|---|---|---|---|
| 1 | Customer → Magento | Registration and storefront activity | HTTPS | Synchronous user interaction | Auth0 |
| 2 | Magento → Profile Service | Registration details, external customer/company IDs | REST/HTTPS | Synchronous | OAuth 2.0 / service credential |
| 3 | Profile Service → API Management → CRM API / Dataverse | Create/update Account, Contact, Company Membership and onboarding | REST + Dataverse Web API / Custom API | Synchronous command | OAuth 2.0; Entra service principal at Dataverse boundary |
| 4 | Admin Backoffice → Dataverse | Read case, record decision, reason and reviewer | Dataverse Web API / Custom API | Synchronous | Entra ID; staff security roles |
| 5 | Dataverse outbox → Service Bus | CompanyRegistered, KYBSubmitted, KYBApproved, KYBRejected | Service Bus AMQP 1.0 | Asynchronous event | Managed identity |
| 6 | Service Bus → Magento integration worker | Approved/rejected company status and eligibility | AMQP in; REST/HTTPS out | Asynchronous consumer | Managed identity + Magento API credential in Key Vault |
| 7 | Magento → catalog integration worker → Dataverse | Product, SKU, category, price/catalog state, external product ID | REST/HTTPS and Dataverse Web API; optionally Service Bus for bulk sync trigger | Asynchronous batch/event-driven | Entra service principal |
| 8 | Financing Portal → API Management → Dataverse | Financing application and applicant/company references | REST/HTTPS | Synchronous command; returns application reference | Auth0 customer token validated at API edge; authorization against company membership |
| 9 | Financing integration worker ↔ Partner Banks | Application submission, partner reference, status, offer | Bank-specific REST/HTTPS, webhook or agreed file interface | Asynchronous preferred; synchronous only if bank contract requires | Mutual TLS / OAuth / signed webhook as agreed |
| 10 | Dataverse / event worker → Notification Service | Template ID, recipient reference, locale, event data | Service Bus | Asynchronous | Managed identity |
| 11 | Customer / staff → Identity providers | Customer and employee authentication | OIDC / OAuth 2.0 | Token-based | Auth0 for customers; Entra ID for staff |
| 12 | CRM / integration worker ↔ Document Service | Upload/download authorization, metadata, verification result | HTTPS REST; private Blob Storage access | Synchronous metadata request; upload may use pre-signed/short-lived access | Scoped access; malware scan |

## Notes

- The exact bank interface is not specified. The design deliberately allows REST APIs, webhooks or a partner-agreed file exchange.
- For customer APIs, Auth0 authenticates the end user. The backend validates the JWT and maps the subject to a Contact and Company Membership. Do not pass a customer token directly to Dataverse unless a separately designed and supported identity model explicitly requires it.
- For bulk catalog synchronization, use a batch-oriented process with paging, checkpoints and upsert by Magento Product ID. Do not send one synchronous API call per product during a large catalog import.
- A request that needs immediate validation/confirmation is synchronous. Notifications, status propagation, catalog batch processing and partner callbacks are asynchronous.
