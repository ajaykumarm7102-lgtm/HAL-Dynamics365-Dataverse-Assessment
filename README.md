# HAL – Dynamics 365 & Dataverse Solution Design

**Assessment:** Principal / Lead Developer – Microsoft Dynamics 365 & Dataverse  
**Prepared by:** Ajay Kumar  
**Scope:** Target-state data model and integration architecture

## 1. Executive summary

This proposal introduces Microsoft Dataverse as the system of record for HAL's CRM domain: customer contacts, companies, company membership, onboarding, KYB decisions and CRM-facing financing records. Existing customer-facing channels and domain services remain operational. Magento continues to own storefront and catalog operations; partner banks remain authoritative for lending decisions.

The design uses an API-led approach for request/response operations and Azure Service Bus for asynchronous domain events. Azure API Management provides a governed API entry point. Azure Functions (or an equivalent integration worker) handle message publication, transformation and delivery to downstream services.

The goal is to introduce Dataverse without making it a synchronous dependency for every storefront or customer interaction.

## 2. Design principles

1. **Clear ownership:** each business capability has one authoritative system.
2. **Loose coupling:** use events for state changes that do not require an immediate response.
3. **Synchronous only when needed:** use APIs for commands and queries that require an immediate result.
4. **Secure by default:** validate customer identity at the API boundary; use Entra ID service principals for service-to-service access to Dataverse.
5. **Resilient integration:** idempotency, retries, dead-letter handling, correlation IDs and operational monitoring.
6. **Auditable decisions:** preserve KYB review decisions and status history.
7. **Incremental adoption:** existing Magento and microservices continue serving channels during migration.

## 3. System responsibilities / source of truth

| System | Owns / is authoritative for | Dataverse interaction |
|---|---|---|
| Dataverse / CRM API | CRM company and contact profile, company membership, onboarding workflow state, KYB decision record, CRM audit and financing application tracking | Primary CRM data store |
| Profile Service | Registration and company-onboarding orchestration | Calls CRM API / Dataverse-facing integration API |
| Magento | Storefront, commerce transactions and product catalog | Sends catalog copy and receives approved company eligibility/status where required |
| Admin Backoffice | Staff user experience for review and administration | Reads and updates Dataverse through secured APIs or a model-driven app |
| Financing Portal | Customer application experience | Creates and reads financing applications through API |
| Partner banks | Their own loan/account decisions and bank-side application status | Status and offer updates are integrated back |
| Document service / Blob Storage | Document binary content and retention | Dataverse stores metadata, references and verification status |
| Auth0 | Customer identity authentication | Token validated at API edge; identity mapped to Contact / Company Membership |
| Microsoft Entra ID | Internal staff and workload identity | Staff authentication and service-to-service authorization |
| Azure Service Bus | Durable event transport | Carries domain events to subscribers |

## 4. Dataverse data model

The proposed model reuses standard tables where their semantics fit and adds custom tables for HAL-specific processes. See `02-Dataverse-Data-Model/entity-relationships.md` and the Mermaid diagram.

### Core tables

- **Account (standard):** one record per legal/business company. Add external Magento company ID, buyer enabled, seller enabled and onboarding/KYB summary status.
- **Contact (standard):** an individual person/customer user.
- **Company Membership (custom):** junction between Contact and Account. Supports a person belonging to multiple companies and having a different role in each company.
- **Company Onboarding (custom):** one onboarding case/application per company or onboarding attempt.
- **KYB Verification (custom):** verification run/result, provider reference, status, checked date and expiry.
- **KYB Document (custom):** document metadata and secure storage reference; not the document binary itself.
- **Onboarding Review (custom):** staff review actions, decision, reason, reviewer and timestamp.
- **Onboarding Status History (custom):** append-only business status history.
- **Integration Event (custom):** outbox record for reliable event publication and operational tracking.

### Marketplace catalog

- **Product (standard, extended as needed):** synchronized catalog copy, with Magento product ID as an alternate key.
- **Product Category (standard or custom):** category mapping where required.
- **Product Sync Log (custom):** optional operational record for import batches and failures.

Magento remains the catalog master. Dataverse catalog data is a read-oriented copy for CRM visibility, customer support and reporting. Do not attempt uncontrolled two-way updates to catalog fields.

### Financing

- **Financing Application (custom):** company, applicant/contact, requested product/amount, application reference, status and submission date.
- **Lending Partner (custom):** partner bank configuration and external partner identifier.
- **Financing Partner Submission (custom):** junction/child record for each bank submission, external reference and partner-specific status.
- **Financing Offer (custom):** offer terms returned by a partner, subject to access controls.
- **Financing Status History (custom):** status transitions and source.

One financing application may be submitted to multiple lending partners, so model partner submissions separately rather than storing a single bank lookup on the application.

## 5. Integration approach

### Synchronous APIs
Use HTTPS REST APIs through Azure API Management for operations that need an immediate response:
- Profile Service creates or updates a company/contact during registration.
- Financing Portal submits an application and receives an application ID / accepted response.
- Admin Backoffice retrieves a company case and records a review decision.
- A service retrieves a specific CRM profile or application status when required.

