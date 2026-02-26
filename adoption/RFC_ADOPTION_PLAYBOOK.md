# Getting API Design Guidelines to Reference Your RFC

## Playbook for draft-ratnawat-httpapi-async-problem-details-00

**Date:** 2026-02-26  
**Goal:** Get 2-3 major API design guidelines to adopt/reference the RFC's
extension members, establishing real-world credibility before IETF submission.

---

## Why This Matters

The IETF evaluates drafts on two axes:
1. **Technical merit** — Your RFC already has this.
2. **Evidence of demand** — "Who will actually use this?"

If you show up at an HTTPAPI WG meeting saying "Zalando's guidelines
already recommend these extension members," the conversation changes
from "will anyone adopt this?" to "people already are — let's
standardize it."

---

## Target API Guidelines (Ranked by Impact)

### Tier 1 — High Impact (public, widely referenced, already use RFC 9457)

| Guideline | Maintainer | RFC 9457 Status | Async Coverage | Repository | Stars |
|---|---|---|---|---|---|
| **Zalando RESTful API Guidelines** | Zalando SE | ✅ MUST use RFC 9457 (Rule #176) | ⚠️ Mentions 202 Accepted but no structured async error format | https://github.com/zalando/restful-api-guidelines | ~2.5k |
| **Microsoft REST API Guidelines** | Microsoft | ✅ Recommends RFC 9457 | ⚠️ Has Long-Running Operations section but custom error format | https://github.com/microsoft/api-guidelines | ~22k |
| **Google API Improvement Proposals (AIP)** | Google | ❌ Uses google.rpc.Status | ✅ AIP-151 covers LROs but gRPC-only | https://github.com/aip-dev/google.aip.dev | ~1.2k |

### Tier 2 — Medium Impact (government/sector standards)

| Guideline | Maintainer | RFC 9457 Status | Async Coverage | Repository |
|---|---|---|---|---|
| **UK Government API Standards** (gov.uk) | GDS (UK) | ✅ Recommends RFC 9457 | ❌ No async guidance | https://www.api.gov.uk/ |
| **Australian Government API Design Standard** | DTA (Australia) | ✅ References RFC 9457 | ❌ No async guidance | https://github.com/apigovau/api-design-guide |
| **Adidas API Guidelines** | Adidas | ✅ Uses RFC 9457 | ⚠️ Basic async patterns | https://github.com/adidas/api-guidelines |
| **Swiss Government eCH API Guidelines** | eCH (Switzerland) | ✅ References RFC 9457 | ❌ No async guidance | https://www.ech.ch/ |

### Tier 3 — Niche but Valuable (framework docs, community standards)

| Guideline | Maintainer | Why Valuable |
|---|---|---|
| **JSON:API** | jsonapi.org | Competing error format — unlikely to adopt, but worth monitoring |
| **Heroku Platform API Guidelines** | Heroku/Salesforce | Influential in the developer community |
| **PayPal API Standards** | PayPal | Major fintech with async patterns |

---

## Strategy Per Target

### 1. Zalando RESTful API Guidelines (BEST TARGET)

**Why Zalando first:**
- Most-referenced open-source API guidelines (~2.5k GitHub stars)
- Already MANDATES RFC 9457 for errors (Rule #176)
- Already covers 202 Accepted for async (Rule #152)
- Has a clear gap: no structured format for async job failures
- Active GitHub repository accepting contributions
- European company (same timezone/culture as IMTF)

**The gap in Zalando's guidelines today:**

Zalando Rule #152 says:
> "Servers SHOULD return 202 Accepted with Location header for async operations"

But says nothing about what the error response looks like when the
async job fails. Your RFC fills exactly this gap.

**Action plan:**

1. **Fork** https://github.com/zalando/restful-api-guidelines

2. **Find the relevant sections** to extend:
   - Chapter on error handling (RFC 9457 / Problem Details)
   - Chapter on async operations (202 Accepted / respond-async)

3. **Draft a pull request** that adds a new rule, something like:

   ```markdown
   ### SHOULD use standard extension members for async job failure reporting

   When an asynchronous operation (initiated per Rule #152) fails,
   the job status resource SHOULD return a Problem Details object
   (per Rule #176) that includes the following extension members
   from [draft-ratnawat-httpapi-async-problem-details]:

   | Member           | Type    | Description                          |
   |------------------|---------|--------------------------------------|
   | `jobId`          | string  | Unique identifier for the async job  |
   | `jobStatus`      | string  | Current state (FAILED, TIMED_OUT, etc.) |
   | `submittedAt`    | string  | RFC 3339 timestamp (UTC) of submission |
   | `completedAt`    | string  | RFC 3339 timestamp (UTC) of completion |
   | `retryable`      | boolean | Whether the client should retry      |
   | `retryAfter`     | integer | Seconds to wait before retrying      |
   | `processingStage`| string  | Pipeline stage where failure occurred |
   | `correlationId`  | string  | Client-supplied trace identifier     |

   Example:

       GET /jobs/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1

       HTTP/1.1 200 OK
       Content-Type: application/problem+json

       {
         "type": "https://api.example.com/problems/rendering-failed",
         "title": "Document Rendering Failed",
         "status": 500,
         "detail": "Template contains malformed HTML at line 42",
         "jobId": "550e8400-e29b-41d4-a716-446655440000",
         "jobStatus": "FAILED",
         "submittedAt": "2026-02-26T10:00:00Z",
         "completedAt": "2026-02-26T10:00:05Z",
         "retryable": false,
         "processingStage": "rendering"
       }

   This provides structured, machine-readable failure context that
   works consistently across APIs and transports.
   ```

4. **Write a PR description** that:
   - Links to the Internet-Draft
   - Explains the gap (Zalando covers 202 submission but not failure reporting)
   - Shows the industry fragmentation problem (AWS vs Azure vs Google field names)
   - Notes it builds on RFC 9457 which Zalando already mandates

5. **Engage on the PR** — Zalando's team is responsive. Be prepared
   for feedback and iterate.

**Template PR title:**
```
feat: Add structured async job failure reporting (RFC 9457 extensions)
```

**Template PR body:**
```markdown
## Problem

Zalando guidelines cover async job submission (Rule #152, 202 Accepted)
and error format (Rule #176, RFC 9457 Problem Details), but there's
no guidance on what the error response looks like when an async job
FAILS.

Today, different Zalando APIs report async failures differently:
- API A: `{"error": "failed", "jobId": "123"}`
- API B: `{"status": "FAILED", "id": "456", "reason": "..."}`
- API C: RFC 9457 Problem Details with ad-hoc extension members

## Proposal

Add a rule recommending standard extension members for async job
failure reports, based on [draft-ratnawat-httpapi-async-problem-details]
(an Internet-Draft extending RFC 9457 for async job context).

This gives Zalando APIs a consistent way to report:
- Which job failed (`jobId`)
- What its status is (`jobStatus`)
- When it was submitted/completed (`submittedAt`/`completedAt`)
- Whether it's retryable (`retryable`, `retryAfter`)
- Where in the pipeline it failed (`processingStage`)

## Why these specific members?

They're the result of surveying how AWS, Azure, Google, and Stripe
handle this (all differently) and identifying the common concepts
that every async API needs but names differently.

See the full analysis:
https://datatracker.ietf.org/doc/draft-ratnawat-httpapi-async-problem-details/
```

---

### 2. Microsoft REST API Guidelines

**Why Microsoft:**
- Massive reach (~22k GitHub stars)
- Already has a Long-Running Operations section
- Their LRO pattern uses a CUSTOM error format (not RFC 9457)
- An RFC-based alternative gives them an upgrade path

**The gap:**

Microsoft's guidelines define their own `OperationResult` schema:
```json
{
  "id": "job-xyz",
  "status": "Failed",
  "error": { "code": "RenderingFailed", "message": "..." }
}
```

This is NOT RFC 9457. Your RFC provides a path to align with standards.

**Action plan:**

1. **Don't propose replacing** their LRO pattern (too disruptive)
2. Instead, **open an issue** suggesting an alternative/recommended
   approach for new APIs that want RFC 9457 compliance
3. Frame it as: "For APIs that already use `application/problem+json`,
   here's how to extend it for async operations"

**Template GitHub Issue title:**
```
RFC 9457 Problem Details extension members for Long-Running Operation failures
```

**Template Issue body:**
```markdown
## Context

The guidelines currently define a custom error schema for LRO
failures (§13). Meanwhile, RFC 9457 (Problem Details for HTTP APIs)
is increasingly adopted as the standard error format for HTTP APIs.

## Suggestion

For APIs that use RFC 9457 as their error format, provide guidance
on standard extension members for LRO failures. An Internet-Draft
(draft-ratnawat-httpapi-async-problem-details) defines exactly these:

- `jobId`, `jobStatus`, `submittedAt`, `completedAt`
- `retryable`, `retryAfter` (transport-independent retry guidance)
- `processingStage` (which pipeline step failed)

This would complement the existing LRO guidance for APIs in the
RFC 9457 ecosystem.

## Benefit

APIs using RFC 9457 get consistent async failure reporting without
inventing their own extension members.
```

---

### 3. UK Government API Standards (gov.uk)

**Why gov.uk:**
- Government standards carry authority
- They already recommend RFC 9457
- UK government has many async APIs (tax processing, visa applications,
  benefit claims) where structured failure reporting matters
- HMRC APIs, Companies House API, etc. all process requests asynchronously

**Action plan:**

1. Contact the GDS (Government Digital Service) API standards team
2. Their standards are maintained at https://www.api.gov.uk/ and
   the Technology Code of Practice
3. **Write to:** api-standards@digital.cabinet-office.gov.uk
   (or the equivalent current contact)

**Template email:**

```
Subject: Structured async job failure reporting — RFC 9457 extension
         members for government APIs

Dear API Standards team,

The gov.uk API Technical and Data Standards recommend RFC 9457
(Problem Details for HTTP APIs) for error reporting. Many government
APIs also process requests asynchronously — tax calculations, visa
applications, benefit assessments — returning 202 Accepted and
providing results later.

Currently, there's no standard for what the error response looks
like when these async operations fail. Each API defines its own
fields for job identifiers, status values, and retry guidance.

I've authored an Internet-Draft that defines standard RFC 9457
extension members for async job failure reports:

  https://datatracker.ietf.org/doc/draft-ratnawat-httpapi-async-problem-details/

It defines 8 extension members (jobId, jobStatus, submittedAt,
completedAt, retryable, retryAfter, processingStage, correlationId)
that provide structured, machine-readable failure context.

Would the API standards team be interested in reviewing this for
potential inclusion as guidance for government APIs that process
work asynchronously?

Best regards,
Gaurav Ratnawat
IMTF
```

---

### 4. Swiss eCH API Standards

**Why eCH:**
- IMTF is based in Switzerland — this is your home market
- Swiss federal APIs increasingly follow eCH standards
- eCH-0056 (Application Interface Standard) references REST best practices
- Government APIs in Switzerland (tax, customs, identity) process
  work asynchronously

**Action plan:**

1. Check https://www.ech.ch/ for the current API working group
2. eCH standards are developed through Fachgruppen (expert groups)
3. Propose the extension members to the relevant Fachgruppe
4. Being Swiss-based at IMTF gives you direct access to participate

---

### 5. Adidas API Guidelines

**Why Adidas:**
- Open-source API guidelines on GitHub
- Already reference RFC 9457
- Smaller community = easier to get a PR merged
- Good stepping stone before tackling Zalando or Microsoft

**Action plan:** Same as Zalando — fork, PR, engage. This is a
good "practice run" before the higher-profile targets.

Repository: https://github.com/adidas/api-guidelines

---

## Execution Timeline

```
Week 1-2 (NOW):
├── Fork Zalando guidelines repo
├── Draft the PR with the new rule
├── Open Microsoft issue
└── Draft gov.uk email

Week 3-4:
├── Submit Zalando PR
├── Submit Microsoft issue
├── Send gov.uk email
├── Submit Adidas PR (practice run)
└── Contact eCH Fachgruppe

Week 5-8:
├── Engage on PR reviews (expect 2-3 rounds of feedback)
├── Iterate based on feedback
├── Follow up on gov.uk email if no response
└── Document adoption/interest for IETF submission

Week 9-12:
├── Reference any adoptions in the Internet-Draft
├── Update the RFC Acknowledgements to thank reviewers
└── Use adoption evidence in IETF HTTPAPI WG presentation
```

---

## What to Say When They Push Back

### "Why not just document your own extension members per-API?"

> "That's what everyone does today — and it's why every API uses
> different field names for the same concept. `executionArn` vs `id`
> vs `name` vs `jobId` for the exact same thing. Monitoring tools
> can't parse across APIs. Client libraries need custom error types
> per service. Standardizing the names lets tooling work generically."

### "Isn't this too narrow for a standard?"

> "RFC 9457 itself started as a narrowly-focused individual draft
> (RFC 7807 before it). 'Problem Details for HTTP APIs' seemed narrow
> too — now it's in Spring Boot, ASP.NET Core, and Quarkus. Async job
> error reporting is an equally common pattern that deserves the same
> treatment."

### "We'd rather wait for the RFC to be published before referencing it."

> "That's a chicken-and-egg problem — the IETF wants evidence of
> adoption before publishing, and you want publication before adopting.
> Referencing it as a RECOMMENDED practice (not REQUIRED) breaks the
> cycle. You can always strengthen the language when it's published."

### "Our existing async error format works fine."

> "It works fine for YOUR API. The problem is cross-API consistency.
> When a monitoring dashboard ingests errors from 50 different
> microservices, and each one reports async failures differently,
> you need a common schema. This is that schema."

### "retryable/retryAfter — we already have the Retry-After header."

> "The Retry-After header only exists in HTTP responses. When your
> async job failure is delivered via Kafka, a webhook, or an SSE
> stream, that header doesn't exist. The JSON members work across all
> transports. This is the RFC's most novel contribution."

---

## Measuring Success

| Metric | Bronze | Silver | Gold |
|---|---|---|---|
| GitHub issues/PRs opened | 3+ | 3+ | 3+ |
| Positive engagement (comments, not closed immediately) | 1 | 2+ | 3+ |
| PR merged or guideline updated | 0 | 1 | 2+ |
| Referenced in IETF presentation | "We proposed to X, Y, Z" | "X is reviewing" | "X adopted it" |

Even **Bronze** (PRs opened with positive engagement) is enough to
tell the IETF HTTPAPI WG: "Industry is interested."

---

## Template: Blog Post to Build Awareness

Before submitting PRs, consider publishing a blog post that you can
link to from every PR/issue. This gives reviewers context.

**Title:** "The Missing Standard: How Should Async Job Failures Look in REST APIs?"

**Outline:**

1. The problem: You poll `/jobs/123` and get `{"status": "failed"}` — now what?
2. How AWS, Azure, Google, and Stripe each answer this differently (table)
3. Why RFC 9457 is the right envelope (already adopted by Spring/ASP.NET/Quarkus)
4. The 8 extension members and what they solve
5. The transport-independence argument (Kafka/webhook example)
6. Link to the Internet-Draft
7. Call to action: "Review it, implement it, give feedback"

**Where to publish:**
- Medium / dev.to (broad developer audience)
- IMTF engineering blog (if one exists)
- LinkedIn article (professional network reach)
- IETF HTTPAPI WG mailing list (technical audience)

---

## Key Links You'll Need

| Resource | URL |
|---|---|
| Your Internet-Draft (after submission) | `https://datatracker.ietf.org/doc/draft-ratnawat-httpapi-async-problem-details/` |
| IETF HTTPAPI WG | `https://datatracker.ietf.org/wg/httpapi/about/` |
| HTTPAPI WG Mailing List | `https://www.ietf.org/mailman/listinfo/httpapi` |
| Zalando Guidelines Repo | `https://github.com/zalando/restful-api-guidelines` |
| Microsoft Guidelines Repo | `https://github.com/microsoft/api-guidelines` |
| Adidas Guidelines Repo | `https://github.com/adidas/api-guidelines` |
| gov.uk API Standards | `https://www.api.gov.uk/` |
| eCH Standards | `https://www.ech.ch/` |
| RFC 9457 | `https://www.rfc-editor.org/rfc/rfc9457` |
