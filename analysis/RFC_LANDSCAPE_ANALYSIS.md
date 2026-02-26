# Landscape Analysis: Does This RFC Make Sense?

## draft-ratnawat-httpapi-async-problem-details-00

**Date:** 2026-02-26  
**Analysis by:** GitHub Copilot  
**Question:** Is there already something similar? Does this RFC fill a genuine gap?

---

## TL;DR Verdict

**YES, the RFC makes sense. NO, nothing equivalent exists today.**

But the value proposition is narrower than it appears, and there are
legitimate arguments that the problem doesn't need an RFC at all. This
analysis covers both sides honestly.

---

## 1. The Exact Claim This RFC Makes

The RFC claims to fill ONE specific gap:

> RFC 9457 provides the error envelope. Nobody has defined reusable
> extension members for the async job domain. Every API invents its
> own. This document standardizes them.

Let me decompose this into testable sub-claims:

| # | Sub-Claim | Verdict |
|---|-----------|---------|
| A | Async job processing is a common API pattern | ✅ TRUE — universally agreed |
| B | RFC 9457 exists and is the standard for HTTP API errors | ✅ TRUE — published July 2023 |
| C | RFC 9457 allows extension members but defines none | ✅ TRUE — Section 3.1 |
| D | No IETF spec defines reusable extension members for async jobs | ✅ TRUE — verified (see §2 below) |
| E | No non-IETF spec fills this gap either | ⚠️ PARTIALLY TRUE (see §3 below) |
| F | The fragmentation causes real problems | ⚠️ DEBATABLE (see §4 below) |
| G | Transport independence (Kafka/webhook) is a real need | ✅ TRUE — strongest argument |

---

## 2. IETF Landscape — What Actually Exists

### 2.1. Directly Relevant IETF Work

| Specification | What It Covers | What It DOESN'T Cover | Overlap? |
|---|---|---|---|
| **RFC 9457** (Problem Details) | Error envelope: type, title, status, detail, instance | Any extension members. No async-specific anything. | Foundation — this RFC extends it |
| **RFC 9110 §15.3.3** (202 Accepted) | "Request accepted for processing" semantics | What to do when the job fails. No structured failure format. | None — different lifecycle phase |
| **RFC 7240 §4.1** (respond-async) | Client preference for async handling | Nothing about failure reporting | None — different concern |
| **RFC 8288** (Web Linking) | Link relations for discovering resources | No job-specific link relations or failure semantics | None — complementary |
| **draft-ietf-httpapi-idempotency-key-header** | Idempotent request submission | Nothing about async outcomes | None |
| **draft-ietf-httpapi-ratelimit-headers** | Rate limit signaling for polling | Nothing about job failure structure | None |
| **RFC 6585** (429 Too Many Requests) | Throttling status code | Nothing about async jobs | None |

**Conclusion: There is NO IETF specification that defines structured error reporting for async job failures. The gap is real at the IETF level.**

### 2.2. IETF Work That COULD Have Covered This But Didn't

| Potential Source | Why It Doesn't Cover This |
|---|---|
| **RFC 9457 itself** | Explicitly punts on extension members — "domain-specific" |
| **HTTPAPI Working Group** | Has idempotency, rate-limiting, deprecation drafts. No async job error reporting. The WG charter focuses on HTTP API design patterns but hasn't tackled this specific gap. |
| **draft-ietf-httpapi-rest-robustness** (if it existed) | No such draft exists in the IETF datatracker |
| **draft-ietf-httpapi-api-health-check** | Covers service health, not individual job failures |

### 2.3. The Critical Missing Piece

The IETF has standardized the "happy path" of async jobs quite well:

```
Client → POST + Prefer:respond-async → Server
Client ← 202 Accepted + Location header ← Server
Client → GET /jobs/{id} → Server (poll)
Client ← 200 OK {status: "processing"} ← Server
Client → GET /jobs/{id} → Server (poll again)
Client ← 200 OK {status: "completed"} ← Server  ← RFC stops here
Client ← 200 OK {status: "FAILED", ???} ← Server  ← WHAT GOES HERE?
```

The `???` is genuinely unspecified. RFC 9457 says "use extension
members" but doesn't say which ones. Every API fills in `???`
differently. **This is the gap the RFC addresses.**

---

## 3. Non-IETF Landscape — What the Industry Has Built

