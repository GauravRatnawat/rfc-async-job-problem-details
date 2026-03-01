# draft-ratnawat-httpapi-async-problem-details

**Problem Details for Asynchronous Job Failures**

An Internet-Draft extending [RFC 9457 (Problem Details for HTTP APIs)](https://www.rfc-editor.org/rfc/rfc9457) with reusable extension members for asynchronous job processing.

---

## Overview

HTTP APIs that process work asynchronously need a standard way to report job failures. RFC 9457 provides the error envelope; this document defines extension members that fill it with asynchronous-job-specific context.

Eight core extension members are specified, plus a ninth (`results`) for batch operations:

| Member | Type | Description |
|---|---|---|
| `jobId` | string | Unique identifier for the async job |
| `jobStatus` | string | Current state (ACCEPTED, PROCESSING, COMPLETED, FAILED, CANCELLED, TIMED_OUT, COMPLETED_WITH_ERRORS) |
| `submittedAt` | string (date-time) | RFC 3339 timestamp (UTC) when the job was accepted |
| `completedAt` | string (date-time) | RFC 3339 timestamp (UTC) when the job reached a terminal state |
| `retryable` | boolean | Whether the client should retry the job |
| `retryAfter` | integer | Seconds to wait before retrying |
| `processingStage` | string | Pipeline stage where failure occurred |
| `correlationId` | string | Client-supplied trace identifier |
| `results` | array | Per-item outcomes for batch operations |

## The Problem

Every major platform invents its own async job error format:

```
AWS Step Functions:  { "executionArn": "...", "status": "FAILED", "error": "...", "cause": "..." }
Google AIP-151:     { "name": "...", "done": true, "error": { "code": 3, "message": "..." } }
Azure LRO:          { "id": "...", "status": "Failed", "error": { "code": "...", "message": "..." } }
Stripe:             { "id": "...", "status": "failed", "last_payment_error": { ... } }
```

None builds on RFC 9457. None provides transport-independent retry semantics. None includes processing stage identification.

## Key Differentiators

1. **Transport Independence** — The `retryable` + `retryAfter` JSON members work in HTTP responses, Kafka messages, webhooks, and SSE streams (where the `Retry-After` HTTP header is unavailable)
2. **Processing Stage Identification** — The `processingStage` member tells you *where* in the pipeline the failure occurred
3. **Built on RFC 9457** — Extends the IETF standard error envelope rather than inventing a new one

## Quick Example

```json
{
  "type": "https://api.example.com/problems/rendering-failed",
  "title": "Document Rendering Failed",
  "status": 500,
  "detail": "Template 'invoice-v2' contains an unclosed element at line 87",
  "instance": "/api/v1/documents/jobs/550e8400-e29b-41d4-a716-446655440000",
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "jobStatus": "FAILED",
  "submittedAt": "2026-02-26T10:00:00Z",
  "completedAt": "2026-02-26T10:00:03Z",
  "retryable": false,
  "processingStage": "rendering",
  "correlationId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

## Repository Structure

```
.
├── README.md                          # This file
├── CONTRIBUTING.md                    # How to contribute
├── LICENSE                            # BSD-3-Clause (IETF Trust)
├── draft/
│   ├── draft-ratnawat-httpapi-async-problem-details-00.md    # The Internet-Draft (source)
│   ├── draft-ratnawat-httpapi-async-problem-details-00.xml   # xml2rfc format
│   ├── draft-ratnawat-httpapi-async-problem-details-00.html  # HTML rendering
│   └── draft-ratnawat-httpapi-async-problem-details-00.txt   # Plain text rendering
├── schema/
│   └── async-job-problem-details.schema.json                 # JSON Schema
└── examples/
    ├── http-rendering-failure.json    # HTTP response example
    ├── http-timeout-with-retry.json   # Timeout + retry example
    ├── kafka-conversion-failure.json  # Kafka transport example
    ├── kafka-transient-failure.json   # Kafka retry guidance example
    ├── webhook-batch-partial.json     # Webhook batch example
    ├── sse-job-failed.txt             # Server-Sent Events example
    ├── batch-partial-failure.json     # Batch partial failure
    └── successful-completion.json     # Success example
```

## Status

| Item | Status |
|---|---|
| **Draft version** | -00 |
| **Intended status** | Standards Track (may adjust to Informational per WG feedback) |
| **Target WG** | [IETF HTTPAPI](https://datatracker.ietf.org/wg/httpapi/about/) |
| **Datatracker** | Not yet submitted |
| **xml2rfc conversion** | Complete |

## Roadmap

- [x] Write the Internet-Draft (-00)
- [x] Landscape analysis — verify the gap is real
- [x] Simulated IETF review — identify and fix issues
- [x] Adoption playbook — strategy for real-world adoption
- [x] Convert to xml2rfc format for IETF submission
- [ ] Build reference implementation (Quarkus / Spring Boot)
- [ ] Submit PR to Zalando RESTful API Guidelines
- [ ] Open issue on Microsoft REST API Guidelines
- [ ] Submit to IETF Datatracker
- [ ] Present at IETF HTTPAPI WG meeting
- [ ] Seek co-authors from other organizations

## Related Standards

| Standard | Relationship |
|---|---|
| [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) | Foundation — this draft extends it |
| [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) | HTTP Semantics (202 Accepted) |
| [RFC 7240](https://www.rfc-editor.org/rfc/rfc7240) | Prefer Header (respond-async) |
| [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) | Web Linking |
| [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562) | UUIDs for jobId |
| [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) | Timestamps |
| [CloudEvents](https://cloudevents.io/) | Complementary — defines event envelope |
| [AsyncAPI](https://www.asyncapi.com/) | Complementary — documentation tool |

## Author

**Gaurav Ratnawat**
IMTF
gaurav.ratnawat@imtf.com

## License

This repository contains an Internet-Draft subject to the provisions of [BCP 78](https://www.rfc-editor.org/info/bcp78) and the [IETF Trust's Legal Provisions](https://trustee.ietf.org/license-info).
