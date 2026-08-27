# A Recovery-First Test for DKIM Rotation, Polling Events, and SaaS Compliance

Short answer: choose an email deliverability platform by running one recovery drill across domain verification, DKIM rotation, suppression, polling, and EU/US data handling; don't choose it from a send-API feature grid.

The useful comparison unit is not a message sent under ideal conditions. It is a message whose evidence survives a missed callback, a DNS change, a known-bad recipient, and a regional audit question. For a one-person SaaS, that distinction matters because every afternoon spent reconciling mail state is an afternoon not spent shipping the next weekly release. Outsource transport. Keep the evidence chain.

## How should an EU/US SaaS test email API polling events and compliance?

Start with a disposable subdomain and a written acceptance script. The script should create a sending identity, observe domain ownership and signing readiness as separate states, send to a normal test recipient, suppress another recipient, interrupt event intake, and then recover the missing interval through polling or replay. Run the same script for each region in scope. Record payload shapes and timestamps, but use synthetic addresses rather than customer data.

This is a build test, not a demo.

Domain ownership and DKIM readiness deserve separate assertions. Publishing the requested DNS record proves control only after the platform observes it; readiness to sign is another operational state. For rotation, publish a second selector, wait until its state is observable, switch signing according to the platform's supported procedure, and keep the prior selector through the planned overlap. DNS caching means there is no honest universal overlap duration. I'm not sure a fixed number would help anyway; the decision should come from observed DNS state, documented signing behavior, and the rollback window your team accepts.

Yahoo's sender guidance is a useful external floor for the authentication part of the fixture. It says all senders should authenticate with SPF or DKIM, while bulk senders must use both SPF and DKIM and publish a DMARC policy. It also calls for valid forward and reverse DNS, low spam complaint rates, and easy unsubscribe support. Those controls don't guarantee inbox placement. They do turn several vague promises into checks that can fail before a customer-facing migration.

For events, disconnect the push consumer on purpose. Generate an accepted message, a delivery outcome, and a suppression-related outcome, then restore the consumer and use the platform's documented polling or replay mechanism. The local ledger should converge without duplicate business transitions. If the platform cannot provide a stable event identifier, cursor, or another documented way to delimit the recovery interval, price the custom reconciliation work into the decision. A webhook is a latency path. It shouldn't be the only record.

Finally, make “EU support” concrete. Ask where recipient addresses, message bodies, templates, event payloads, logs, backups, and support-access data are processed and retained. Ask how deletion works and what crosses regions. The answer belongs beside the technical test results, not in a separate legal folder nobody opens during implementation. EU and US traffic can share one application contract while using distinct credentials, queues, and evidence stores when the selected data-handling model requires that separation.

## The constraint is recovery time, not send syntax

Most candidates can accept a request containing a sender, recipient, subject, and body. That happy path reveals little. The expensive work begins when a customer asks why the product says “sent” while no message appears, or when a callback was unavailable during deployment. “Accepted by the API” is not “delivered,” and neither state means “placed in the inbox.” A useful system preserves those distinctions.

I evaluate the choice through revenue per engineering hour. The concrete constraint is a support question that must be answerable from one internal timeline, without manually correlating three dashboards. That pushes the application toward a small delivery ledger with message intent, transport acceptance, later delivery events, and suppression decisions stored as different facts. It also changes procurement: event recovery and state semantics become release blockers, while a polished template editor becomes optional.

Run that support question as a timed exercise before signing a contract. Give the operator only an internal message ID and remove access to the candidate's dashboard. The ledger should first show when the product committed the intent and which idempotency key crossed the transport boundary. It should then distinguish API acceptance from the later recipient event, preserve the suppression check that ran before submission, and link every transition to a raw event or an explicit local decision. Halfway through the exercise, stop the event consumer, create another synthetic message, restart the worker, and reconcile from the last committed cursor. Replaying the final page must not send the message again or move its state backward. Then ask the operator to identify the region that processed the address, the retention rule covering the raw payload, and the deletion path. This one drill puts the API contract, event model, operational evidence, and compliance claims under the same clock. A candidate can have excellent sending infrastructure and still be a poor fit if answering that ordinary support question requires an undocumented join between dashboard exports. The test doesn't prove inbox placement. It proves that the SaaS can explain what its own system observed, recover what it missed, and avoid inventing certainty where the evidence ends.

Make recovery boring.

The comparison worksheet can stay short:

| Surface | Evidence to capture | Walk-away condition |
| --- | --- | --- |
| Domain | Separate ownership and signing states | One ambiguous status with no machine-readable detail |
| DKIM rotation | Observable old/new selector overlap | A cutover that cannot be staged or verified |
| Suppression | Scope, reason, and timestamp | A silent non-send with no queryable decision |
| Events | Signed push plus polling or replay | A missed callback creates a permanent evidence gap |
| Regions | Processing, retention, deletion, and access terms by data category | “EU available” with no operational definition |
| Templates | Versioned source and deterministic fixtures | Production edits are the only retained copy |

Error handling belongs in the fixture too. Inject a `401` by using a deliberately invalid test credential and confirm that it is classified as an authentication failure, not retried forever. Feed the consumer the same event twice and confirm one state transition. Feed it an unknown event type and confirm the raw event remains available for later interpretation. These aren't dramatic scenarios — that's why they are worth automating. They catch the small contract mismatches that otherwise consume a Thursday afternoon.

## The smallest boundary worth owning