This is where the analysis gets more nuanced. Several major efforts
exist outside the IETF:

### 3.1. Google AIP-151 (Long-Running Operations)

**URL:** https://google.aip.dev/151

| Aspect | Coverage | Comparison to This RFC |
|---|---|---|
| Job identifier | ✅ `name` field | Equivalent to `jobId` |
| Status tracking | ✅ `done` boolean + `Operation` resource | Simpler model (done/not-done vs. 6 status values) |
| Error reporting | ✅ `google.rpc.Status` (code + message + details) | Different format — Protobuf, not RFC 9457 |
| Retry guidance | ❌ Not specified | **Gap** — this RFC fills it |
| Processing stage | ❌ Not specified | **Gap** — this RFC fills it |
| Transport independence | ❌ Designed for gRPC only | **Gap** — this RFC fills it |
| Batch partial failure | ❌ Not specified | **Gap** — this RFC fills it |
| Built on RFC 9457? | ❌ No — uses google.rpc.Status | **Key difference** |

**Verdict:** AIP-151 is the closest competitor but is:
- Google-specific (not an open standard)
- gRPC/Protobuf-native (not JSON/REST)
- Doesn't use RFC 9457 as the envelope
- Missing retry guidance, processing stage, and transport independence

**This RFC offers genuine value over AIP-151 for REST/JSON APIs.**

### 3.2. Microsoft Azure Long-Running Operations (LRO)

**URL:** https://github.com/microsoft/api-guidelines/blob/vNext/azure/Guidelines.md

| Aspect | Coverage | Comparison |
|---|---|---|
| Job identifier | ✅ `id` | Equivalent |
| Status tracking | ✅ `status` enum (Succeeded/Failed/Cancelled) | Similar but fewer states |
| Error reporting | ✅ `error: {code, message}` | Not RFC 9457 |
| Retry guidance | ⚠️ `Retry-After` header only | **Gap** — header-only, lost in non-HTTP |
| Processing stage | ❌ Not specified | **Gap** |
| Transport independence | ❌ HTTP-only design | **Gap** |

**Verdict:** Azure LRO is HTTP-only and doesn't use RFC 9457. The retry
guidance is header-bound (lost in Kafka/webhook). This RFC adds value.

### 3.3. AWS Step Functions / Batch / Lambda

| Aspect | Coverage |
|---|---|
| Job identifier | ✅ `executionArn` / `jobId` |
| Status tracking | ✅ `RUNNING/SUCCEEDED/FAILED/TIMED_OUT/ABORTED` |
| Error reporting | ⚠️ `error` + `cause` (flat strings) |
| Retry guidance | ❌ Handled by Step Functions state machine, not in the error payload |
| Processing stage | ❌ Not in the error response |

**Verdict:** AWS has rich async job infrastructure but zero
standardization of the error payload format. Each service (Step
Functions, Batch, Lambda, SQS) uses different shapes.

### 3.4. CloudEvents (CNCF)

**URL:** https://cloudevents.io/

| Aspect | Coverage | Comparison |
|---|---|---|
| Event envelope | ✅ id, source, type, time, subject | Metadata about the event |
| Event payload | ❌ Completely user-defined | **Gap** — this RFC defines the payload |
| Error semantics | ❌ No concept of error/failure/retry | **Gap** |
| Transport independence | ✅ Designed for HTTP, Kafka, AMQP, MQTT | Aligned design principle |

**Verdict:** CloudEvents and this RFC are **complementary, not
competing**. CloudEvents defines the envelope for events; this RFC
defines what goes INSIDE when the event is a job failure. The RFC
already acknowledges this in Appendix D.

### 3.5. AsyncAPI

**URL:** https://www.asyncapi.com/

AsyncAPI is a specification for documenting message-driven APIs (like
OpenAPI for async). It defines how to DESCRIBE channels, messages,
and schemas — but it does NOT define standard schemas for common
patterns like job failures. You use AsyncAPI to document your
proprietary error format, whatever that happens to be.

**Verdict:** AsyncAPI is a documentation tool, not a data format
standard. No overlap. The RFC even shows how to integrate with
AsyncAPI (Appendix C).

### 3.6. JSON:API Errors

**URL:** https://jsonapi.org/format/#errors

