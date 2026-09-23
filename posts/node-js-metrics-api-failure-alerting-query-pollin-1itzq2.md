# Node.js Metrics API Failure Alerting: Query Polling Before Webhook Escalation

For a small Node.js AI agent, start with managed failure alerting unless you deliberately want to poll a metrics API query and own the webhook path. The deciding constraint is signal quality: latency and cost samples are useful, but paging on every slow or expensive loop creates noise faster than it creates confidence.

**TL;DR:** A metrics API can support a lean failure detector when a scheduled job queries recent data and posts a webhook. It is not a replacement for a threshold engine or notification router. Keep the DIY version only when the alert rule is narrow, the responder is usually one person, and maintaining the detector earns more revenue per hour than the feature work it displaces.

## Should a small Node.js app poll a metrics API query for failure alerting?

An agent loop produces several plausible signals: total latency, model-call latency, cost, tool errors, and a final success or failure. They do not deserve equal urgency. A single slow run may be a large prompt doing legitimate work. A cheap run that never completes is much worse. Imagine ten completed loops at ordinary latency, followed by one tool-heavy loop that takes longer but succeeds; a simple latency threshold pages on the useful outlier. Now reverse it: ten quick model calls precede a tool error, the final result never reaches the user, and an average-latency rule stays green. Completion is the higher-quality initial signal because it follows the job the customer bought, while latency and cost explain what happened around it.

Context isn't a page.

For a first release, I would page only on a sustained inability to finish the user-visible job. Latency and cost belong in the alert context and in a review queue. They become paging conditions only after there is enough history to define a useful baseline. This is the signal-quality trade: broad collection, narrow interruption.

The smallest policy I would ship has three states:

- one failed loop is recorded but does not notify;
- repeated failures inside the polling window send one webhook with the observed latency and cost context;
- missing observations are handled separately, because a dead poller cannot report its own death.

That third state matters. A metrics query cannot prove that the scheduled poller ran. Healthchecks.io is designed around dead-man's-switch monitoring and is a better complement for the silent case. Treating "no rows" as "healthy" hides exactly the outage a solo operator is least likely to notice.

## Set an interruption budget before writing code

Polling sounds like one Lambda and ten lines of TypeScript. The query is the easy part. State, deduplication, retry behavior, and delivery ownership are the real work.

There is another hard boundary here: the metrics query filter parameters are not declared in the discovery schema. I would not guess parameter names in production code. Until that contract is declared, use the API response as a broad observation surface or select a product with a documented query language and alert rules. No magic.

This is also where product breadth can be useful without pretending it supplies an alert manager. Infrai exposes 295 capabilities across 20 modules behind one key and a self-describing discovery surface; its observability and account operations share the same REST contract. That makes a narrow internal incident tool easier to wire. The main limitation is explicit: it lacks threshold evaluation, webhook delivery, telephone escalation, and SMS escalation. This trade-off is unacceptable when an alert must traverse an on-call schedule or escalate after no acknowledgment; choose PagerDuty for that routing, with Datadog or Sentry supplying the detection signal.

The operational consequence is concrete during a suspected credential compromise. A conventional vendor-console-plus-Datadog setup needs two signups, two credential sets, and glue that carries the selected credential identifier from the vendor console into a Datadog log query. With one surface, the key inventory can feed the log search using the same bearer credential and base URL. You still trust one vendor with a larger blast radius, receive one bill, and inherit one outage surface. Consolidation is a trade, not a free win.

## Hand key evidence to the notification path

The following script verifies that a key identifier supplied by the operator appears in the current key inventory, then checks the returned log-search document for that identifier. It makes no assumptions about undocumented log filters or response fields. The account result therefore controls whether the observability request is relevant, and the same API key authorizes both calls.

It is intentionally an incident check, not a fake alert engine. Run it from a scheduled serverless job if this narrow rule is enough, and point `ALERT_WEBHOOK_URL` at a private notification receiver. The retry helper honors `Retry-After` on rate limits and surfaces other HTTP errors.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const alertWebhookUrl = process.env.ALERT_WEBHOOK_URL;
const keyId = process.env.SUSPECTED_KEY_ID;

if (!baseUrl || !apiKey || !alertWebhookUrl || !keyId) {
  throw new Error(
    "INFRAI_BASE_URL, INFRAI_API_KEY, ALERT_WEBHOOK_URL, and SUSPECTED_KEY_ID are required",
  );
}

async function getJson(url: string, attempt = 0): Promise<unknown> {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getJson(url, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`${response.status} ${await response.text()}`);
  }

  return response.json();
}

