# Assumptions and architecture decisions

## Assumptions

1. Dataverse is the authoritative CRM store for Account, Contact, Company Membership, onboarding and KYB decision records.
2. Magento remains authoritative for storefront, orders and product catalog.
3. Profile Service continues to orchestrate registration and onboarding.
4. Existing customer channels must continue to operate during the transition.
5. Azure Service Bus is available for durable asynchronous messaging.
6. Document binaries are held in private document storage / Azure Blob Storage; Dataverse stores metadata and secure references.
7. Partner banks have integration interfaces, but their exact protocols and SLAs are not supplied.
8. Auth0 remains the customer identity provider; Microsoft Entra ID is used for staff and service identities.
9. Admin Backoffice may be an existing portal or a model-driven Power App; both must use controlled authorization.
10. Throughput, availability targets, data residency, retention and regulatory obligations require confirmation.

## Key decisions and rationale

| Decision | Rationale |
|---|---|
| Account and Contact standard tables | Reuse standard Dataverse semantics and platform capabilities |
| Company Membership junction table | Supports users associated with multiple companies and company-specific roles |
| Buyer and Seller as Account capabilities | A company may be both a buyer and seller; avoids duplicate company records |
| Magento owns catalog | Avoid conflicting writes and unclear master-data ownership |
| Service Bus for domain events | Decouples services and supports fan-out, buffering and independent consumers |
| API Management for API entry | Central authentication, throttling, routing, versioning and observability |
| Outbox for Dataverse-originated events | Avoids the dual-write gap between Dataverse commit and message publication |
| Separate partner submission table | One financing application can be submitted to multiple lending partners |
| Blob storage for document binaries | Better fit for large files, lifecycle and controlled download; Dataverse stores metadata |
| Append-only review/status history | Preserves who decided what, when and why |

## Outbox implementation outline

1. A synchronous Dataverse plug-in or supported transaction-bound business operation creates/updates the business record and inserts an Integration Event row in the same Dataverse transaction.
2. A background publisher (Azure Function/worker) queries pending outbox rows using paging/checkpoints.
3. It publishes the event to the appropriate Service Bus topic using managed identity.
4. On successful send, it marks the outbox row Published and records the timestamp/message ID.
5. On transient failure, it increments retry count and schedules retry; after threshold, it marks for operational intervention.
6. A crash after send but before marking Published can cause a duplicate send. This is expected; consumers deduplicate using eventId.

Avoid doing slow external HTTP calls inside a synchronous Dataverse plug-in transaction.

## Resilience and operations

- Idempotent upsert using alternate keys for external company/product/bank references.
- Correlation ID propagated across API, Dataverse, outbox, Service Bus and downstream logs.
- Bounded exponential retry for transient failures; no infinite retry loops.
- Dead-letter queue and operational replay procedure.
- Monitoring for API failures/latency, queue depth, oldest message age, outbox backlog, dead-letter count and sync errors.
- Use Application Insights / Azure Monitor for distributed tracing and alerting.