JSON:API defines an error object with `id`, `status`, `code`, `title`,
`detail`, `source`, and `meta`. This is a competing error format to
RFC 9457, not an extension of it.

| Aspect | Coverage |
|---|---|
| General error structure | ✅ Rich error objects |
| Async job specifics | ❌ No job ID, status, retry, processing stage |
| Transport independence | ❌ HTTP-centric |

**Verdict:** Different ecosystem entirely. JSON:API competes with RFC
9457, not with this RFC's extension members.

### 3.7. GraphQL Errors

GraphQL has its own error format (`errors` array with `message`,
`locations`, `path`, `extensions`). The `extensions` field is
analogous to RFC 9457's extension members — free-form and
unstandardized. Same fragmentation problem exists.

**Verdict:** Different protocol. No overlap, but same underlying
problem.

### 3.8. OpenTelemetry / Distributed Tracing

OpenTelemetry defines trace context propagation (W3C Trace Context)
and span status (OK, ERROR, UNSET). It's about observability
instrumentation, not about structured error payloads sent to API
consumers.

**Verdict:** Complementary. The RFC's `correlationId` bridges to
trace context. No overlap.

---

## 4. The Hard Question: Does the Fragmentation Actually Hurt?

Your RFC argues that fragmentation is costly. Let me steelman both sides:

### Arguments FOR standardization (your RFC's position)

| Argument | Strength | Evidence |
|---|---|---|
| Monitoring tools can't parse across APIs | Strong | Datadog, New Relic, etc. all need custom parsers per API |
| Client SDKs must handle bespoke error shapes | Strong | Every AWS SDK, Azure SDK, etc. has custom error types |
| API designers reinvent the same fields | Moderate | True but design time is small vs. implementation time |
| Transport independence is genuinely unaddressed | **Very Strong** | Retry-After header is lost in Kafka. Nobody else solves this. |
| Processing stage info aids debugging | Moderate | Useful but not standardized anywhere |
| RFC 9457 adoption is growing | Strong | Major frameworks (Spring, ASP.NET, Quarkus) now produce problem+json |

### Arguments AGAINST standardization (devil's advocate)

| Argument | Strength | Rebuttal |
|---|---|---|
| "Nobody will adopt it — AWS/Google/Azure have their own patterns" | **Strong** | Fair point. Hyperscalers won't rewrite. But NEW APIs (startups, enterprises, open-source projects) might. |
| "RFC 9457 adoption itself is still limited" | Moderate | Growing rapidly — Spring Boot 3, ASP.NET Core 8, Quarkus all support it natively now |
| "Extension members don't need standardization — just document your API" | Moderate | True for single-API consumers. Falls apart for multi-API monitoring tools and generic client libraries. |
| "The industry already converged informally on similar field names" | Weak | The survey in Table 2 shows they have NOT converged — every platform uses different names |
| "This is too narrow for an RFC — just publish a blog post" | Moderate | Blog posts don't get adopted by frameworks. RFCs do. RFC 9457 itself started as an individual draft. |
| "Batch semantics and processing stages are too domain-specific" | Moderate | Processing stages ARE domain-specific. The RFC acknowledges this but argues structural presence has value even without standardized values. |

---

## 5. Competitive Matrix — What Exists vs. What This RFC Adds

```
                    Error    Job    Status   Retry    Processing  Transport   Batch    Built on
                    Envelope ID     Tracking Guidance Stage       Indep.     Failure  RFC 9457
                    ──────── ────── ──────── ──────── ─────────── ────────── ──────── ────────
RFC 9457            ✅       ❌     ❌       ❌       ❌          ❌         ❌       N/A
Google AIP-151      ❌(own)  ✅     ✅       ❌       ❌          ❌         ❌       ❌
Azure LRO           ❌(own)  ✅     ✅       ⚠️(hdr)  ❌          ❌         ❌       ❌
AWS Step Functions  ❌(own)  ✅     ✅       ❌       ❌          ❌         ❌       ❌
Stripe              ❌(own)  ✅     ✅       ❌       ❌          ❌         ❌       ❌
CloudEvents         ❌(meta) ❌     ❌       ❌       ❌          ✅         ❌       ❌
AsyncAPI            ❌(doc)  ❌     ❌       ❌       ❌          ❌         ❌       ❌
JSON:API Errors     ❌(own)  ❌     ❌       ❌       ❌          ❌         ❌       ❌
THIS RFC            ✅(9457) ✅     ✅       ✅       ✅          ✅         ✅       ✅
```

