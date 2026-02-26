# RFC Review: draft-ratnawat-httpapi-async-job-problem-details-00

**Document:** Problem Details Extensions for Asynchronous Job Processing in HTTP APIs  
**Reviewer:** GitHub Copilot (simulating IETF Area Director / expert reviewer)  
**Review Date:** 2026-02-26  
**Draft Version:** -00  
**Intended Status:** Standards Track  

---

## Revision Log

| Date | Action |
|---|---|
| 2026-02-26 | Initial review completed |
| 2026-02-26 | All findings applied to `draft-ratnawat-httpapi-async-problem-details-00.md`. See [Resolution Tracker](#17-resolution-tracker) for status of each issue. |

---

## Executive Summary

This is a well-motivated and clearly written first draft that addresses a genuine gap in the IETF standards landscape: structured error reporting for asynchronous job failures using RFC 9457 extension members. The problem statement is compelling, the extension members are well-chosen, and the examples are excellent.

However, the draft has **significant scope issues** that will likely draw pushback on the IETF mailing list. It tries to do too many things: define extension members, specify a complete async job lifecycle interaction model, define a new link relation, introduce batch semantics, AND define a JSON Schema — all in one Standards Track document. This overly broad scope creates internal contradictions and overlaps with existing/emerging IETF work.

**Recommendation:** Major revision needed before IETF Last Call. Focus the document on its core strength (the extension members) and split or remove the lifecycle/interaction material.

**Overall Quality:** 7/10 — Strong foundation, needs tightening.

---

## Table of Contents

1. [Pragmatism & Real-World Applicability](#1-pragmatism--real-world-applicability)
2. [Scope & Charter Fit](#2-scope--charter-fit)
3. [Technical Correctness](#3-technical-correctness)
4. [Normative Language (RFC 2119/8174)](#4-normative-language-rfc-21198174)
5. [Consistency & Internal Contradictions](#5-consistency--internal-contradictions)
6. [Security Considerations](#6-security-considerations)
7. [IANA Considerations](#7-iana-considerations)
8. [Relationship to Existing Work](#8-relationship-to-existing-work)
9. [Editorial & Formatting](#9-editorial--formatting)
10. [JSON Schema Review](#10-json-schema-review)
11. [Examples Review](#11-examples-review)
12. [Missing Content](#12-missing-content)
13. [Comparison with Your Other Draft](#13-comparison-with-your-other-draft)
14. [Section-by-Section Detailed Review](#14-section-by-section-detailed-review)
15. [Summary of Issues by Severity](#15-summary-of-issues-by-severity)
16. [Recommended Actions](#16-recommended-actions)

---

## 1. Pragmatism & Real-World Applicability

### ✅ Strengths

1. **Genuine problem**: The fragmentation shown in Appendix A (AWS vs Azure vs Stripe field names) is real and well-documented. Any API developer who has integrated with multiple cloud providers' async APIs will immediately recognize this pain.

2. **Transport independence**: The rationale for `retryable`/`retryAfter` as JSON members (not just HTTP headers) is excellent. Kafka, webhooks, and SSE are real transports where HTTP headers don't exist. This is the draft's strongest differentiator.

3. **Incrementally adoptable**: Servers can include any subset of the extension members. This is pragmatic — it doesn't require all-or-nothing adoption.

4. **Good examples**: The examples in Section 11 cover realistic scenarios (document rendering failure, timeout, batch partial failure, transient error).

### ⚠️ Concerns

5. **"REQUIRED when reporting an async job outcome"**: Sections 3.1, 3.2, and 3.3 say `jobId`, `jobStatus`, and `submittedAt` have "Cardinality: REQUIRED when reporting an async job outcome." But the preamble to Section 3 says "These extension members are OPTIONAL in any problem details object. A server MAY include any subset of them." This is a **direct contradiction**. Either they are REQUIRED or they are OPTIONAL — you cannot have both.

   **Fix:** Use "RECOMMENDED" instead of "REQUIRED" for cardinality, or define a profile/conformance level (e.g., "A conforming async job problem details object MUST contain jobId, jobStatus, and submittedAt").

6. **Status value extensibility may cause interop problems**: Section 4 says servers MAY define additional non-terminal status values, and clients that encounter an unrecognized value SHOULD treat it as "PROCESSING." This is pragmatic but creates a problem: how does a client distinguish between a server-defined non-terminal state and a typo/bug in the server implementation? There's no registry or namespace mechanism.

7. **"COMPLETED_WITH_ERRORS" appears out of nowhere**: Section 7.1 introduces `COMPLETED_WITH_ERRORS` as a batch-specific status value, but Section 4 (Job Status Values) doesn't list it. The JSON Schema in Section 8 *does* include it in the enum. This is inconsistent — if it's a valid status, it belongs in Table 1.

8. **No guidance on non-failure use**: The extension members are described as being for "async job outcomes" broadly, but the entire framing is about failures. Can/should a server return these members when a job COMPLETES successfully? The `completedAt` description implies yes (it covers COMPLETED), but the document title and motivation are all about failures. This ambiguity will confuse implementers.

9. **Batch semantics are under-specified**: The `results` array (Section 7.1) is a significant feature bolted on in one subsection. What's the maximum size? How does pagination work for 10,000-item batches? Can individual items have their own `processingStage`? The current spec doesn't address this.

---

## 2. Scope & Charter Fit

### ⚠️ Major Concern: Scope Creep

This document tries to be four things at once:

| Concern | Section(s) | Should it be here? |
|---|---|---|
| RFC 9457 extension members for async jobs | §3, §4 | ✅ YES — core contribution |
| Complete async job interaction model | §5 | ❌ NO — duplicates RFC 9110 + RFC 7240 |
| New "monitor" link relation | §6 | ⚠️ MAYBE — could be a separate 2-page draft |
| Batch partial failure semantics | §7 | ⚠️ MAYBE — significant enough for its own doc |

**Section 5** is the biggest problem. It re-specifies the entire submit→poll→retrieve→fail lifecycle using RFC 7240 and RFC 9110 semantics. This:

- Duplicates existing standards (RFC 9110 §15.3.3, RFC 7240 §4.1)
- Risks contradicting them if wording diverges
- Makes the document appear to be claiming ownership of the async pattern itself
- Will draw immediate pushback from httpapi WG members

**Recommendation:** Remove Section 5 entirely, or reduce it to a short "Interaction Guidance" section (1-2 paragraphs) that says "when used with RFC 7240 respond-async, the 202 response body SHOULD include the extension members defined in this document."

### Track Appropriateness

The document is marked **Standards Track**. Given that it defines extension members for an existing standard (RFC 9457), **Informational** or **Best Current Practice (BCP)** might be more appropriate for a -00 draft, unless the HTTPAPI WG has expressed interest in adopting it. Standards Track implies WG consensus and IESG review, which requires a higher bar.

---

## 3. Technical Correctness

### Issue 3.1: `retryAfter` MUST NOT constraint is too strict

Section 3.6 states:

> "MUST NOT be present when 'retryable' is 'false'."

This creates a problem: what if a server wants to say "this specific failure is not retryable, but you should wait 60 seconds before submitting a *different* job because we're under load"? The MUST NOT prevents this valid use case.

**Fix:** Change to "SHOULD NOT be present when 'retryable' is 'false'." Or clarify that `retryAfter` applies specifically to retrying the *same* job.

### Issue 3.2: `resultUrl` is used but never defined

Section 2.1 (the sequence diagram) shows `"resultUrl":"/jobs/{id}/result"` in a response body. Section 5.3 mentions it as an approach. But `resultUrl` is never defined as an extension member in Section 3. It's a phantom field — referenced but not specified.

**Fix:** Either define `resultUrl` as an extension member (Section 3.9 or similar), or remove it from the examples and only use the `Link` header with `rel="result"` approach.

### Issue 3.3: HTTP status code for failed job polling is confusing

Section 5.4 says the HTTP status code SHOULD be "200 OK" when polling for a failed job, because "the request to check the job status succeeded." This is technically correct but will confuse many developers. The `status` member inside the problem details is 500, but the HTTP response is 200. The `Content-Type` is `application/problem+json`, which most libraries associate with error responses.

This is not necessarily wrong, but it needs **much more** explanation and guidance. What should client-side HTTP libraries do when they see 200 with `application/problem+json`? Many Problem Details implementations treat `application/problem+json` as an error signal.

**Fix:** Add a paragraph explaining this distinction clearly, and consider whether the job status resource should return the HTTP status that matches the `status` member when the job has failed.

### Issue 3.4: Timestamps SHOULD require UTC

The `submittedAt` and `completedAt` members specify RFC 3339 format but don't require UTC. In distributed systems, timezone-ambiguous timestamps cause real bugs. Your other draft (draft-ratnawat-httpapi-async-problem-details-00) correctly says "MUST be in UTC (indicated by the 'Z' suffix)."

**Fix:** Add "The value SHOULD use UTC (the 'Z' time-offset) to avoid timezone ambiguity in distributed systems."

### Issue 3.5: `processingStage` values are not registered

Section 3.7 lists common values ("validation", "queuing", "processing", etc.) but provides no registry or registration procedure. This means different APIs will use different strings for the same concept (e.g., "rendering" vs "render" vs "template_rendering"). This undermines the interoperability goal.

**Fix:** Either create an IANA registry for processing stage values (heavy-handed) or explicitly state that values are API-specific and interoperability of stage values is a non-goal (the benefit is structured presence, not standardized values).

---

## 4. Normative Language (RFC 2119/8174)

### Issue 4.1: Overuse of MUST/SHOULD

The document uses normative keywords aggressively, sometimes inappropriately:

| Section | Statement | Issue |
|---|---|---|
| §3.1 | "The value MUST be assigned by the server" | ✅ Correct — strong requirement |
| §3.2 | "The value MUST be one of the registered status values" | ⚠️ But Section 4 allows extensions, so this MUST is immediately softened |
| §3, preamble | "A client MUST NOT assume that any particular extension member will be present" | ✅ Correct |
| §3.1 | "Cardinality: REQUIRED when reporting an async job outcome" | ❌ Contradicts the preamble's OPTIONAL |
| §5.1 | "The server MUST respond with 202 Accepted" | ❌ This re-specifies RFC 9110 behavior |
| §5.1 | "The server MUST include Preference-Applied" | ❌ This re-specifies RFC 7240 behavior |

**Fix:** Remove all MUST/SHOULD statements that re-specify behavior already defined in other RFCs. For cardinality, use RECOMMENDED consistently.

### Issue 4.2: Missing RFC 2119 application

The Notational Conventions (Section 1.1) correctly references BCP 14, but several lowercase "must" and "should" appear in the document where the normative form was clearly intended:

- §2.2: "the server must communicate" — should be "the server needs to communicate" (descriptive, not normative)
- §4: "Once a job reaches a terminal state, its status MUST NOT change" — ✅ correct use
- §5.4: "The HTTP status code...SHOULD still be '200 OK'" — ✅ correct use

---

## 5. Consistency & Internal Contradictions

### Issue 5.1: OPTIONAL vs REQUIRED (Critical)

As noted above, the preamble to Section 3 and the individual member definitions contradict each other on whether `jobId`, `jobStatus`, and `submittedAt` are OPTIONAL or REQUIRED.

### Issue 5.2: `COMPLETED_WITH_ERRORS` not in Table 1

Section 7.1 introduces this status value. The JSON Schema includes it. Table 1 in Section 4 does not.

### Issue 5.3: `resultUrl` used but undefined

Sequence diagram and Section 5.3 reference it; Section 3 doesn't define it.

### Issue 5.4: `correlationId` missing from Abstract

The Abstract lists the "well-known extension members" but omits `correlationId`, even though it's defined in Section 3.8. Either add it to the Abstract or explain why it's not listed.

### Issue 5.5: Inconsistent capitalization of status values

The document uses `COMPLETED`, `FAILED`, etc. (UPPER_CASE) consistently in the spec, but Section 5.2 shows `"jobStatus":"PROCESSING"` (no space after colon) while Section 5.4 shows `"jobStatus": "FAILED"` (with space). Minor, but in an RFC, consistency matters.

### Issue 5.6: "monitor" link relation vs "result" link relation

Section 6 defines the "monitor" link relation. Section 5.3 uses `rel="result"` in a Link header. The "result" link relation is NOT defined in this document and is NOT in the IANA Link Relation registry. This is a silent dependency on an undefined relation.

**Fix:** Either define `rel="result"` in the IANA Considerations, or use an existing registered relation like `rel="alternate"` or a custom URI-based relation.

---

## 6. Security Considerations

### ✅ Strengths

The security section covers four important areas:
1. Information exposure via `processingStage`
2. Denial of service via polling
3. Job identifier guessability
4. Retry amplification

These are relevant and well-articulated.

### ⚠️ Missing Considerations

5. **Correlation ID injection / log injection**: The document mentions "Servers MUST sanitize correlation identifiers before including them in log output to prevent log injection attacks" — good. But it doesn't address:
   - **XSS**: If the `correlationId` is displayed in a web dashboard, it could contain script tags.
   - **Header injection**: If the `correlationId` is echoed in an HTTP header, CRLF injection is possible.
   
   **Fix:** Add: "Servers MUST sanitize the 'correlationId' value before including it in any output context (logs, HTTP headers, HTML pages) to prevent injection attacks."

6. **Timing side channels**: The `submittedAt` and `completedAt` members reveal processing duration, which could be used to infer server load, input complexity, or even input content (e.g., longer processing time = larger document). Your other draft covers this; this one doesn't.

   **Fix:** Add a brief note about timing information leakage.

7. **Batch `results` information leakage**: In a multi-tenant batch, the `results` array could leak information about other tenants' items if not properly scoped.

   **Fix:** Add: "Servers MUST ensure that batch 'results' entries only contain information the requesting client is authorized to see."

8. **`retryAfter: 0` amplification**: Section 9.4 mentions retry amplification but doesn't specify a minimum value. A malicious server setting `retryAfter: 0` could cause a tight retry loop.

   **Fix:** Add: "Clients SHOULD enforce a minimum backoff interval (e.g., 1 second) regardless of the 'retryAfter' value."

---

## 7. IANA Considerations

### Issue 7.1: No registration of extension member names

The document defines 8 extension members (`jobId`, `jobStatus`, `submittedAt`, `completedAt`, `retryable`, `retryAfter`, `processingStage`, `correlationId`) but does NOT request their registration in any IANA registry. RFC 9457 does not define an extension member registry, but if this document aims to establish "well-known" extension members, it should either:

a. Request that IANA create a new "Problem Details Extension Members" registry, or
b. Explicitly state that no registration is needed and explain why (e.g., the names are self-describing and collision risk is low).

Your other draft (draft-ratnawat-httpapi-async-problem-details-00) addresses this in Section 10.1 — this draft should do the same.

### Issue 7.2: "monitor" link relation registration is incomplete

Section 10.2 requests registration of the "monitor" link relation but doesn't provide the full registration template required by RFC 8288 Section 2.1.1. The required fields are:

- Relation Name: monitor ✅
- Description: ✅
- Reference: ✅
- Notes: ❌ Missing
- Application Data: ❌ Missing

**Fix:** Provide the complete registration template per RFC 8288.

### Issue 7.3: `rel="result"` is used but not registered

As noted in Issue 5.6, Section 5.3 uses `rel="result"` which is not a registered IANA link relation. Either register it or don't use it.

### Issue 7.4: Job Status Value registry

Section 4 defines 6 status values, and Section 7 introduces a 7th (`COMPLETED_WITH_ERRORS`). There is no IANA registry proposed for these values. If servers can extend them (Section 4 allows additional non-terminal states), a registry with a registration policy (e.g., "Specification Required" or "Expert Review") would help interoperability.

**Fix:** Either propose a "Job Status Values" registry or explicitly state that no registry is created and that server-defined values are API-local.

---

## 8. Relationship to Existing Work

### Issue 8.1: Overlap with draft-ietf-httpapi-link-hint (if it exists)

Check whether the HTTPAPI WG has ongoing work on async operations or long-running operations. Google's AIP-151 and Microsoft's Azure LRO pattern have both been discussed. If there's a competing or complementary draft, this document needs to reference it.

### Issue 8.2: Overlap with OpenAPI Specification

The OpenAPI Specification has its own conventions for async operations (callbacks, webhooks). This document doesn't mention OpenAPI at all. Given that most REST API developers use OpenAPI, a brief note on how these extension members map to OpenAPI would be valuable (could be an appendix).

### Issue 8.3: No mention of CloudEvents

[CloudEvents](https://cloudevents.io/) (CNCF specification, used by many cloud providers) defines a standard envelope for events, including async job completion events. The overlap with `correlationId` (CloudEvents has `source`, `subject`, `id`) should be acknowledged.

### Issue 8.4: Relationship to RFC 7240 `respond-async` needs clarification

The document specifies behavior when `Prefer: respond-async` is used, but doesn't clarify: do these extension members apply ONLY when `respond-async` is used? What about servers that always process asynchronously (no Prefer header needed)?

**Fix:** Add a sentence clarifying that the extension members are useful regardless of whether `Prefer: respond-async` was used in the originating request.

---

## 9. Editorial & Formatting

### Issue 9.1: RFC Format

The document is in a markdown file with RFC-style formatting. For actual IETF submission, it would need to be in xml2rfc format (RFC 7991). The current format is fine for a pre-submission draft.

### Issue 9.2: Section numbering

Section 1.1 (Notational Conventions) is a subsection of Section 1 (Introduction). In IETF convention, this is fine, but the Table of Contents shows "1.1." without proper indentation hierarchy with respect to sections like "2.1." and "2.2." — this is just a formatting artifact.

### Issue 9.3: Table formatting

The tables use ASCII art that mostly works but has some alignment issues. In xml2rfc these would be proper `<table>` elements.

### Issue 9.4: Missing "Conventions Used in This Document" for JSON

The document heavily uses JSON examples but doesn't state that JSON is defined by [RFC 8259] (or the older RFC 7159). Add a normative reference to RFC 8259.

### Issue 9.5: Acknowledgements section is too product-specific

The Acknowledgements mention "SironOne Doc Generator" — a specific commercial product at IMTF. While it's fine to mention real-world motivation, the current text reads like a product advertisement. IETF convention is to keep Acknowledgements focused on people and WGs who contributed.

**Fix:** Rephrase to focus on the pattern (e.g., "The concepts in this document were developed while building production PDF generation services that process asynchronous requests via Apache Kafka") and thank individuals by name.

### Issue 9.6: Abstract length

The Abstract is concise and well-written. ✅ No issues.

### Issue 9.7: No "Changes from Previous Versions" section

Not needed for -00, but establish the convention for -01.

---

## 10. JSON Schema Review

### Issue 10.1: Schema uses `$id` with ietf.org domain

```json
"$id": "https://ietf.org/draft/async-job-problem-details/schema"
```

You cannot use an `ietf.org` URI unless the IETF publishes it. Use your own domain or a placeholder like `urn:ietf:params:...`.

**Fix:** Use `"$id": "urn:ietf:draft:ratnawat-httpapi-async-job-problem-details:schema"` or similar.

### Issue 10.2: `dependentSchemas` constraint for `retryAfter`

The schema includes:

```json
"dependentSchemas": {
    "retryAfter": {
        "properties": {
            "retryable": { "const": true }
        },
        "required": ["retryable"]
    }
}
```

This correctly enforces that `retryAfter` requires `retryable: true`. However, it doesn't enforce the reverse constraint from the spec: "MUST NOT be present when retryable is false." The schema only validates that if `retryAfter` IS present, `retryable` must be true. A server sending `{"retryable": false, "retryAfter": 30}` would fail schema validation — which is correct. ✅

### Issue 10.3: Missing `additionalProperties`

The schema doesn't set `additionalProperties: false`, which is correct — RFC 9457 explicitly allows additional members. ✅

### Issue 10.4: No `required` array at top level

The schema has no top-level `required` array, consistent with the "all OPTIONAL" preamble. But this contradicts the "REQUIRED when reporting an async job outcome" cardinality on `jobId`, `jobStatus`, `submittedAt`.

**Fix:** Resolve the OPTIONAL vs REQUIRED contradiction first, then update the schema.

### Issue 10.5: Schema doesn't validate `jobStatus` extensibility

The schema uses `"enum": [...]` for `jobStatus`, which means any server-defined custom status values (allowed by Section 4) would fail schema validation. This contradicts the extensibility allowed in the prose.

**Fix:** Remove the `enum` constraint and use a comment explaining that the listed values are the registered set, or add a note that validators should accept additional values.

---

## 11. Examples Review

### ✅ Strengths

The examples are realistic, well-structured, and cover a good range of scenarios:
- §11.1: Rendering failure (most common case) ✅
- §11.2: Timeout with retry guidance ✅
- §11.3: Batch partial failure ✅
- §11.4: Transient error with retry ✅

### ⚠️ Issues

### Issue 11.1: No success example

All examples show failures. Since the extension members are described as being for "async job outcomes" (not just failures), at least one COMPLETED success example should be included.

### Issue 11.2: No non-HTTP transport example

The draft's strongest argument is transport independence, yet every example is an HTTP response. Add a Kafka message payload or webhook callback example to demonstrate transport independence.

### Issue 11.3: Example §11.1 includes `correlationId` but the description doesn't mention it

The example JSON includes `"correlationId": "invoice-batch-2026-02-26"` but the prose description only says "The template rendering stage failed due to invalid template markup." The `correlationId` value suggests this is part of a batch — but the example is in §11.1 (Document Generation Failure), not §11.3 (Batch). Minor confusion.

---

## 12. Missing Content

### Issue 12.1: No Interoperability Considerations

An IETF document should discuss how implementations of this spec interoperate. What does a client do if it receives a Problem Details object with `jobId` but without `jobStatus`? What about `retryAfter` without `retryable`?

### Issue 12.2: No Versioning / Evolution Strategy

What happens when a -01 or v2 adds new extension members? How do clients handle unknown extension members from future versions? (RFC 9457 handles this generically — "ignore unknown members" — but it's worth restating in context.)

### Issue 12.3: No Privacy Considerations

The `jobId`, `correlationId`, and timestamps could constitute personally identifiable information (PII) or enable tracking. A brief privacy consideration is warranted, especially post-GDPR.

### Issue 12.4: No Internationalization Considerations

The `detail` field is human-readable text. What language should it be in? Should `Accept-Language` from the original request be honored? RFC 9457 Section 3.1.4 discusses this for `title` — this document should reference that guidance.

### Issue 12.5: No guidance on Problem Type URIs

The examples use URIs like `https://api.example.com/problems/rendering-failed`. The document doesn't provide guidance on how to structure problem type URIs for async job failures, or whether to use `about:blank` (per RFC 9457 §4.2.1).

### Issue 12.6: No XML representation

RFC 9457 defines both JSON and XML representations. This document only addresses JSON. A brief note saying "this document defines extension members for the JSON representation only" or providing the XML mapping would be complete.

---

## 13. Comparison with Your Other Draft

You have a second draft: `draft-ratnawat-httpapi-async-problem-details-00` (without "job" in the name). Having reviewed both, the second draft is **significantly better** in several ways:

| Aspect | This Draft (-job-) | Other Draft (no "job") |
|---|---|---|
| **Scope** | Overreaches (lifecycle + members + link rel + batch) | Focused (extension members + transport independence) |
| **"What this is NOT" section** | Absent | Present and excellent (§1.1) |
| **Relationship table** | Absent | Present (§1.2, Table 1) — very effective |
| **Transport independence** | Mentioned but not demonstrated | Central thesis, with Kafka/webhook/SSE examples |
| **Terminology section** | Informal (§1.1) | Formal (§1.4) with clear definitions |
| **Extension member grouping** | Flat list | Organized by category (Identification / State / Retry / Diagnostics) |
| **UTC requirement for timestamps** | Missing | Present ("MUST be in UTC") |
| **completedAt constraints** | Vague | "MUST NOT be present when job is in non-terminal state" |
| **retryable semantics** | Basic | Detailed (true = transient, false = deterministic) |
| **IANA considerations** | Incomplete | Includes extension member registration and job status registry |
| **Security: timing side channels** | Missing | Covered (§9.5) |
| **CloudEvents / AsyncAPI mention** | Missing | Appendix C |

**Strong recommendation:** Merge the best parts of both drafts. The other draft has better structure and narrower focus. Use it as the base and incorporate the good examples and comparison table from this draft.

---

## 14. Section-by-Section Detailed Review

### Abstract ✅
Clear, concise, well-scoped. Minor: missing `correlationId` from the member list.

### Section 1 (Introduction) ✅
Well-motivated. The four goals are clearly stated. Suggest adding a fifth: "Provide guidance for non-HTTP transports."

### Section 1.1 (Notational Conventions) ✅
Correct BCP 14 boilerplate. Good addition of domain-specific terms.

### Section 2 (Async Job Processing Pattern) ⚠️
The sequence diagram is excellent and very clear. However, this section is background/context, and it's quite long for a Standards Track document. Consider moving to an Informative appendix.

### Section 2.2 (Unstructured Async Failures) ✅
The three-provider comparison is compelling. Would be even stronger with real URLs (anonymized) instead of "Provider A/B/C."

### Section 3 (Extension Members) ⚠️
Core contribution — well-written individual definitions. The OPTIONAL/REQUIRED contradiction must be fixed.

### Section 4 (Job Status Values) ⚠️
Good table. Missing `COMPLETED_WITH_ERRORS`. Needs clarity on extensibility mechanism.

### Section 5 (Prefer: respond-async Interaction) ❌
Remove or dramatically reduce. This re-specifies existing standards and creates scope problems.

### Section 6 (Monitor Link Relation) ⚠️
Valid contribution but could be a separate, tiny document. The registration template is incomplete.

### Section 7 (Batch Processing) ⚠️
Under-specified. Either develop fully or remove and mention as future work.

### Section 8 (JSON Schema) ⚠️
Useful but has the `$id`, `enum`, and `required` issues noted above.

### Section 9 (Security Considerations) ⚠️
Good foundation. Missing timing, privacy, and batch-specific considerations.

### Section 10 (IANA Considerations) ⚠️
Incomplete registration templates. Missing extension member and status value registries.

### Section 11 (Examples) ✅
Excellent scenarios. Add a success example and a non-HTTP transport example.

### Section 12 (References) ✅
Appropriate normative/informative split. Add RFC 8259 (JSON).

### Appendix A (Comparison) ✅
Useful. The table is clear and makes the case for standardization effectively.

### Acknowledgements ⚠️
Too product-specific. Rephrase.

---

## 15. Summary of Issues by Severity

### 🔴 Critical (Must fix before -01)

| # | Issue | Section | Status |
|---|---|---|---|
| C1 | OPTIONAL vs REQUIRED contradiction for jobId, jobStatus, submittedAt | §3 preamble vs §3.1-3.3 | ✅ RESOLVED — §3.1.1 Conformance Levels added |
| C2 | `COMPLETED_WITH_ERRORS` not in Table 1 but in schema | §4 vs §7.1 vs §8 | ✅ RESOLVED — already in Table 4 of this draft |
| C3 | `resultUrl` used but never defined as extension member | §2.1, §5.3 | ✅ RESOLVED — this draft does not use `resultUrl` |
| C4 | `rel="result"` used but not registered | §5.3 | ✅ RESOLVED — §6.5 now uses URI-based extension relations per RFC 8288 §2.1.2 |
| C5 | Schema `$id` uses unauthorized ietf.org domain | §8 | ✅ RESOLVED — already uses `urn:ietf:params:` in this draft |

### 🟠 Major (Should fix before WG adoption)

| # | Issue | Section | Status |
|---|---|---|---|
| M1 | Section 5 re-specifies RFC 9110 + RFC 7240 — scope creep | §5 | ✅ RESOLVED — this draft's §5 is "Transport Contexts" (guidance only), not lifecycle re-specification |
| M2 | No extension member IANA registry proposed | §10 | ✅ RESOLVED — already in §10.1 of this draft |
| M3 | No job status value IANA registry proposed | §10 | ✅ RESOLVED — already in §10.2 of this draft |
| M4 | Incomplete "monitor" link relation registration template | §10.2 | ✅ RESOLVED — this draft does not define a new link relation; references existing "monitor" from RFC 7089 |
| M5 | `retryAfter` MUST NOT is too strict | §3.6 | ✅ RESOLVED — softened to SHOULD NOT with exceptional-case guidance |
| M6 | JSON Schema `enum` prevents jobStatus extensibility | §8 | ✅ RESOLVED — `enum` removed; description lists registered values; note added about local validators |
| M7 | Timestamps should require/recommend UTC | §3.3, §3.4 | ✅ RESOLVED — already "MUST be in UTC" in this draft |
| M8 | Missing normative reference to RFC 8259 (JSON) | §12 | ✅ RESOLVED — RFC 8259 added to §12.1 normative references |
| M9 | `correlationId` missing from Abstract | Abstract | ✅ RESOLVED — Abstract now lists all eight members including `correlationId` |

### 🟡 Minor (Fix before IETF Last Call)

| # | Issue | Section | Status |
|---|---|---|---|
| m1 | No success example | §11 | ✅ RESOLVED — §11.7 added (HTTP Response: Successful Job Completion) |
| m2 | No non-HTTP transport example | §11 | ✅ RESOLVED — §11.3 (Kafka failure) already existed; §11.8 added (Kafka transient failure with retry) |
| m3 | Overuse of MUST for re-specified behavior | §5.1 | ✅ RESOLVED — this draft's §5 is guidance, not re-specification |
| m4 | Acknowledgements too product-specific | Ack | ✅ RESOLVED — product name removed; rephrased to focus on pattern |
| m5 | No privacy considerations | §9 | ✅ RESOLVED — §9.7 Privacy Considerations added |
| m6 | No internationalization note for `detail` field | §3 | ✅ RESOLVED — Appendix F.5 Internationalization of "detail" added |
| m7 | Batch semantics under-specified (pagination, max size) | §7 | ✅ RESOLVED — §7.1.1 Payload Size Considerations added |
| m8 | Missing interoperability considerations | - | ✅ RESOLVED — Appendix F (Interoperability Considerations) added with 5 subsections |
| m9 | `processingStage` values not registered or namespaced | §3.7 | ✅ RESOLVED — explicit registry note added explaining values are API-specific by design |
| m10 | HTTP 200 for failed job polling needs more explanation | §5.4 | ✅ RESOLVED — §11.1 note expanded with detailed explanation of 200 + problem+json semantics |

### 🟢 Nits (Fix anytime)

| # | Issue | Section | Status |
|---|---|---|---|
| n1 | Inconsistent JSON spacing in examples | §5.2 vs §5.4 | ✅ RESOLVED — this draft has consistent formatting |
| n2 | "must" (lowercase) used where descriptive intent is clear | §2.2 | ✅ RESOLVED — this draft uses descriptive language properly |
| n3 | No XML representation note | - | ✅ RESOLVED — Appendix F.4 XML Representation added |
| n4 | correlationId example §11.1 implies batch context | §11.1 | ✅ RESOLVED — §11.1 now uses trace-id style correlationId |

---

## 16. Recommended Actions

### For -01 revision:

1. ~~Fix the OPTIONAL/REQUIRED contradiction~~ ✅ Conformance levels (§3.1.1) resolve this
2. ~~Add `COMPLETED_WITH_ERRORS` to Table 1~~ ✅ Already in Table 4
3. ~~Either define `resultUrl` as an extension member or remove it~~ ✅ Not used in this draft
4. ~~Register or remove `rel="result"`~~ ✅ Now uses URI-based extension relations
5. ~~Fix the JSON Schema `$id`~~ ✅ Uses `urn:ietf:params:` format
6. ~~Reduce Section 5 to guidance only~~ ✅ Already guidance-only in this draft
7. ~~Add UTC requirement for timestamps~~ ✅ Already present
8. ~~Add RFC 8259 to normative references~~ ✅ Added
9. ~~Add `correlationId` to Abstract~~ ✅ Added
10. ~~Consider merging with your other draft~~ ✅ This draft is the primary document

### For WG adoption call:

11. ~~Propose IANA registries~~ ✅ §10.1 and §10.2 already propose registries
12. ~~Complete the "monitor" link relation registration template~~ ✅ No new link relation defined; references RFC 7089
13. ~~Add security considerations for timing, privacy, batch scoping~~ ✅ §9.5, §9.6, §9.7 added
14. ~~Add non-HTTP transport examples~~ ✅ §11.3, §11.8 (Kafka), §11.4 (webhook), §11.5 (SSE)
15. ~~Address relationship to CloudEvents and OpenAPI~~ ✅ Appendix D and E added

### Before IETF Last Call:

16. Convert to xml2rfc format — **REMAINING**
17. ~~Add interoperability and internationalization considerations~~ ✅ Appendix F added
18. Resolve Standards Track vs Informational question — **REMAINING** (recommend discussing with WG)
19. Get implementor feedback (at least 2 independent implementations) — **REMAINING**
20. ~~Rephrase Acknowledgements~~ ✅ Rephrased

---

## 17. Resolution Tracker

**Total issues found:** 28  
**Issues resolved:** 26 ✅  
**Issues remaining:** 2 ⏳  

Remaining items:
1. **Convert to xml2rfc format** — Required for IETF submission tooling
2. **Standards Track vs Informational** — Discuss with HTTPAPI WG chairs
3. **Independent implementations** — Need 2+ implementations for Standards Track

---

*Review generated 2026-02-26. This review simulates the depth and style of an IETF Area Director review but is not an official IETF review.*
