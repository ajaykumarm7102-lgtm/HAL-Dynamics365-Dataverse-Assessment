# Dataverse entity design

## Modeling notes

- Reuse standard **Account**, **Contact** and **Product** tables where suitable.
- Use a custom **Company Membership** junction table because a person may belong to multiple companies and have different permissions in each company. The standard Contact `parentcustomerid` relationship alone does not model this many-to-many business rule.
- Buyer and Seller are company capabilities, not separate company records. A single Account can have both flags enabled.
- Use Dataverse GUIDs as internal primary keys. Use alternate keys for external IDs and integration references to support safe upsert.

## Table inventory

| Table | Type | Important columns | Relationships / notes |
|---|---|---|---|
| Account | Standard | Name, Magento Company ID, Is Buyer, Is Seller, Onboarding Status, KYB Status | Company master in CRM |
| Contact | Standard | Full Name, Email, Mobile, Auth0 Subject | Individual person |
| Company Membership | Custom | Account, Contact, Role, Status, Is Primary | N:1 to Account and Contact; enforce uniqueness for active membership + role as business rules require |
| Company Onboarding | Custom | Account, Application Reference, Status, Submitted On, Completed On | 1:N from Account |
| KYB Verification | Custom | Onboarding, Provider Reference, Result, Verified On, Expires On | Allows multiple checks / re-verification attempts |
| KYB Document | Custom | Onboarding, Document Type, Blob Reference, Verification Status, Uploaded On | Metadata only; binary in secure document storage |
| Onboarding Review | Custom | Onboarding, Reviewer, Decision, Reason, Reviewed On | Append a new review action rather than overwriting previous decisions |
| Onboarding Status History | Custom | Onboarding, From Status, To Status, Changed By, Changed On, Reason | Business audit trail |
| Integration Event | Custom | Event ID, Event Type, Payload/Reference, Schema Version, Status, Retry Count, Correlation ID | Transactional outbox |
| Product | Standard / extended | Magento Product ID, Name, SKU, Category, Catalog Status, Last Synced On | Magento is catalog master |
| Product Sync Log | Custom | Batch ID, External ID, Status, Error, Processed On | Operational sync trace |
| Financing Application | Custom | Account, Applicant Contact, Application Reference, Requested Amount, Status, Submitted On | CRM tracking record |
| Lending Partner | Custom | Name, External Partner ID, Status | Partner configuration |
| Financing Partner Submission | Custom | Financing Application, Lending Partner, Bank Reference, Partner Status | One child per partner submission |
| Financing Offer | Custom | Partner Submission, Offered Amount, Rate, Tenure, Offer Status | Restrict access to sensitive terms |
| Financing Status History | Custom | Financing Application, Old Status, New Status, Source, Changed On | Status audit |

## Important relationship decisions

1. Account to Company Onboarding: 1:N to support re-submission / renewal history if business policy permits.
2. Contact to Account: implemented through Company Membership to support multi-company users.
3. Company Onboarding to KYB Verification: 1:N to preserve each verification attempt.
4. Company Onboarding to KYB Document: 1:N.
5. Company Onboarding to Onboarding Review: 1:N to preserve review history.
6. Financing Application to Financing Partner Submission: 1:N, because the same application may be sent to several banks.
7. Financing Partner Submission to Financing Offer: 1:N to preserve revised offers.
8. Product Category to Product: 1:N.

## Status examples

- Company Onboarding: Draft, Submitted, Under Review, More Information Required, Approved, Rejected, Expired.
- KYB Verification: Pending, In Progress, Passed, Failed, Needs Manual Review, Expired.
- Financing Application: Draft, Submitted, Processing, More Information Required, Approved, Rejected, Withdrawn, Closed.
- Integration Event: Pending, Publishing, Published, Retry, Failed / Dead-lettered.

Use Dataverse Choice columns for controlled statuses. Define explicit transition rules in a Custom API / plug-in or application service; do not rely only on client-side form logic.