**No existing specification covers all columns. This RFC is the only
one that builds on the IETF standard error envelope (RFC 9457) AND
provides transport-independent retry semantics AND includes processing
stage identification.**

---

## 6. The Unique Contributions (What Nobody Else Has)

After thorough analysis, the RFC has **three genuinely novel
contributions** that no existing standard provides:

### 6.1. Transport-Independent Retry Semantics ⭐⭐⭐

**Novelty: HIGH — This is the RFC's killer feature.**

The `Retry-After` HTTP header exists in RFC 9110. But when a job
failure is communicated via Kafka, a webhook callback, or an SSE
stream, that header doesn't exist. Nobody — not Google, not AWS, not
Azure, not any IETF RFC — has solved the problem of "how do I tell a
Kafka consumer to retry in 60 seconds?"

The `retryable` + `retryAfter` JSON members solve this elegantly.
This alone justifies the RFC's existence.

### 6.2. Processing Stage Identification ⭐⭐

**Novelty: HIGH — Nobody else has this.**

The survey in Table 2 confirms: not one major platform includes
pipeline stage information in job failure reports. This RFC's
`processingStage` member (validation → queuing → processing →
rendering → conversion → storage → delivery) is genuinely novel and
practically useful for multi-step pipelines.

### 6.3. RFC 9457 as the Envelope for Async Errors ⭐⭐

**Novelty: MODERATE — Obvious in hindsight but nobody has done it.**

Every cloud provider invents a custom error object. None builds on
RFC 9457. This RFC is the first to say "RFC 9457 IS the right
envelope — here are the extension members it needs for async jobs."
This isn't revolutionary, but it's the right architectural choice and
nobody else has formalized it.

### Non-Novel Parts (Already Exist Elsewhere)

| Member | Already Exists In |
|---|---|
| `jobId` | Every async API ever (just named differently) |
| `jobStatus` | Every async API ever (just named/valued differently) |
| `submittedAt` / `completedAt` | Most async APIs (named differently) |
| `correlationId` | W3C Trace Context, X-Correlation-ID convention |
| `results` (batch) | Azure Batch, AWS Batch (proprietary formats) |

These are not novel — but standardizing their names and semantics
within RFC 9457 IS valuable, even if the concepts aren't new.

---

## 7. Who Would Actually Adopt This?

### Likely Adopters

| Audience | Why |
|---|---|
| **New REST APIs** being designed today | No legacy to maintain; RFC 9457 is already their error format |
| **Enterprise API platforms** (API gateways, mesh) | Need cross-API consistency for monitoring |
| **Open-source frameworks** (Spring, Quarkus, ASP.NET) | Already support RFC 9457; adding extension members is incremental |
| **Event-driven architectures** (Kafka, RabbitMQ users) | Transport independence is directly valuable |
| **API design guidelines** (Zalando, Adidas, gov.uk) | These already reference RFC 9457; would likely adopt standard extensions |

### Unlikely Adopters

| Audience | Why |
|---|---|
| **AWS / Google / Azure** native services | Already have deeply embedded proprietary patterns |
| **GraphQL APIs** | Different error model entirely |
| **Legacy REST APIs** | Migration cost outweighs benefit |
| **gRPC-first services** | google.rpc.Status is too entrenched |

---

## 8. Honest Assessment: Strengths and Weaknesses

### What Makes the RFC Worth Pursuing ✅

1. **The gap is real and verified** — No IETF spec and no open standard covers this
2. **Transport independence is genuinely novel** — The strongest differentiator
3. **Processing stage is genuinely novel** — Nobody else has it
4. **Timing is good** — RFC 9457 adoption is accelerating; extension members are the natural next step
5. **Scope is disciplined** — The "What This Document Is NOT" section is excellent
6. **Builds on existing standards** — Doesn't reinvent; extends
7. **Practical examples across transports** — HTTP, Kafka, webhook, SSE

### What Makes the RFC Risky ⚠️

