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

Job failure fields vary by platform. Draft adds shared job context and retry hints across transports.
See [problem statement](draft/draft-ratnawat-httpapi-async-problem-details-00.html#problem-statement).
See [platform comparison](draft/draft-ratnawat-httpapi-async-problem-details-00.html#appendix-comparison).

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

[Draft source](draft/draft-ratnawat-httpapi-async-problem-details-00.md), [XML](draft/draft-ratnawat-httpapi-async-problem-details-00.xml), [HTML](draft/draft-ratnawat-httpapi-async-problem-details-00.html), [plain text](draft/draft-ratnawat-httpapi-async-problem-details-00.txt).
[Schema](schema/async-job-problem-details.schema.json).
[Examples](examples/).
[How to help](CONTRIBUTING.md).

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

See [how draft works with other standards](draft/draft-ratnawat-httpapi-async-problem-details-00.html#interaction).

## Author

**Gaurav Ratnawat**
IMTF
gaurav.ratnawat@imtf.com

## License

This repository contains an Internet-Draft subject to the provisions of [BCP 78](https://www.rfc-editor.org/info/bcp78) and the [IETF Trust's Legal Provisions](https://trustee.ietf.org/license-info).
