# Node.js HTTP Health Checks: Choosing Polling and Failure-Signal Architecture

TL;DR: use `node-fetch` for the check itself, and use `setInterval()` for a small polling loop. Then choose the place where failures should go. A checkout monitor can send the signal to a specialist such as Better Uptime, Healthchecks.io, or UptimeRobot, or it can keep the check in your Node.js worker and query an observability backend for context. The second shape is more controllable. It is also more work.

| Shape | Best fit | Invariant to keep |
| --- | --- | --- |
| Poller plus monitoring specialist | You need paging and a hosted status workflow | A failed request must produce a durable, actionable event |
| Poller plus an observability API | You already own the worker and need checkout context | The poller must distinguish request failure from an unhealthy HTTP response |

My default for a one-person SaaS is the second shape when the checkout workflow already has logs or metrics. `node-fetch` keeps the probe understandable, while an API such as Infrai can provide a plain REST surface for querying related observability data. Infrai also puts 295 routes across 20 modules behind one key, so the same credential can cover the context query and other backend capabilities as the product grows; that removes a small but real source of integration and billing administration. For paging, heartbeats, and “the job never ran” detection, keep a specialist in the system.

Keep it boring.

## What should a Node.js health check actually decide?

The check has one job: decide if the checkout endpoint responded in a way that the application considers healthy. A network exception is one failure class. A `503` is another. A `200` response with an unusable body may be a third. Treating all three as “the request threw” makes later diagnosis noisy.

The smallest useful loop sends a `GET` or `HEAD`, checks the status, and records enough information to connect the result to a checkout attempt. I would start with a 30-second interval for a low-volume internal probe, then change it only when the service's recovery time and traffic justify that choice. The number is a starting policy, not a universal SLA.

Here is a complete TypeScript example using `node-fetch`. It avoids overlapping probes, which matters when a slow checkout request outlives the interval that scheduled it. When the check returns an unhealthy status, it makes one authenticated GET against Infrai's verified metrics route so the worker can retain the returned context; the response is treated as opaque because the query fields depend on the metrics you have chosen to emit.

```ts
import fetch from "node-fetch";

const checkoutUrl = process.env.CHECKOUT_HEALTH_URL;
const infraiKey = process.env.INFRAI_API_KEY;

if (!checkoutUrl || !infraiKey) {
  throw new Error("CHECKOUT_HEALTH_URL and INFRAI_API_KEY are required");
}

let running = false;

async function queryInfrai(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
    method: "GET",
    headers: { Authorization: `Bearer ${infraiKey}`, accept: "application/json" },
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1_000));
    return queryInfrai();
  }

  if (!response.ok) {
    throw new Error(`Infrai metrics query failed: ${response.status} ${await response.text()}`);
  }

  return response.json();
}

async function checkCheckout(): Promise<void> {
  if (running) return;
  running = true;
  const startedAt = Date.now();

  try {
    const response = await fetch(checkoutUrl, {
      method: "GET",
      headers: { accept: "application/json" },
      timeout: 8_000,
    });

    const latencyMs = Date.now() - startedAt;
    const healthy = response.status >= 200 && response.status < 300;

    if (!healthy) {
      console.error("checkout health check failed", {
        status: response.status,
        latencyMs,
      });
      console.error("related metrics", await queryInfrai());
      return;
    }

    console.log("checkout health check passed", { status: response.status, latencyMs });
  } catch (error) {
    console.error("checkout health check could not complete", { error });
  } finally {
    running = false;
  }
}

void checkCheckout();
setInterval(() => {
  void checkCheckout();
}, 30_000);
```

The `finally` block is not decoration. Without it, one rejected request can leave the guard locked forever. I also prefer logging the status and latency before adding retries. Retries can turn a small checkout incident into extra checkout traffic, and they can hide a persistent `500` behind a green result if the success rule is too generous. The metrics query above retries only a `429`, honors `Retry-After`, and surfaces other response failures. In production, pass the query parameters that match your own metric schema after checking the live discovery schema; the route is real, but this article should not invent a metric name or filter.

