# RFC Analysis — SironOne Doc Generator

**Date**: 2026-02-26
**Author**: Gaurav Ratnawat
**Status**: Living Document
**Project**: SironOne Doc Generator (SOC)

---

## Table of Contents

1. [Overview](#overview)
2. [RFCs Currently Implemented](#rfcs-currently-implemented)
3. [RFCs to Implement (Gaps & Improvements)](#rfcs-to-implement-gaps--improvements)
4. [Priority Matrix](#priority-matrix)
5. [Proposed New RFC: Async Job Problem Details](#proposed-new-rfc-async-job-problem-details)

---

## Overview

This document maps the SironOne Doc Generator's features to relevant IETF RFCs, identifies compliance gaps, recommends improvements, and proposes a new RFC that the team could submit to the IETF based on real-world patterns in this project.

The service sits at the intersection of **REST APIs, event-driven (Kafka) architecture, PDF document generation, and OAuth2/OIDC security** — all areas with rich RFC coverage.

---

## RFCs Currently Implemented

These are RFCs that the project already implements, either directly or via framework support (Quarkus OIDC, SmallRye, etc.).

### RFC 9457 — Problem Details for HTTP APIs

| | |
|---|---|
| **RFC** | [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (supersedes RFC 7807) |
| **Status** | ✅ Implemented |
| **Where** | `GlobalExceptionHandlers.kt` |
| **Details** | Auth errors (401/403), validation errors (400), and internal errors (500) return `application/problem+json` responses with `type`, `title`, `status`, and `detail` fields. Custom problem type URIs defined in `ProblemType` object (e.g., `/problems/validation-error`, `/problems/pdf-conversion-failed`). |

**Example response produced by the service:**
```json
{
  "type": "about:blank",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Authentication required"
}
```

---

### RFC 6749 — OAuth 2.0 Authorization Framework

| | |
|---|---|
| **RFC** | [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) |
| **Status** | ✅ Implemented |
| **Where** | Keycloak integration, `application.properties`, `JwtAudienceValidator.kt` |
| **Details** | The service uses the **client credentials grant** (`grant_type=client_credentials`) for service-to-service (S2S) authentication. The calling service (`acm`) authenticates with Keycloak to obtain a JWT, which is then presented to the doc generator. ROPC grant is explicitly disabled. |

---

### RFC 7519 — JSON Web Token (JWT)

| | |
|---|---|
| **RFC** | [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519) |
| **Status** | ✅ Implemented |
| **Where** | Quarkus OIDC validation, `JwtAudienceValidator.kt` |
| **Details** | Full JWT validation chain: signature verification (via JWKS), expiry (`exp`), issuer (`iss`), audience (`aud`), and custom claims (`resource_access/doc-service/roles`). Custom `SecurityIdentityAugmentor` provides defense-in-depth audience validation with detailed logging. |

---

### RFC 7517 — JSON Web Key (JWK)

| | |
|---|---|
| **RFC** | [RFC 7517](https://www.rfc-editor.org/rfc/rfc7517) |
| **Status** | ✅ Implemented (via framework) |
| **Where** | Quarkus OIDC auto-discovery |
| **Details** | Quarkus automatically downloads Keycloak's public keys via the JWKS endpoint (`/protocol/openid-connect/certs`) for JWT signature validation. No custom code required. |

---

### RFC 6750 — Bearer Token Usage

| | |
|---|---|
| **RFC** | [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750) |
| **Status** | ✅ Implemented |
| **Where** | `PdfGenerationResource.kt`, path security policies |
| **Details** | All `/api/*` endpoints require `Authorization: Bearer <token>` header. Missing or invalid tokens return 401. Tokens with insufficient roles return 403. |

---

### RFC 9110 — HTTP Semantics (supersedes RFC 7231)

| | |
|---|---|
| **RFC** | [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) |
| **Status** | ✅ Partially implemented |
| **Where** | `PdfGenerationResource.kt`, `GlobalExceptionHandlers.kt` |
| **Details** | Proper use of HTTP methods (POST for generation), status codes (200, 400, 401, 403, 500), and `Content-Type` headers. See [gaps](#1-rfc-9110--etag-and-conditional-requests) for missing features. |

---

### RFC 6266 — Content-Disposition in HTTP

| | |
|---|---|
| **RFC** | [RFC 6266](https://www.rfc-editor.org/rfc/rfc6266) |
| **Status** | ✅ Implemented |
| **Where** | `PdfGenerationResource.kt` |
| **Details** | PDF responses include `Content-Disposition: attachment; filename="document-{templateId}-{timestamp}.pdf"` to trigger browser downloads with meaningful filenames. |

---

### RFC 7234 / RFC 9111 — HTTP Caching

| | |
|---|---|
| **RFC** | [RFC 9111](https://www.rfc-editor.org/rfc/rfc9111) (supersedes RFC 7234) |
| **Status** | ✅ Implemented |
| **Where** | `PdfGenerationResource.kt` |
| **Details** | PDF responses include `Cache-Control: no-cache, no-store, must-revalidate` to prevent caching of generated documents. Appropriate since each PDF is generated dynamically. |

---

### RFC 8259 — The JSON Data Interchange Format

| | |
|---|---|
| **RFC** | [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) |
| **Status** | ✅ Implemented (via framework) |
| **Where** | All JSON request/response bodies, Kafka message serialization |
| **Details** | Jackson handles JSON serialization/deserialization. Request bodies (`GeneratePdfRequest`), error responses (`ProblemDetail`), and Kafka event payloads all use JSON. |

---

### Summary of Implemented RFCs

| RFC | Title | Compliance |
|-----|-------|------------|
| RFC 9457 | Problem Details for HTTP APIs | ✅ Full (with minor gaps — see below) |
| RFC 6749 | OAuth 2.0 Authorization Framework | ✅ Full |
| RFC 7519 | JSON Web Token (JWT) | ✅ Full |
| RFC 7517 | JSON Web Key (JWK) | ✅ Full (framework) |
| RFC 6750 | Bearer Token Usage | ✅ Full |
| RFC 9110 | HTTP Semantics | ⚠️ Partial (missing ETag) |
| RFC 6266 | Content-Disposition | ✅ Full |
| RFC 9111 | HTTP Caching | ✅ Full |
| RFC 8259 | JSON Data Interchange | ✅ Full (framework) |

---

## RFCs to Implement (Gaps & Improvements)

### 1. RFC 9110 — ETag and Conditional Requests

| | |
|---|---|
| **RFC** | [RFC 9110 §8.8.3](https://www.rfc-editor.org/rfc/rfc9110#section-8.8.3) |
| **Priority** | 🟡 Medium |
| **Effort** | Low |

**Gap**: PDF responses don't include an `ETag` header. Since PDFs are generated on-the-fly, a hash of the PDF bytes could serve as a strong ETag.

**Proposed Change**:
```kotlin
// In PdfGenerationResource.kt
val etag = MessageDigest.getInstance("SHA-256")
    .digest(pdfBytes)
    .joinToString("") { "%02x".format(it) }

return Response.ok(pdfBytes)
    .header("ETag", "\"$etag\"")
    // ...existing headers...
    .build()
```

**Benefit**: Clients can use `If-None-Match` to avoid re-downloading identical PDFs. Useful when the same template + data produces the same output.

---

### 2. RFC 9457 — Full Compliance (Extension Members)

| | |
|---|---|
| **RFC** | [RFC 9457 §3.1](https://www.rfc-editor.org/rfc/rfc9457#section-3.1) |
| **Priority** | 🔴 High |
| **Effort** | Low |

**Gap A — Validation error extension members**: The `ProblemDetail` data class doesn't include an `errors` or `violations` extension array for validation failures. RFC 9457 explicitly allows extension members.

**Proposed Change**:
```kotlin
data class ProblemDetail(
    val type: String = "about:blank",
    val title: String,
    val status: Int,
    val detail: String? = null,
    val instance: String? = null,
    val violations: List<FieldViolation>? = null,  // Extension member
)

data class FieldViolation(
    val field: String,   // JSON Pointer (RFC 6901) e.g., "/username"
    val message: String,
)
```

**Gap B — `instance` field never populated**: The `instance` property exists in the data class but is always `null`. It should be set to the request URI.

**Proposed Change**:
```kotlin
// In exception mappers, pass the request URI
@Context
private lateinit var uriInfo: UriInfo

// Then in toResponse():
ProblemDetail(
    // ...existing fields...
    instance = uriInfo.requestUri.path,
)
```

**Benefit**: Consumers get machine-readable field-level errors and can correlate errors with specific requests.

---

### 3. RFC 6901 — JSON Pointer (for Validation Errors)

| | |
|---|---|
| **RFC** | [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901) |
| **Priority** | 🔴 High (pairs with RFC 9457 extension members) |
| **Effort** | Low |

**Gap**: Validation error messages reference field names as plain strings (e.g., `"Username is required"`). They should use JSON Pointer syntax to identify the exact field in the request body.

**Proposed Change**:
```json
{
  "type": "/problems/validation-error",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request validation failed",
  "instance": "/api/v1/pdf/generate",
  "violations": [
    { "field": "/templateId", "message": "Template ID is required" },
    { "field": "/templateData", "message": "Template data is required" }
  ]
}
```

**Benefit**: API consumers can programmatically map errors to form fields. Standard across the industry (OpenAPI, JSON Schema, etc.).

---

### 4. RFC 6585 — Rate Limiting (429 Too Many Requests)

| | |
|---|---|
| **RFC** | [RFC 6585 §4](https://www.rfc-editor.org/rfc/rfc6585#section-4) |
| **Priority** | 🟡 Medium |
| **Effort** | Medium |

**Gap**: No rate limiting on the PDF generation endpoint. PDF rendering (iText + HTML → PDF) is CPU/memory-intensive. A burst of requests could exhaust resources.

**Proposed Change**:
- Add a rate limiter (e.g., Quarkus `quarkus-rate-limiter` or a `@RateLimited` interceptor)
- Return `429 Too Many Requests` with `Retry-After` header when limits are exceeded
- Add a corresponding `ProblemDetail` response:

```json
{
  "type": "/problems/rate-limit-exceeded",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "PDF generation rate limit exceeded. Try again in 30 seconds.",
  "retryAfter": 30
}
```

**Benefit**: Protects the service from overload. Essential for production workloads.

---

### 5. RFC 7240 / RFC 8144 — Prefer Header (respond-async)

| | |
|---|---|
| **RFC** | [RFC 7240](https://www.rfc-editor.org/rfc/rfc7240) |
| **Priority** | 🔴 High |
| **Effort** | Medium |

**Gap**: The service has two PDF generation paths — synchronous (REST) and asynchronous (Kafka) — but clients can't choose between them. The `Prefer` header provides a standard mechanism for this.

**Proposed Change**:
```
# Client requests async processing
POST /api/v1/pdf/generate
Prefer: respond-async
Content-Type: application/json

{"templateId": "certificate", "templateData": {...}}
```

**Response when `Prefer: respond-async` is present**:
```
HTTP/1.1 202 Accepted
Preference-Applied: respond-async
Content-Type: application/json
Location: /api/v1/pdf/jobs/550e8400-e29b-41d4-a716-446655440000

{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "ACCEPTED",
  "submittedAt": "2026-02-26T10:00:00Z"
}
```

**Response when header is absent** (current behavior):
```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="document-certificate-20260226100000.pdf"

<pdf bytes>
```

**Benefit**: Unifies the REST and Kafka flows behind a single endpoint. Clients choose sync vs. async without needing to know about Kafka. Follows IETF standards rather than inventing a custom mechanism.

---

### 6. RFC 8288 — Web Linking

| | |
|---|---|
| **RFC** | [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) |
| **Priority** | 🟢 Nice to have |
| **Effort** | Low |

**Gap**: No `Link` headers in responses. Useful for discoverability, especially in the async flow.

**Proposed Change**:
```
# On 202 Accepted (async flow)
Link: </api/v1/pdf/jobs/550e8400>; rel="monitor", </api/v1/templates/certificate>; rel="describedby"

# On 200 OK (sync flow)
Link: </api/v1/templates/certificate>; rel="describedby"
```

**Benefit**: HATEOAS-style discoverability without requiring full hypermedia. Clients can follow links to check job status or inspect template definitions.

---

### 7. RFC 9421 — HTTP Message Signatures

| | |
|---|---|
| **RFC** | [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) |
| **Priority** | 🟢 Nice to have |
| **Effort** | High |

**Gap**: S2S requests are authenticated via JWT but not signed at the HTTP message level. A compromised intermediary could modify request bodies while replaying a valid JWT.

**Proposed Change**: Sign HTTP request bodies using `Signature` and `Signature-Input` headers. The doc generator verifies the signature before processing.

**Benefit**: Defense-in-depth for high-security environments. Protects against request body tampering even when TLS termination happens at a load balancer.

**Note**: This is a significant effort and should only be pursued if the threat model justifies it. JWT + TLS is sufficient for most deployments.

---

### 8. RFC 9512 — YAML Media Type

| | |
|---|---|
| **RFC** | [RFC 9512](https://www.rfc-editor.org/rfc/rfc9512) |
| **Priority** | 🟢 Nice to have |
| **Effort** | Low |

**Gap**: The AsyncAPI specification (`asyncapi.yaml`) is a YAML file. If it were exposed via an HTTP endpoint, it should use `application/yaml` (registered by this RFC) rather than `text/plain`.

**Proposed Change**: If/when exposing the AsyncAPI spec via an endpoint, set `Content-Type: application/yaml`.

---

### 9. RFC 7807 → RFC 9457 Migration Verification

| | |
|---|---|
| **RFC** | [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) |
| **Priority** | 🟢 Nice to have |
| **Effort** | Low |

**Note**: RFC 9457 obsoletes RFC 7807. The key differences:
- `type` defaults to `"about:blank"` (you already do this ✅)
- The media type is still `application/problem+json` (no change ✅)
- Extension members are more formally defined (see gap #2 above)
- JSON Schema for Problem Details is now included in the RFC

**Action**: Add a comment in `GlobalExceptionHandlers.kt` referencing RFC 9457 (not 7807) to ensure future developers reference the correct standard.

---

## Priority Matrix

| Priority | RFC | Action | Effort | Impact |
|----------|-----|--------|--------|--------|
| 🔴 High | RFC 9457 | Add `violations` extension member + populate `instance` field | Low | Better API DX, machine-readable errors |
| 🔴 High | RFC 6901 | Use JSON Pointer for field references in validation errors | Low | Standard field identification |
| 🔴 High | RFC 7240 | Support `Prefer: respond-async` to unify REST + Kafka flows | Medium | Major architectural improvement |
| 🟡 Medium | RFC 6585 | Add rate limiting (429 + `Retry-After`) | Medium | Production resilience |
| 🟡 Medium | RFC 9110 | Add `ETag` header to PDF responses | Low | Conditional request support |
| 🟢 Nice | RFC 8288 | Add `Link` headers for resource relationships | Low | API discoverability |
| 🟢 Nice | RFC 9512 | Use `application/yaml` for AsyncAPI spec endpoint | Low | Standards compliance |
| 🟢 Nice | RFC 9421 | HTTP Message Signatures for S2S integrity | High | Security hardening |

---

## Proposed New RFC: Async Job Problem Details

### Motivation

The SironOne Doc Generator has a real-world pattern that currently lacks IETF standardization:

1. A client sends a **synchronous HTTP request** to generate a PDF
2. The service optionally processes it **asynchronously via Kafka**
3. The result (success/failure) is published to a **Kafka result topic**
4. If it fails, the client needs a **structured error** that bridges both the synchronous and asynchronous worlds

Every major cloud provider (AWS, Azure, GCP) and many SaaS platforms implement async job processing over HTTP, but **there is no standard for representing async job failures as Problem Details**, and no standard for bridging the synchronous request with the asynchronous outcome.

---

### Proposed Title

> **"Problem Details Extensions for Asynchronous Job Processing in HTTP APIs"**

---

### Target Working Group

**IETF HTTP API Working Group (`httpapi`)**
- This group actively maintains RFC 9457 and works on HTTP API patterns
- Recent work includes structured fields, idempotency keys, and rate limiting — async job processing fits naturally

---

### Abstract (Draft)

> This document defines standard extension members for RFC 9457 (Problem Details for HTTP APIs) to represent failures in asynchronous job processing pipelines. It introduces a set of well-known extension members (`jobId`, `jobStatus`, `submittedAt`, `completedAt`, `retryable`, `retryAfter`, `processingStage`) that allow HTTP API servers to communicate structured information about async job failures. Additionally, it defines a `monitor` link relation for RFC 8288 (Web Linking) and specifies the interaction between RFC 7240's `Prefer: respond-async` preference and Problem Details responses.

---

### Proposed Extension Members

```json
{
  "type": "/problems/async-job-failed",
  "title": "PDF Generation Failed",
  "status": 500,
  "detail": "Template rendering failed for template 'certificate'",
  "instance": "/api/v1/pdf/generate",

  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "jobStatus": "FAILED",
  "submittedAt": "2026-02-26T10:00:00Z",
  "completedAt": "2026-02-26T10:00:05Z",
  "retryable": true,
  "retryAfter": 30,
  "processingStage": "template-rendering",
  "correlationId": "req-abc-123",
  "asyncChannel": "com.imtf.sironone-doc-generator.pdf-job.result.v1"
}
```

#### Extension Member Definitions

| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `jobId` | `string` (UUID) | REQUIRED | Unique identifier for the async job. Assigned at submission time. |
| `jobStatus` | `string` (enum) | REQUIRED | Current status: `ACCEPTED`, `PROCESSING`, `COMPLETED`, `FAILED`, `CANCELLED`, `TIMED_OUT`. |
| `submittedAt` | `string` (RFC 3339) | REQUIRED | Timestamp when the job was accepted for processing. |
| `completedAt` | `string` (RFC 3339) | OPTIONAL | Timestamp when the job reached a terminal state. |
| `retryable` | `boolean` | OPTIONAL | Whether the client should retry the job. Defaults to `false`. |
| `retryAfter` | `integer` | OPTIONAL | Suggested wait time in seconds before retrying. Only meaningful when `retryable` is `true`. |
| `processingStage` | `string` | OPTIONAL | The stage at which the job failed (e.g., `validation`, `template-rendering`, `pdf-conversion`, `storage`). |
| `correlationId` | `string` | OPTIONAL | Client-supplied correlation ID from the original request, for tracing. |
| `asyncChannel` | `string` | OPTIONAL | The messaging channel (e.g., Kafka topic) where the result was/will be published. |

---

### Proposed Lifecycle Flow

```
Client                         API Server                    Message Broker
  │                                │                              │
  │─ POST /api/v1/pdf/generate ──▶│                              │
  │  Prefer: respond-async        │                              │
  │                                │── publish job ──────────────▶│
  │◀── 202 Accepted ──────────────│                              │
  │    Location: /jobs/{jobId}     │                              │
  │    Preference-Applied:         │                              │
  │      respond-async             │                              │
  │                                │                              │
  │─ GET /jobs/{jobId} ──────────▶│                              │
  │◀── 200 OK ────────────────────│                              │
  │    {"jobStatus": "PROCESSING"} │                              │
  │                                │◀── job result ──────────────│
  │─ GET /jobs/{jobId} ──────────▶│                              │
  │◀── 200 OK ────────────────────│                              │
  │    {"jobStatus": "COMPLETED",  │                              │
  │     "resultUrl": "/jobs/{id}/  │                              │
  │      result"}                  │                              │
  │                                │                              │
  │─ GET /jobs/{id}/result ──────▶│                              │
  │◀── 200 OK (PDF bytes) ────────│                              │
```

**On failure, the job status endpoint returns a Problem Detail**:
```
GET /api/v1/pdf/jobs/550e8400-e29b-41d4-a716-446655440000

HTTP/1.1 200 OK
Content-Type: application/problem+json

{
  "type": "/problems/async-job-failed",
  "title": "PDF Generation Failed",
  "status": 500,
  "detail": "Template 'certificate' contains invalid HTML",
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "jobStatus": "FAILED",
  "submittedAt": "2026-02-26T10:00:00Z",
  "completedAt": "2026-02-26T10:00:05Z",
  "retryable": false,
  "processingStage": "template-rendering"
}
```

---

### Batch Processing (Partial Failures)

For batch document generation, the RFC would define how to represent partial failures:

```json
{
  "type": "/problems/async-batch-partial-failure",
  "title": "Batch Processing Partially Failed",
  "status": 207,
  "detail": "3 of 5 documents generated successfully",
  "jobId": "batch-001",
  "jobStatus": "COMPLETED_WITH_ERRORS",
  "submittedAt": "2026-02-26T10:00:00Z",
  "completedAt": "2026-02-26T10:05:00Z",
  "results": [
    { "itemId": "doc-1", "status": "COMPLETED" },
    { "itemId": "doc-2", "status": "COMPLETED" },
    { "itemId": "doc-3", "status": "FAILED", "detail": "Invalid template" },
    { "itemId": "doc-4", "status": "COMPLETED" },
    { "itemId": "doc-5", "status": "FAILED", "detail": "Data validation error", "retryable": true }
  ]
}
```

---

### RFCs This Builds On

| RFC | Relationship |
|-----|-------------|
| [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) | Extends Problem Details with async job extension members |
| [RFC 7240](https://www.rfc-editor.org/rfc/rfc7240) | Uses `Prefer: respond-async` to request async processing |
| [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288) | Defines `monitor` link relation for job status polling |
| [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) | Uses `202 Accepted` and `Location` header semantics |
| [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) | Timestamps in extension members use this format |
| [RFC 9110 §15.3.3](https://www.rfc-editor.org/rfc/rfc9110#section-15.3.3) | Formal semantics of 202 Accepted |

---

### Why This RFC Is Needed

1. **No standard exists**: AWS, Azure, GCP, and Stripe all implement async job patterns differently. There's no interoperable standard.

2. **Problem Details gap**: RFC 9457 defines the envelope but not domain-specific extensions for common patterns like async jobs. The community needs reusable extension member definitions.

3. **Real-world demand**: Document generation, video transcoding, report building, ML inference, data exports — all follow the same pattern (submit → poll → retrieve). All need structured error reporting.

4. **Bridging sync and async**: Most APIs start synchronous and evolve to async. A standard `Prefer: respond-async` + Problem Details integration provides a smooth migration path.

5. **Observability**: Standard extension members (`processingStage`, `correlationId`, `submittedAt`/`completedAt`) enable tooling and dashboards to work across services without custom parsing.

---

### Submission Process

1. **Write the Internet-Draft** as `draft-ratnawat-httpapi-async-job-problem-details-00`
2. **Submit to the IETF Datatracker**: https://datatracker.ietf.org/submit/
3. **Present at IETF httpapi WG** meeting (virtual or in-person)
4. **Iterate** based on WG feedback
5. **Target**: Proposed Standard track

**Tooling**:
- Author in [xml2rfc](https://xml2rfc.tools.ietf.org/) or [kramdown-rfc](https://github.com/cabo/kramdown-rfc) (Markdown-based)
- Use [id-nits](https://www.ietf.org/tools/idnits/) for format validation
- Reference: [IETF Internet-Draft guidelines](https://www.ietf.org/standards/ids/)

---

### Implementation in SironOne Doc Generator

This project can serve as the **reference implementation**:

| Component | Change |
|-----------|--------|
| `PdfGenerationResource.kt` | Support `Prefer: respond-async` header; return `202 Accepted` with `Location` header |
| `GlobalExceptionHandlers.kt` | Add `AsyncJobProblemDetail` with the proposed extension members |
| Kafka result consumer | Produce Problem Detail JSON on failure (not just plain error messages) |
| New: `JobStatusResource.kt` | `GET /api/v1/pdf/jobs/{jobId}` endpoint for polling job status |
| AsyncAPI spec | Document the Problem Detail structure in result events |

---

## References

| RFC | Title | URL |
|-----|-------|-----|
| RFC 3339 | Date and Time on the Internet: Timestamps | https://www.rfc-editor.org/rfc/rfc3339 |
| RFC 6266 | Use of the Content-Disposition Header Field | https://www.rfc-editor.org/rfc/rfc6266 |
| RFC 6585 | Additional HTTP Status Codes | https://www.rfc-editor.org/rfc/rfc6585 |
| RFC 6749 | OAuth 2.0 Authorization Framework | https://www.rfc-editor.org/rfc/rfc6749 |
| RFC 6750 | OAuth 2.0 Bearer Token Usage | https://www.rfc-editor.org/rfc/rfc6750 |
| RFC 6901 | JavaScript Object Notation (JSON) Pointer | https://www.rfc-editor.org/rfc/rfc6901 |
| RFC 7240 | Prefer Header for HTTP | https://www.rfc-editor.org/rfc/rfc7240 |
| RFC 7517 | JSON Web Key (JWK) | https://www.rfc-editor.org/rfc/rfc7517 |
| RFC 7519 | JSON Web Token (JWT) | https://www.rfc-editor.org/rfc/rfc7519 |
| RFC 8144 | Use of the Prefer Header Field in WebDAV | https://www.rfc-editor.org/rfc/rfc8144 |
| RFC 8259 | The JSON Data Interchange Format | https://www.rfc-editor.org/rfc/rfc8259 |
| RFC 8288 | Web Linking | https://www.rfc-editor.org/rfc/rfc8288 |
| RFC 8615 | Well-Known Uniform Resource Identifiers | https://www.rfc-editor.org/rfc/rfc8615 |
| RFC 9110 | HTTP Semantics | https://www.rfc-editor.org/rfc/rfc9110 |
| RFC 9111 | HTTP Caching | https://www.rfc-editor.org/rfc/rfc9111 |
| RFC 9421 | HTTP Message Signatures | https://www.rfc-editor.org/rfc/rfc9421 |
| RFC 9457 | Problem Details for HTTP APIs | https://www.rfc-editor.org/rfc/rfc9457 |
| RFC 9512 | YAML Media Type | https://www.rfc-editor.org/rfc/rfc9512 |