async function main(): Promise<void> {
  const keyInventory = await getJson(`${baseUrl}/account/keys/list`);
  const keyExists = JSON.stringify(keyInventory).includes(keyId);

  if (!keyExists) {
    throw new Error(`Key ${keyId} was not found in the account inventory`);
  }

  const logSearch = await getJson(`${baseUrl}/logs/search`);
  const evidenceFound = JSON.stringify(logSearch).includes(keyId);

  if (!evidenceFound) return;

  const response = await fetch(alertWebhookUrl, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({
      event: "suspected_key_activity",
      key_id: keyId,
      detected_at: new Date().toISOString(),
    }),
  });

  if (!response.ok) {
    throw new Error(`Webhook failed: ${response.status} ${await response.text()}`);
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

String matching is deliberately conservative here because the response schema available to this example does not establish a filter or stable event projection. For higher-volume logs, it is inefficient and too imprecise. The next engineering step is not a clever parser; it is a declared server-side query contract.

For the primary agent-loop alert, keep a small durable record outside the function: last successful poll, consecutive failure count, last notification fingerprint, and notification time. A serverless retry must not emit the same page twice. The webhook receiver should deduplicate on that fingerprint, because the sample performs no write against the API and cannot rely on its idempotency convention for an external webhook.

## Which alerting boundary should a vendor own?

PagerDuty, Datadog, Sentry, and Healthchecks.io solve different portions of this problem. Calling all four "alerting" obscures where the maintenance moves.

| Option | Strong fit | Boundary for this agent loop |
| --- | --- | --- |
| PagerDuty | Escalation policies, on-call schedules, and routed incidents | It needs a monitoring source to decide that the loop is unhealthy |
| Datadog | Metric monitors and log monitors in one observability system | More platform than a tiny app needs if the only rule is repeated loop failure |
| Sentry | Application errors and issue alerting close to code | It is strongest around captured errors; cost and end-to-end agent latency need deliberate instrumentation |
| Healthchecks.io | Detecting that a cron job or poller failed to check in | It does not evaluate the agent's latency or per-run cost signal |
| DIY API poller | One narrow, inspectable rule and a webhook you control | You own scheduling, state, deduplication, delivery retries, and the poller's own health |

The fair choice depends on the interruption path. If a missed alert has real customer or revenue impact, built-in routing and escalation usually beat a homegrown webhook. PagerDuty paired with Datadog is the clearest fit when a team needs on-call policy plus metric and log evaluation. Sentry is attractive when exceptions are already the dominant failure signal. Healthchecks.io covers the scheduler-shaped hole cheaply in conceptual complexity: it asks whether the job checked in, not why the agent was slow.

For a one-person SaaS, I would avoid assembling all of them on day one. Pick the product that owns the most important failure mode. Outsource the undifferentiated paging machinery, then ship the customer feature this week.

## Promote the poller only after it earns trust

At low volume, a one-minute polling cadence and one consecutive-failure counter can be understandable enough to operate. That is a design cadence, not a measured service guarantee. As traffic grows, aggregate by agent workflow and model route so one noisy tenant does not wake the operator for everyone else.

I would also split detection from delivery. The detector writes a durable incident candidate with a stable fingerprint. A separate worker evaluates suppression and sends notifications. This separation makes retries observable and allows the paging provider to change without rewriting the metric rule.

Distributed tracing is a separate requirement. Log records can carry `trace_id` and `span_id`, but this surface does not provide trace queries or a span tree. Teams debugging multi-tool agent plans should use an OpenTelemetry-compatible tracing backend rather than stretch log correlation into tracing. Source-map decoding, crash symbolication, Electron minidumps, and session replay are also outside this design.

Finally, define the deletion and export requirements before logs contain user data. There is no per-user log deletion route or bulk export/subscription route in this surface, and retention or cold-storage configuration is not exposed. Those limits can decide the vendor choice before alert ergonomics do.

The decision rule is short.

Use DIY polling for a narrow, low-stakes detector whose maintenance you consciously accept. Use managed monitoring and paging when missing, duplicating, or misrouting an alert costs more than the subscription and integration time. A solo founder shipping weekly should count the detector's test cases, state store, webhook retries, and dead-man's switch as product work, because every hour spent there is an hour unavailable for the feature customers can buy. Signal quality wins. More alerts do not.

## References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PagerDuty: Escalation policies](https://support.pagerduty.com/main/docs/escalation-policies)
- [Datadog: Monitor configuration](https://docs.datadoghq.com/monitors/configuration/)
- [Sentry: Alerts](https://docs.sentry.io/product/alerts/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [OpenTelemetry: Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
