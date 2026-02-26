# Changelog

All notable changes to this Internet-Draft will be documented in this file.

## [draft-00] — 2026-02-26

### Initial Release

- Defined 8 extension members for RFC 9457: `jobId`, `jobStatus`, `submittedAt`, `completedAt`, `retryable`, `retryAfter`, `processingStage`, `correlationId`
- Defined `results` extension member for batch operations
- Defined Job Status Value registry with 7 initial values: ACCEPTED, PROCESSING, COMPLETED, FAILED, CANCELLED, TIMED_OUT, COMPLETED_WITH_ERRORS
- Transport context guidance for HTTP, Kafka, webhooks, and SSE
- Interaction guidance for 12 existing/emerging IETF specifications
- JSON Schema for validation
- Security considerations covering information exposure, correlation ID injection, job ID enumeration, retry amplification, timing side channels, batch scoping, and privacy
- IANA considerations proposing two new registries
- 8 examples spanning HTTP, Kafka, webhook, SSE, batch, and success scenarios
- Appendices for industry comparison, design decisions, AsyncAPI, CloudEvents, OpenAPI, and interoperability

### Review Findings Applied

- Resolved OPTIONAL vs REQUIRED contradiction via Conformance Levels (§3.1.1)
- Added COMPLETED_WITH_ERRORS to Job Status registry (Table 4)
- Softened `retryAfter` constraint from MUST NOT to SHOULD NOT with exceptional-case guidance
- Added UTC requirement for timestamps
- Added `correlationId` to Abstract
- Added privacy considerations (§9.7)
- Added interoperability considerations (Appendix F)
- Added internationalization guidance (Appendix F.5)
- Removed JSON Schema `enum` constraint on `jobStatus` for extensibility
- Used `urn:ietf:params:` for JSON Schema `$id`

### Remaining Before Submission

- [ ] Convert to xml2rfc format
- [ ] Discuss Standards Track vs Informational with HTTPAPI WG
- [ ] Obtain 2+ independent implementations