1. **Adoption depends on RFC 9457 adoption** — If RFC 9457 stays niche, this RFC is dead
2. **Hyperscalers won't adopt** — AWS, Google, Azure have too much inertia
3. **"Standards Track" may be too ambitious** — "Informational" or "BCP" has a lower bar and might get published faster
4. **The batch section feels bolted on** — Could be a separate document
5. **No existing implementations to point to** — IETF values running code
6. **Single author from a single organization** — WG adoption requires broader support

### What's Missing That Would Strengthen It ⚠️

1. **A reference implementation** — Even a simple one in Spring Boot or Quarkus would help enormously
2. **Adoption by an API design guideline** — If Zalando or gov.uk referenced it, credibility jumps
3. **A real-world deployment report** — "We deployed this at IMTF and here's what happened"
4. **Framework integration** — A Spring Boot starter or Quarkus extension that produces these extension members automatically
5. **A companion client library** — That parses problem+json with these extensions and provides typed access

---

## 9. Final Verdict

### Does it make sense? **YES.**

The RFC addresses a genuine, verified gap that no existing IETF
specification, cloud provider pattern, or open standard fills.
The transport-independence argument is strong and novel. The
processing stage concept is unique. Building on RFC 9457 is the
right architectural choice.

### Is there something similar? **Partially, but nothing equivalent.**

- Google AIP-151 is the closest but is gRPC-specific, Google-proprietary, and missing retry/stage/transport features.
- Azure LRO has retry guidance but only in HTTP headers (lost in non-HTTP transports).
- CloudEvents is complementary, not competing.
- No existing specification combines RFC 9457 + async job context + transport independence.

### Should you pursue it? **YES, with adjustments.**

| Recommendation | Priority |
|---|---|
| Build a reference implementation in your Quarkus project | **Critical** — IETF values running code |
| Consider "Informational" track instead of "Standards Track" | **High** — lower bar, faster publication |
| Seek co-authors from other organizations | **High** — single-author/single-org drafts face skepticism |
| Get the Zalando RESTful API Guidelines or gov.uk API standards to reference it | **High** — instant credibility |
| Remove or simplify the batch section (§7) | **Medium** — keep the core tight |
| Present at an IETF HTTPAPI WG meeting | **Medium** — gauge WG interest before formal submission |
| Write a blog post explaining the transport-independence problem | **Medium** — builds community awareness |

### The One-Sentence Pitch

> "RFC 9457 tells you HOW to report HTTP API errors; this draft tells
> you WHAT to report when an async job fails — and it works even when
> the failure isn't delivered over HTTP."

That pitch is clear, differentiated, and defensible. **The RFC makes sense.**

---

## Appendix: Complete Landscape Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    IETF Standards                               │
│                                                                 │
│  RFC 9110 (202 Accepted)  ──── "accepted for processing"       │
│  RFC 7240 (respond-async) ──── "client prefers async"          │
│  RFC 8288 (Web Linking)   ──── "here's the status URL"         │
│  RFC 9457 (Problem Details) ── "here's the error envelope"     │
│                                     │                          │
│                              ┌──────┴──────┐                   │
│                              │  GAP: What  │                   │
│                              │  goes inside │                   │
│                              │  for async   │                   │
│                              │  job errors? │                   │
│                              └──────┬──────┘                   │
│                                     │                          │
│                          ┌──────────┴──────────┐               │
│                          │  THIS RFC FILLS IT  │               │
│                          │  • jobId             │               │
│                          │  • jobStatus         │               │
│                          │  • submittedAt       │               │
│                          │  • completedAt       │               │
│                          │  • retryable ⭐      │               │
│                          │  • retryAfter ⭐     │               │
│                          │  • processingStage ⭐│               │
│                          │  • correlationId     │               │
│                          └─────────────────────┘               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                 Industry Patterns (NOT standards)               │
│                                                                 │
│  Google AIP-151 ─── gRPC/Protobuf, no RFC 9457, no retry      │
│  Azure LRO      ─── HTTP-only, retry in header only           │
│  AWS Step Func.  ─── Custom JSON, no retry, no stage          │
│  Stripe          ─── Custom JSON, no retry, no stage          │
│  CloudEvents     ─── Event envelope only, no error semantics  │
│  AsyncAPI        ─── Documentation tool, no data format       │
│                                                                 │
│  None of these build on RFC 9457.                              │
│  None provide transport-independent retry.                     │
│  None include processing stage identification.                 │
└─────────────────────────────────────────────────────────────────┘
```