The application should own a narrow mail contract and a provider adapter. This isn't an attempt to erase every platform difference. It is a way to keep billing, onboarding, and account workflows independent from transport syntax while leaving specialized fields available at the edge.

```ts
type DomainState = "pending" | "ready" | "action_required";
type DeliveryState =
  | "accepted"
  | "delivered"
  | "bounced"
  | "complained"
  | "suppressed";

interface DeliveryEvent {
  eventId: string;
  messageId: string;
  recipient: string;
  state: DeliveryState;
  occurredAt: string;
  raw: unknown;
}

interface MailTransport {
  getDomainState(domain: string): Promise<DomainState>;
  isSuppressed(recipient: string): Promise<boolean>;
  send(input: {
    from: string;
    to: string;
    subject: string;
    html: string;
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
  pollEvents(cursor?: string): Promise<{
    events: DeliveryEvent[];
    nextCursor?: string;
  }>;
}

interface EventLedger {
  has(eventId: string): Promise<boolean>;
  append(event: DeliveryEvent): Promise<void>;
  commitCursor(cursor: string): Promise<void>;
}

async function reconcile(
  transport: MailTransport,
  ledger: EventLedger,
  cursor?: string,
): Promise<string | undefined> {
  const page = await transport.pollEvents(cursor);

  for (const event of page.events) {
    if (!await ledger.has(event.eventId)) {
      await ledger.append(event);
    }
  }

  if (page.nextCursor) {
    await ledger.commitCursor(page.nextCursor);
  }

  return page.nextCursor;
}
```

The important ordering is mundane: apply the page, then commit its cursor. A crash before the cursor commit causes replay, which the event ID makes harmless; advancing first could lose the page. Verified push events should enter the same append path so polling is a repair loop rather than a second state machine. Signature verification, allowed timestamp skew, and acknowledgement behavior must follow the selected platform's documented contract, so they belong in the adapter and its contract tests.

Keep template source in the repository as well. Mustache's documented variables, sections, inverted sections, partials, and escaping rules make it possible to build deterministic fixtures without binding product copy to a dashboard. Render representative data in CI, including missing optional values and text containing markup characters, and inspect both HTML and plain-text output. A hosted editor may still serve nontechnical collaborators, but it shouldn't be the sole copy of revenue-critical messages.

There is one deliberate leak in this abstraction: retain the original event payload under a bounded policy. The normalized state supports product logic, while the raw record supports audits, new event types, and correlation. Retention should be explicit because those payloads can contain recipient data. Short enough to respect the data policy. Long enough to investigate the incidents the policy is meant to cover. Your mileage may vary.

## What changes when volume and regions grow?

At low volume, one worker can drain an outbox, send mail, ingest verified events, and run a scheduled reconciliation poll. That is often the right starting point for a weekly shipping cadence. The first scale change should follow a measured failure domain, not an architecture diagram: separate the send worker when queue age affects user actions; separate event ingestion when callback bursts threaten acknowledgement time; separate regional stores when the documented data model calls for it.

The outbox matters because a product database commit and a send request are two systems. Store message intent in the same transaction as the product action, then let a worker deliver it with an idempotency key. Incoming events append before they update the customer-visible projection. Unknown event types stay recorded rather than being coerced into a familiar status. This gives deployments room to add interpretation after capture.

Monitor queue age, oldest unreconciled event, cursor progress, duplicate rate, unknown event types, suppression decisions, domain-state changes, and failed correlations. Aggregate delivery percentages alone cannot explain one customer's message. A daily synthetic message through every active regional path can test authentication, event correlation, and ledger convergence without using customer content, though its exact cadence should follow traffic and risk rather than habit.

DKIM rotation becomes a state machine at this point: request or generate the next selector through the chosen platform's documented process, publish its DNS record, observe readiness, change signing, preserve the previous record for the approved overlap, and retire it after verification. Don't let a calendar reminder be the only orchestration. Store who approved each transition and the evidence observed at that step.

## Trade-offs I would accept

Normalization costs schema work and can hide useful platform-specific detail. I would accept that cost for the core send and event lifecycle, then expose optional metadata rather than pretending every analytics field is portable. The catch is clear: adopting a specialized capability still requires adapter work and a deliberate product decision.

Polling also spends API quota and finds gaps later than healthy push delivery. Use it for scheduled reconciliation, not constant busywork. A push-only design is not suitable when losing one callback can leave customer-visible state wrong forever; a polling-only design is a poor fit when low-latency delivery state drives the user experience. The combination earns its complexity only when both paths converge through one idempotent ledger.

This approach is too heavy for a disposable internal notification where basic acceptance logs are enough and no customer relies on delivery state. At the other end, a regulated company with dedicated messaging and compliance teams may need formal audit exports, policy enforcement, legal review, and region-specific infrastructure beyond this thin adapter. A team operating its own mail transfer agents needs direct controls for queues, feedback loops, abuse response, and reputation; an API-shaped comparison does not cover that operating model.

The final decision is therefore a recovery budget. Can the system rotate signing identity without an abrupt cliff, stop mail to a suppressed recipient, reconstruct missed events, explain a single message, and answer where each data category went? If those answers are observable and testable, the undifferentiated transport can stay outsourced and the next weekly release can stay on schedule.

## References

- https://senders.yahooinc.com/best-practices/
- https://mustache.github.io/mustache.5.html

## Further reading

- Yahoo sender best practices and requirements: https://senders.yahooinc.com/best-practices/
- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
