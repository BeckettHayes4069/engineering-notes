# Password Reset Email Evidence: Node.js API Domain Authentication and Report Dispatch

TL;DR: Treat a password-reset email and a generated student-report attachment as separate evidence-bearing actions. For a small edtech service, the practical default is an application-owned evidence ledger: pin the rendered template version, authenticate a dedicated custom domain with aligned DKIM or SPF, hash the PDF, and reconcile submission status without calling “accepted” a delivery. That keeps the compliance story reviewable while the mail transport stays replaceable.

| Evidence design | What a reviewer can establish | Build burden | Decision |
| --- | --- | --- | --- |
| Provider activity screen only | A transport recorded an event | Low | Too weak as the system of record |
| Application ledger plus transport receipts | Intent, exact artifact, handoff, and later state | Moderate | Default for attached reports |
| Authenticated portal with download log | A fresh access decision and a download event | High | Runner-up for revocable access |

**Recommendation:** keep the application ledger authoritative and use transport status as attached evidence. The two criteria that decide this are evidence completeness and evidence independence. A one-person SaaS can ship this incrementally; building a second mail system or a full document portal before policy requires it is a poor revenue-per-hour trade.

## How should a Node.js transactional email API record password reset delivery?

Start with claims, not fields. A useful record should let a reviewer distinguish five moments: a recovery message was requested, the transport accepted it, the reset credential was redeemed, a particular report was authorized, and a particular PDF was submitted for delivery. Those events don't mean the same thing.

The distinction matters because a transactional API's acceptance response establishes handoff, not inbox placement or human access. Preserve the remote message identifier and its reported state, but don't promote that state into a stronger claim. `accepted`, `delivered`, and “opened by the intended guardian” are three different assertions. Picture a support request from a school administrator: the ledger can show that `recovery-v4` was submitted, that the credential was redeemed, that a newly authorized report job rendered `report-dispatch-v2`, and that the PDF digest attached to that job matches the retained artifact. It still cannot show that the guardian read the attachment. That final claim would require different evidence, so the support answer must stop short of it.

Claims drift.

Keep reset credentials out of logs. Also keep student names and report content out of transport metadata. An opaque job identifier can join records inside the application, while a SHA-256 digest identifies the attached bytes without copying those bytes into the evidence ledger.

This is the first decision criterion: **can each recorded fact support the exact claim made about it?** If the answer depends on interpreting a screenshot months later, the design is fragile.

Template evidence needs the same discipline. Store an immutable version such as `recovery-v4` or `report-dispatch-v2` with the send attempt. Record the variables used only when their data classification permits it. The goal isn't to archive every secret; it is to show which approved communication logic ran.

## Domain authentication is evidence, not decoration

A custom visible sender gives recipients a stable identity. DMARC then connects that visible `From` domain to authentication by requiring alignment with an SPF-authenticated domain or a DKIM signing domain. RFC 7489 defines both alignment and the policy/reporting model.

For compliance evidence, retain the domain configuration as controlled infrastructure and retain dated verification results. DNS proves what is published now, not what was published for an earlier send. A change record closes that gap.

DKIM and SPF do different jobs in this chain. DKIM associates a cryptographic signature with the message and signing domain. SPF evaluates whether the sending host is authorized for the relevant envelope domain. DMARC evaluates alignment against the visible author domain. A DMARC record declares `v=DMARC1`; its policy can rely on either of two aligned authentication mechanisms. Don't collapse all of that into a single “domain verified” boolean.

Roll policy forward only after legitimate senders are inventoried and aggregate reports are reviewed. That recommendation follows the staged deployment guidance in RFC 7489: monitoring and limited rollout reduce the chance that an incomplete inventory blocks valid mail.

Small teams benefit from a hard boundary here. Outsource message transport. Keep domain ownership, policy changes, and the meaning of evidence inside the product's control.

## Build one append-only dispatch record

The implementation can stay narrow. The application renders the content, hashes the artifact, allocates an idempotency key, submits through a generic adapter, and appends observations. It never rewrites history to make the latest state look like the only state.