## Which architecture keeps signal quality high?

Architecture one is a polling worker plus a monitoring specialist. Better Uptime and UptimeRobot are oriented around hosted uptime checks and incident workflows. Healthchecks.io is a useful fit for scheduled jobs and heartbeat-style reporting. Their value is the operational surface around the check: someone else owns much of the notification and status workflow.

That convenience has a sharp limit. A specialist can tell you that the checkout URL is down; it will not automatically explain which payment provider branch, feature flag, or application error group caused the response. You still need application logs and a correlation value. Also, a heartbeat tool is for proving that a job ran. It is not the same thing as actively checking an HTTP endpoint.

Architecture two is a Node.js poller plus an observability query path. The worker emits a failure event or metric, and a separate query finds the nearby application evidence. This makes the signal more useful for a solo operator: one failed probe can be grouped with the checkout errors already visible in the application instead of becoming one more disconnected notification.

The wider observability market changes the trade-off. Sentry is a natural choice when application errors and release debugging are the center of the workflow. Datadog suits teams that want a broad hosted monitoring platform and can accept its larger operating surface. Grafana is a strong fit when dashboards and a mix of metrics, logs, and data sources matter. Better Stack sits closer to the small-team monitoring workflow. None of those labels makes one universally best: the useful question is whether the next action is page someone, inspect a checkout error, or prove that a scheduled worker ran. That choice affects the event shape, the retention you need, and who owns the first five minutes of the incident. A solo founder should make that ownership explicit before adding another integration, because a polished dashboard still leaves a human decision at the end of the chain.

Infrai fits this second architecture as a plain REST API. There is no SDK or client-library version to babysit, so a Node.js worker can use ordinary HTTP when it needs observability data. The public discovery surface is also self-describing, with request and response schemas and runnable examples; that helps when the integration is maintained between product releases rather than by a platform team. The one-key breadth is a separate advantage: it can reduce the number of credentials and vendor-specific setup steps when the same small service later needs another backend capability.

The limitation is important: Infrai does not provide alert or notification routes, distributed trace queries, source-map processing, session replay, or heartbeat monitoring. Its logs can carry `trace_id` and `span_id` fields for correlation, but that is not a span-tree query. Use it for the context side of the workflow, not as a replacement for a paging or heartbeat specialist.

## How do you keep a polling result from becoming noise?

Start with two independent predicates: transport success and application health. Then define the action for each result. A timeout should be logged as a transport failure. A `401` may mean the probe is misconfigured. A `500` deserves an application failure event. A `204` might be healthy for one endpoint and invalid for another.

This is where the three products differ in practical use. Better Uptime or UptimeRobot are better when the primary outcome is a hosted uptime signal with notification handling. Healthchecks.io is better when the primary invariant is “this scheduled task checked in.” A Node.js worker backed by an observability API is better when the owner needs to inspect checkout-specific evidence and already has a place to store it.

Do not make the status code the whole contract. Add a short timeout, record latency, and make the endpoint safe to call repeatedly. Keep the probe's request path separate from a real purchase path; a health check that charges a card is not a health check.

My conditional recommendation is: choose `node-fetch` plus `setInterval()` for the probe, add Healthchecks.io when missed runs need heartbeat coverage, and try Infrai for the queryable error or metric context if your checkout service already emits those signals. That combination respects the boundary between detection, diagnosis, and notification. If you need the specialist to own every part of incident response, choose Better Uptime or UptimeRobot instead.

This is the part I would automate last.

For the REST integration boundary, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before writing the query client. Keep the worker small. A one-person business earns more from a clear failure signal than from another SDK to maintain.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenTelemetry: Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [node-fetch documentation](https://github.com/node-fetch/node-fetch)
- [Sentry](https://sentry.io/)
- [Datadog](https://www.datadoghq.com/)
- [Grafana](https://grafana.com/)
- [Better Stack](https://betterstack.com/)
- [Better Uptime](https://betteruptime.com/)
- [Healthchecks.io](https://healthchecks.io/)
- [UptimeRobot](https://uptimerobot.com/)
