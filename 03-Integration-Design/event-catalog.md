# Domain event catalog

Events describe facts that have already happened. They are not commands to force another system to perform an action.

| Event | Producer | Consumers | When published / key data |
|---|---|---|---|
| CompanyRegistered | CRM / Profile integration | Magento, Notifications | Company and external IDs, correlation ID, occurred-at |
| KYBSubmitted | CRM | Admin Backoffice notifications, KYB provider integration | Onboarding ID, company ID, document references |
| KYBApproved | CRM | Magento, Profile Service, Notifications, Financing eligibility | Company ID, onboarding ID, decision time |
| KYBRejected | CRM | Magento, Profile Service, Notifications | Company ID, onboarding ID, rejection reason code (avoid unnecessary sensitive detail) |
| KYBMoreInformationRequired | CRM | Profile Service, Notifications | Company/onboarding IDs, requested action |
| ProductCatalogBatchUpdated | Catalog integration | CRM reporting / dependent services | Batch ID, item count, source timestamp |
| FinancingApplicationSubmitted | CRM / Financing integration | Partner submission worker | Application ID, company ID, requested product and amount as permitted |
| FinancingPartnerStatusUpdated | Bank integration | CRM, Financing Portal, Notifications | Application ID, partner ID, partner status, external reference |
| FinancingOfferReceived | Bank integration | CRM, Financing Portal | Application/partner submission IDs, offer reference and restricted offer details |

## Event envelope

```json
{
  "eventId": "unique-guid-or-uuid",
  "eventType": "KYBApproved",
  "schemaVersion": "1.0",
  "occurredAt": "2026-11-20T12:00:00Z",
  "correlationId": "request-or-workflow-correlation-id",
  "source": "HAL.CRM",
  "data": {
    "companyId": "dataverse-account-guid",
    "onboardingId": "dataverse-onboarding-guid"
  }
}
```

Use UTC timestamps. Include only the data required by subscribers; prefer record IDs and references over copying full personal or financial records into messages.

## Delivery guarantees

- Assume at-least-once delivery; consumers must be idempotent.
- Use `eventId` as the deduplication key.
- Use a schema version and additive-compatible changes.
- Retry transient failures with backoff.
- Dead-letter messages after configured attempts and alert support.
- Provide controlled replay after correcting the underlying issue.