```ts
import { createHash } from "node:crypto";

type ObservedState = "queued" | "accepted" | "delivered" | "failed" | "unknown";

type EvidenceEvent = {
  at: string;
  kind: "intent-recorded" | "transport-observed";
  state?: ObservedState;
  remoteMessageId?: string;
};

type Dispatch = {
  dispatchId: string;
  authorizedRecipient: string;
  templateVersion: "report-dispatch-v2";
  pdf: Uint8Array;
};

interface TransactionalMail {
  submit(input: {
    idempotencyKey: string;
    from: string;
    to: string;
    subject: string;
    text: string;
    html: string;
    attachment: {
      filename: string;
      contentType: "application/pdf";
      bytes: Uint8Array;
    };
  }): Promise<{ messageId: string; state: "queued" | "accepted" }>;

  lookup(messageId: string): Promise<{ state: ObservedState; observedAt: string }>;
}

interface EvidenceLedger {
  append(dispatchId: string, event: EvidenceEvent): Promise<void>;
}

export async function dispatchReport(
  mail: TransactionalMail,
  ledger: EvidenceLedger,
  job: Dispatch,
) {
  const digest = createHash("sha256").update(job.pdf).digest("hex");
  const idempotencyKey = `report:${job.dispatchId}:${job.templateVersion}:${digest}`;

  await ledger.append(job.dispatchId, {
    at: new Date().toISOString(),
    kind: "intent-recorded",
  });

  const submitted = await mail.submit({
    idempotencyKey,
    from: "Academic Reports <reports@notices.example.edu>",
    to: job.authorizedRecipient,
    subject: "Your requested academic report",
    text: "Your requested academic report is attached.",
    html: "<p>Your requested academic report is attached.</p>",
    attachment: {
      filename: "academic-report.pdf",
      contentType: "application/pdf",
      bytes: job.pdf,
    },
  });

  await ledger.append(job.dispatchId, {
    at: new Date().toISOString(),
    kind: "transport-observed",
    state: submitted.state,
    remoteMessageId: submitted.messageId,
  });

  return { digest, messageId: submitted.messageId };
}
```

The digest and template version belong in the durable dispatch record even though this compact interface returns only the digest and remote identifier. In production, commit the intent before calling the transport. A timeout creates ambiguity: the message may have been accepted even when the caller received no response. Reuse the same idempotency key and reconcile the existing attempt before creating another logical send.

Polling is useful for that reconciliation path. Poll nonterminal records on bounded backoff, append every observed transition, and stop when the transport reports a terminal state. Keep `unknown` honest. An unfamiliar remote value should not silently become `failed`, because an automated resend could duplicate a report.

Don't guess.

Contract tests should cover four cases: ordinary acceptance, a terminal failure, a timeout after remote acceptance, and an unrecognized state. Add a template test with a missing variable and a representative PDF. Those six fixtures cost little to run and target the awkward boundaries where evidence commonly becomes misleading.

## Independence is the second criterion

Evidence is easier to defend when its meaning survives a transport change. Store normalized states alongside the original remote value rather than throwing the original away. Keep template source in version control. Export the message identifier, timestamps, and authentication observations into storage governed by the application's retention policy.

**The transport should report events; it shouldn't define what your business claims those events prove.** This boundary also keeps weekly shipping realistic. A vendor adapter can change without rewriting the authorization rules, evidence vocabulary, or old audit records. The limitation is extra application storage and reconciliation code. This approach is a poor fit when the organization has already standardized on a governed evidence warehouse that ingests transport events and preserves their original semantics; duplicating that control in a small service would create two authorities.

Retention deserves a separate schedule. Reset-request metadata, authentication events, report artifacts, and delivery observations serve different purposes and needn't expire together. Define the purpose and deletion condition for each class. More data is not automatically better evidence.

SMS is a separate recovery channel, not evidence that an email attachment reached its recipient. The WebOTP API can help a web origin receive a specially formatted one-time code from an SMS after user consent, subject to browser support and security constraints documented by MDN. It doesn't replace email-domain authentication, attachment controls, or delivery reconciliation. Keep those claims apart too.

## When should the report move behind a portal?

Choose the runner-up when access must be revocable after notification, when every retrieval requires a fresh authorization decision, or when policy forbids an attachment that can be forwarded. The email then announces availability; the portal records its own authorization and download events.

That stronger control surface isn't free. It adds object access rules, expiration, download handling, session recovery, and support for recipients who can't complete the sign-in flow. For a solo operator, those are feature hours. I would accept that burden when the disclosure policy requires revocation or access-time evidence, not because a portal looks more sophisticated.

For attached reports, the application ledger remains the tighter default. Separate recovery from report dispatch, authenticate the sending domain, preserve the exact template and artifact identity, and state delivery evidence with precision. It is a small system with a defensible boundary. Ship that boundary first.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- MDN, WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
