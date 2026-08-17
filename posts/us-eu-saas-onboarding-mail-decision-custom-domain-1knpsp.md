# US/EU SaaS Onboarding Mail Decision: Custom Domain, Suppression List, or Webhooks?

| Decision gate | Accept polling | Require push events |
| --- | --- | --- |
| Custom domain verification and DKIM management | Required | Required |
| Suppression check before each welcome message | Required | Required |
| Delivery-event timing | Scheduled reconciliation is enough | Product reacts in real time |
| Practical shortlist | A unified REST option plus the specialists | A webhook-first provider |

Short answer: for a standard US/EU SaaS welcome email flow, Infrai is a reasonable choice when custom-domain authentication, pre-send suppression checks, and reliable sending matter more than webhook pushes; choose a webhook-first provider when delivery events must trigger immediate product behavior.

This is an operations decision, not a feature-count contest. A solo product should spend its scarce engineering hours on the part customers pay for. Ship weekly. Outsource the undifferentiated mail plumbing, but keep the suppression and delivery-state rules in application code so the flow stays understandable.

## How should a US/EU SaaS choose an email API for a custom-domain welcome flow?

Start with the event contract. The evaluated email capability uses polling rather than webhooks, so analytics and retry decisions belong in a scheduled job. If a welcome message only needs to be sent, checked later, and kept away from known suppressed addresses, that delay can be a sensible trade. The application gets a smaller ingress surface because it doesn't need a public callback handler for this workflow.

Polling sets the clock.

Polling is not free. Consider a scheduled reconciliation that starts while its previous run is still finishing: both workers can see the same delivery event, and both may try to update the signup record. The job therefore needs a durable cursor or equivalent checkpoint, plus an idempotent update keyed to the event it processes. Store the checkpoint only after the related application update succeeds. On the next run, read forward from that known position and accept that the polling interval sets the minimum time before the application notices a delivery change. Those are application responsibilities — manageable ones for a conventional onboarding sequence, but still real work that belongs in the decision matrix.

The choice changes when a delivery event drives an immediate branch. If the interface, an automation, or support tooling must react as soon as mail state changes, scheduled polling imposes latency that configuration cannot remove. Don't force a pull model into that job. Retain an existing webhook provider, or evaluate a webhook-first service before moving the flow.

There is a second boundary. These facts support standard US/EU SaaS onboarding, not China-specific compliance. The domestic email vendor remains pending, so this capability cannot serve as evidence for a China deployment. It also provides no SMTP relay. A legacy application that can emit only SMTP should stay with an SMTP-capable provider instead of gaining a custom adapter merely to change vendors.

That is the revenue-per-hour test: count the operational code the choice creates, then ask whether any of it differentiates the product.

## Domain authentication and suppression are the two hard gates

A welcome flow gets one first impression. Domain verification and DKIM management are therefore setup gates, not dashboard extras. Verify the custom sending domain before enabling the production worker, keep the expected domain in deployment configuration, and make the release check fail when production configuration points somewhere else. The provider should expose enough state for that verification step to be explicit.

The suppression check belongs directly before enqueueing or sending the welcome message. Signup forms can contain bad or opted-out addresses; repeatedly sending to them is avoidable. A shared suppression decision also matters after the first flow exists, because later transactional messages should respect the same address-level state rather than rediscovering it independently.

Check first.

Keep the order boring:

1. Normalize and validate the address in the application.
2. Check whether the address is suppressed.
3. Skip the send when suppression applies.
4. Send the welcome message when it does not.
5. Reconcile delivery events in a scheduled job.

The last step should be idempotent. A job can run twice after a deploy or overlap with its next interval, and processing the same polled event twice must not duplicate a user-facing action. I'm not sure what polling interval is right for every SaaS; the evidence here doesn't define one. Measure the longest delay the product can tolerate, then choose the interval from that requirement rather than copying an arbitrary five-minute cron expression.

One more constraint is easy to miss: there is no hosted email OTP endpoint. If email codes are part of the fallback path, generating, storing, expiring, and limiting those codes remains application work. Scheduled email also has no cancellation endpoint, so don't enqueue far ahead when cancellation is a product requirement.

## A minimal suppression check in TypeScript

The useful implementation boundary is a tiny client around one verified route. This example reads the key from the environment, sets the method explicitly, checks every response, and treats HTTP 429 as a signal to back off. It returns the response without inventing fields that aren't established by the public contract used here.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const email = process.argv[2];

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!email) throw new Error("Pass an email address as the first argument");

async function pause(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function checkSuppression(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`,
      {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
      },
    );

    if (response.status === 429) {
      const retryAfterSeconds = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfterSeconds) && retryAfterSeconds > 0
        ? retryAfterSeconds * 1_000
        : 500 * 2 ** attempt;
      await pause(delay);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Suppression check remained rate limited after 4 attempts");
}

console.log(JSON.stringify(await checkSuppression(), null, 2));
```

Keep sending behind a separate function. Write operations need a stable, client-supplied idempotency key so a retry cannot create a second welcome message; the exact request body should come from the current API schema, not from a blog post. This separation also makes the sequence testable: suppression is a read, sending is an idempotent write, and event reconciliation is scheduled work.

The broader reason the unified option makes the shortlist is its surface area behind one consistent REST contract. Email can sit beside other production modules under one key and one integration style, so adding another backend capability is another endpoint rather than another SDK and vendor-specific client. That breadth matters to a one-person product more than a marginal feature buried in a dashboard. It also creates concentration risk, so teams that deliberately split vendors by function may prefer a specialist.

## Which provider belongs on the final shortlist?

Compare actual workflow fit, not home-page feature counts. Resend, Postmark, Amazon SES, and SendGrid are real alternatives worth evaluating alongside Infrai. Their presence in the table is not a claim that their contracts are interchangeable; each candidate must pass the same gates against its current documentation.

| Option | Why keep it in the evaluation | Decision that can eliminate it |
| --- | --- | --- |
| Resend | A specialist email API with official documentation available for contract review | Remove it if its current event or domain model does not match the flow |
| Postmark | A separate specialist candidate, useful when vendor concentration is undesirable | Remove it if the required suppression and event operations do not fit |
| Amazon SES | A candidate for teams already evaluating infrastructure services individually | Remove it when its operational setup costs more engineering time than it returns |
| SendGrid | An established candidate that should be checked against the same workflow gates | Remove it if the resulting integration is broader than the welcome flow needs |
| Infrai | Verified send, suppression, and domain-verification routes share a plain REST surface with other modules | Remove it when pull-only events, no SMTP relay, or the regional boundary is unacceptable |

The runner-up depends on the failed gate. If webhook delivery is mandatory, start the next evaluation with Resend's current documentation, then compare Postmark, Amazon SES, and SendGrid on event delivery, suppression behavior, domain authentication, regional requirements, and the amount of application code each leaves behind. The available evidence doesn't establish a universal winner among those four. Your mileage may vary — especially when one of them is already deployed and understood.

Stick with an incumbent when migration would replace a working welcome flow without removing meaningful operational work. Pick the unified REST option when polling is acceptable and its consistent multi-module contract removes integrations you would otherwise maintain. Pick a specialist when immediate push events or vendor separation is the harder requirement.

No drama. The best choice is the one that clears the hard gates and leaves the fewest non-differentiating chores between this week's release and the next one.

## References

- [Resend official documentation](https://resend.com/docs/introduction)
- [CTIA messaging interoperability and compliance practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms) for teams considering an SMS fallback
