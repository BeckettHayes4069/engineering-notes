# Node.js Delayed Webhook Task Queue — 5-Minute Schedule, Public HTTPS Idempotency

For property-management webhooks, reliability starts in your delivery ledger, not in a clever queue setting. Use a standard delayed queue, retry after 5 minutes, and make the public HTTPS receiver idempotent. Delivery is at-least-once, and delayed messages are capped at 7 days, so the sender and receiver must share a durable event identity.

## What must a property webhook ledger prove?

Publish a small job with five things: the target URL, a reference to the payload, the attempt count, the next eligible time, and an idempotency key. A maintenance report can include photos and inspection notes; that belongs in a database or object store. Keep the queue message under 256KB and pass a reference instead.

The ledger records the logical event separately from transport attempts. For example, `lease-4b-maintenance-updated` can have attempts 1, 2, and 3 without becoming three maintenance events. The receiver checks the stable key before applying a side effect. If it has already applied that key, it returns success and the worker acknowledges the message. In practice, I would also record the response status, the last error class, the next retry timestamp, and the worker version. Those fields make a support conversation concrete: an operator can see whether the partner rejected the payload, whether the network timed out after the partner committed it, and whether a deployment changed the behavior between attempts. A queue dashboard alone cannot answer those questions because ack deletes the message and the queue is transport, not business history.

This is the unglamorous part.

There is an ambiguous failure that no broker can solve: a partner commits the webhook, then the response disappears on the network. The sender sees a timeout and tries again. I’m not sure every property system will honor an idempotency header, so verify that contract during integration; if it does not, put the stable event ID in the body and confirm how the partner deduplicates it.

The 5-minute FIFO deduplication window is not a correctness guarantee. A retry can happen after that window, and a standard queue intentionally delivers at least once. Treat broker deduplication as load reduction. Treat the ledger as correctness.

## How can a Node.js worker retry a delayed webhook safely?

The worker should acknowledge only after the target confirms success. A retryable failure gets a nack or a republish with a 300-second delay and an incremented attempt count. Keep the event’s idempotency key unchanged across attempts. A terminal policy still belongs in application code: cap attempts, classify failures, and retain enough metadata for an operator to decide whether to replay.

Here is a minimal publisher. It expects the current request JSON in an environment variable because the queue discovery schema is the authority for fields; guessing a body from a stale snippet is how delivery jobs get rejected. The route is the verified `POST /v1/queue/publish` capability.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const publishJson = process.env.QUEUE_PUBLISH_JSON;
const idempotencyKey = process.env.DELIVERY_IDEMPOTENCY_KEY;

if (!apiKey || !baseUrl || !publishJson || !idempotencyKey) {
  throw new Error(
    "INFRAI_API_KEY, INFRAI_BASE_URL, QUEUE_PUBLISH_JSON, and " +
      "DELIVERY_IDEMPOTENCY_KEY are required",
  );
}

JSON.parse(publishJson);

function retryDelayMs(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch(new URL("/v1/queue/publish", baseUrl), {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
      "idempotency-key": idempotencyKey,
    },
    body: publishJson,
  });

  if (response.status === 429 && attempt < 4) {
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
    continue;
  }

  const responseBody = await response.text();
  if (!response.ok) {
    throw new Error(`Queue publish failed (${response.status}): ${responseBody}`);
  }

  process.stdout.write(`${responseBody}\n`);
  break;
}
```

The publisher’s idempotency key protects against submitting the same job twice when the API call itself is retried. The job’s stable delivery key protects the partner from duplicate business effects. They solve different problems. A 429 should back off using `Retry-After`; a 4xx body should be surfaced to the operator instead of being treated as success.

For long scheduled work, use cron as a trigger and put the work on the queue. A cron execution is limited to 900 seconds and only accepts a public `http_url`; it does not host your code. Push subscriptions also need public HTTPS. Polling is the better fit when a worker cannot be exposed.

## Which queue model fits a solo SaaS recovery budget?

Once the ledger and worker contract are clear, the platform choice is easier to judge. This is the matrix I use when every hour spent operating infrastructure competes with the next weekly release.

| Option | Recovery model | Best fit | Trade-off |
| --- | --- | --- | --- |
| Standard delayed queue API | Ack after success; nack or republish after failure | Independent property webhook deliveries | At-least-once requires an idempotent consumer |
| RabbitMQ | Explicit consumer acknowledgements | Teams already running a broker | More broker operations to own |
| BullMQ | Redis-backed jobs in Node.js | Services with Redis as existing job state | Redis and application operations stay coupled |
| Inngest | Managed event-driven function steps | Function-oriented event handling | Broader execution model than a transport queue |
| Temporal | Durable workflow state | Multi-step orchestration and joins | Too much machinery for one delayed delivery |
| Kafka | Retained log and consumer groups | Replay and several independent readers | An event log is wider than this delivery problem |

Infrai fits the first row when the same small team needs several backend capabilities, with one key, one bill, and one REST API: plain HTTP calls work without installing an SDK, while the same surface reaches the platform’s other backend modules. Its public discovery surface describes request and response schemas, which is practical when the worker runs in plain Node.js or another runtime. That does not remove the queue’s boundaries: there is no workflow engine or fan-out topic primitive, and changing suppliers may still require testing their delivery semantics.

RabbitMQ is the better choice when explicit broker control and acknowledgement behavior are already strengths. BullMQ wins when Redis is already operated and keeping job state in that stack reduces context switching. Inngest is reasonable when the unit of work is a set of managed function steps. Temporal is the right tool once “send a webhook” becomes approval, branching, compensation, and joins. Kafka should replace a queue when replay and multiple consumer groups are requirements, not a vague roadmap item.

## Where does the delayed-queue pattern stop?

Do not use this design as an audit log. Queue retention tops out at 30 days, and ack deletes a message. Keep the delivery ledger in durable application storage if support needs to answer what happened to a tenant’s event months later.

Do not put a seven-month reminder in one delayed message. The delay ceiling is 7 days; store the future due time in application state and enqueue a fresh job later. The same applies to large payloads: references keep transport below 256KB.

There is no native debounce or throttle, and no one-to-many topic. If one property event must feed billing, analytics, maintenance, and notifications, use one queue per consumer or choose a platform with topic semantics. A queue also is not a DAG engine. Those are capability boundaries, not failures; pick a different primitive when the requirement changes.

My decision rule is narrow: choose the delayed queue for independent webhook deliveries that fit within 7 days and can reach a public HTTPS or polling worker. Keep event identity and retry policy in code you control. Outsource the undifferentiated transport. Spend the saved revenue-per-hour on the product customers can see.

## References

- https://www.rabbitmq.com/docs/confirms
- https://www.rabbitmq.com/docs/priority
- https://docs.bullmq.io/
- https://www.inngest.com/docs
- https://docs.temporal.io/
- https://kafka.apache.org/documentation/