The API layer validates the caller, enforces authorization, validates payloads and maps external identifiers to Dataverse records. Use Dataverse Web API and, where business validation must be atomic or encapsulated, a Dataverse Custom API.

### Asynchronous events
Use Azure Service Bus topics for business state changes and fan-out:
- CompanyRegistered
- KYBSubmitted
- KYBApproved
- KYBRejected
- FinancingApplicationSubmitted
- FinancingPartnerStatusUpdated
- ProductCatalogBatchUpdated (or a catalog sync message)

A publisher sends outbox events to Service Bus. Subscribers process events independently. Use separate subscriptions for Magento, notifications, financing integration and analytics as appropriate.

### Reliable publication
For events originating from Dataverse, create an Integration Event / outbox row in the same Dataverse transaction as the business update (for example, from a synchronous plug-in registered on the relevant message). A background publisher reads pending rows, sends each event to Service Bus, then marks it published. If sending fails, retry with backoff; after configured attempts, surface the event for operational replay. Consumers must be idempotent because at-least-once delivery can produce duplicates.

## 6. Security

- Customer: Auth0 authenticates the customer. The API validates the Auth0 JWT (issuer, audience, signature, expiry and scopes/claims), then maps the subject to a Contact and Company Membership. Do not expose Dataverse credentials or grant customers direct broad Dataverse access.
- Staff: Microsoft Entra ID authenticates employees. Use least-privilege Dataverse security roles, teams and business-unit boundaries where appropriate.
- Services: use Entra ID application identities / service principals for Dataverse access; store secrets/certificates in Key Vault and prefer managed identity where supported.
- Service Bus: managed identity and least-privilege sender/receiver roles.
- Documents: private Blob Storage, short-lived access mechanisms, malware scanning and access checks; store only metadata and a secure reference in Dataverse.
- Audit: enable Dataverse auditing for sensitive tables/columns and retain business decision history separately.

## 7. Reliability, operations and non-functional considerations

- Use a unique event ID, correlation ID, causation ID, event type, schema version and occurred-at timestamp.
- Use alternate keys for external IDs (Magento company/product IDs and bank application references) to support safe upsert.
- Apply API timeouts, bounded retries with exponential backoff and circuit-breaker behavior in integration services.
- Use Service Bus dead-letter queues for poison messages; alert and provide a controlled replay process.
- Monitor API latency/error rates, queue depth, oldest-message age, outbox backlog and failed sync records.
- Version event schemas and API contracts. Consumers should tolerate additive fields.
- Minimize personal and financial data in events; send identifiers and necessary business fields rather than full records by default.
- Define retention, data residency and privacy requirements with HAL's compliance/legal stakeholders.

## 8. Assumptions

1. Dataverse is authoritative for CRM company/contact data and the CRM onboarding/KYB decision record.
2. Magento remains authoritative for product catalog and commerce transactions.
3. Profile Service remains the registration/onboarding orchestrator; it is not replaced by Dataverse workflows.
4. Customer-facing channels must remain available during the transition; integrations are introduced incrementally.
5. Azure Service Bus is available as the event backbone.
6. Documents are stored in a dedicated document service or private Azure Blob Storage; Dataverse stores metadata and references.
7. Partner banks expose APIs or agreed file/message interfaces. Exact protocol and response-time behavior will vary by bank.
8. Auth0 remains the customer identity provider; Entra ID is used for staff and service identities.
9. The Admin Backoffice may be implemented as a model-driven Power App or an existing custom portal. In either case, access to Dataverse is secured through supported APIs and identity controls.
10. Exact volumes, SLA, data residency, retention and bank-specific contracts are not supplied; they must be confirmed during discovery.

## 9. Decisions requiring confirmation

- Expected registration and catalog sync volumes and peak rates.
- Whether onboarding can have multiple attempts per company and the required legal-entity uniqueness rules.
- KYB provider(s), verification expiry rules and re-verification triggers.
- Whether any customer-facing operation requires a real-time KYB decision.
- Bank API protocols, idempotency support and callback/webhook availability.
- Document retention, UAE data residency and regulatory requirements.
- Whether Admin Backoffice is an existing application or can be delivered as a model-driven app.

## 10. Suggested implementation phases

1. Establish Dataverse environment, security model, core Account/Contact/Membership tables and API integration.
2. Implement onboarding, KYB, document metadata, staff review and audit history.
3. Add outbox publisher, Service Bus topics, consumer idempotency and monitoring.
4. Integrate Magento catalog copy and approved-company status synchronization.
5. Integrate financing application and partner-specific submission/status handling.
6. Load test, security test, operationalize replay/monitoring and cut over incrementally.

## 11. Repository contents

- `01-Architecture/architecture-diagrams.md` – rendered Mermaid context and sequence diagrams.
- `02-Dataverse-Data-Model/entity-relationship-diagram.md` – rendered Mermaid ER diagram; `entity-details.md` – table inventory and relationship notes.
- `03-Integration-Design/` – interaction matrix and event catalog.
- `04-Technical-Decisions/` – security, resilience and assumptions.

The diagrams use Mermaid so they remain editable as source in GitHub.
