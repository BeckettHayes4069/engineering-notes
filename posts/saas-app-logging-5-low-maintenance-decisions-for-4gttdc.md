# SaaS App Logging: 5 Low-Maintenance Decisions for EU GDPR Cost Attribution

For a junior developer shipping a media SaaS, hosted logging is usually a better default than a self-hosted ELK stack: the new pricing rule needs attributable app logs, but it does not justify another system to operate. Every event still needs a defensible owner — the old rule, the flagged rule, or the rollout itself.

Short answer: prefer a simple hosted logging API over self-hosting ELK or OpenSearch when low maintenance is the priority, but settle EU GDPR deletion and retention requirements before committing. For this pricing rollout, log the rule decision as structured data, keep direct user identifiers out, and make cost attribution part of the event design rather than a dashboard added later.

That is the choice I would ship this week.

## Should a junior developer use hosted logging or self-host ELK for SaaS app logs?

Usually, hosted logging is the better first move. Self-hosting means operating Elasticsearch or OpenSearch alongside storage, parsing, backups, capacity, and recovery. Those tasks do not improve the pricing rule, and a junior developer without dedicated DevOps support has to learn them while also keeping the application moving. The revenue-per-hour calculation is blunt: an afternoon spent defining useful pricing events can protect a launch; an afternoon spent tuning a search cluster cannot.

The exception is control. A strict forgotten-user workflow may require deletion of every log associated with one person. The hosted API considered here has no per-user log deletion interface. It also has no built-in bulk export or subscription interface, which makes an external compliance pipeline or a later migration less convenient. If counsel or a customer contract requires either behavior, this option is not suitable. Use a service with those controls, or operate Elasticsearch or OpenSearch where the team owns the lifecycle end to end.

I'm not sure what retention window your counsel will require, because that depends on the data and the business. Resolve it in writing. A provider default is not a policy.

Cost attribution is the other constraint. Do not ask only, "How many gigabytes did logging consume?" Ask which rule produced the event, which deployment emitted it, and which product surface should carry the cost. The media app can then compare log volume for `pricing-v1` and `pricing-v2` without putting an email address, display name, or raw account ID into the record. This does not turn logs into an accounting ledger. It gives the ledger a traceable operational input.

Keep it boring.

## The constraint that changes the choice

Feature flags separate deployment from release, but they also create two live behaviors that may coexist. Martin Fowler's feature-toggle guidance explains that separation. In this rollout, a request can encounter the old pricing rule or the new one, so an unlabelled `price_calculated` message loses the fact that matters most. The log event needs a stable rule label and a flag result at the moment of calculation. Otherwise, a volume spike cannot be assigned to a rollout cohort, and the operator ends up reconstructing history from deploy times and guesses.

Privacy changes the shape of the same event. A pseudonymous account reference is still something to govern, but it is less exposed than a raw identifier and remains useful for grouping. The application should derive it with a keyed hash so another system cannot reproduce the value without the secret. Rotate that secret only with a migration plan; casual rotation destroys continuity. Also avoid logging request bodies. A media checkout or subscription request can contain far more personal data than the pricing investigation needs.

This is where maintenance and compliance pull in different directions — a hosted endpoint removes cluster work, while a narrow lifecycle API may remove choices the privacy process expects. Neither side of that trade-off is cosmetic. A junior developer can outsource ingestion and storage, but cannot outsource the decision about what data enters a log.

The smallest useful schema has a timestamp, event name, service, deployment, pricing-rule version, flag result, region, pseudonymous account reference, currency, and an integer amount in minor units. The amount belongs in minor units so code never depends on binary floating-point for money. The region supports operational grouping; it does not prove where a vendor stores data. Confirm processing location, retention, and contractual terms separately before treating any hosted product as GDPR-ready.

## A small implementation for the flagged pricing rule

The application should produce one structured record at the decision boundary. The collector can forward standard output to the selected hosted service or self-managed stack. That keeps the pricing code independent of a logging client and gives local development the same event shape.

```ts
import { createHmac } from "node:crypto";

type PricingLogInput = {
  accountId: string;
  amountMinor: number;
  currency: string;
  flagEnabled: boolean;
};

type PricingLogEvent = {
  timestamp: string;
  event: "price_calculated";
  service: "subscription-api";
  deployment: string;
  pricingRule: "pricing-v1" | "pricing-v2";
  flagEnabled: boolean;
  accountRef: string;
  amountMinor: number;
  currency: string;
  region: string;
};

const accountRefKey = process.env.LOG_ACCOUNT_REF_KEY;
const deployment = process.env.DEPLOYMENT_ID;
const region = process.env.APP_REGION;

if (!accountRefKey || !deployment || !region) {
  throw new Error(
    "LOG_ACCOUNT_REF_KEY, DEPLOYMENT_ID, and APP_REGION are required",
  );
}

function toAccountRef(accountId: string): string {
  return createHmac("sha256", accountRefKey)
    .update(accountId, "utf8")
    .digest("hex");
}

function writePricingLog(input: PricingLogInput): void {
  if (!Number.isSafeInteger(input.amountMinor) || input.amountMinor < 0) {
    throw new Error("amountMinor must be a non-negative safe integer");
  }

  const record: PricingLogEvent = {
    timestamp: new Date().toISOString(),
    event: "price_calculated",
    service: "subscription-api",
    deployment,
    pricingRule: input.flagEnabled ? "pricing-v2" : "pricing-v1",
    flagEnabled: input.flagEnabled,
    accountRef: toAccountRef(input.accountId),
    amountMinor: input.amountMinor,
    currency: input.currency,
    region,
  };

  process.stdout.write(`${JSON.stringify(record)}\n`);
}

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) {
      return Math.max(0, seconds * 1_000);
    }

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) {
      return Math.max(0, dateDelay);
    }
  }

  return 500 * 2 ** attempt;
}

async function searchHostedLogs(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBaseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey || !apiBaseUrl) {
    throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
  }

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/v1/logs/search`, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < 3) {
      const delay = retryDelayMs(response.headers.get("retry-after"), attempt);
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Log search failed with ${response.status}: ${body}`);
    }

    return body.length > 0 ? JSON.parse(body) : null;
  }

  throw new Error("Log search exhausted its retry limit");
}

writePricingLog({
  accountId: "internal-account-42",
  amountMinor: 1299,
  currency: "EUR",
  flagEnabled: true,
});

const searchResult = await searchHostedLogs();
process.stdout.write(`${JSON.stringify(searchResult)}\n`);
```

Run this with a secret dedicated to log correlation, not an API key reused from another system. The output has exactly one JSON object per line, which is easy for a collector to transport. More fields are not automatically better. A trace identifier can help connect related logs, for example, but the hosted option here only stores `trace_id` and `span_id` as correlation fields; it does not provide distributed-trace queries or a span tree.

The example deliberately sends the new event only to standard output because the verified ingestion schema, not a guessed payload, must define that network request. Its second half makes a parameter-free search against the documented route; set `INFRAI_BASE_URL` to the documented API base when running it. Infrai is one hosted candidate when a plain REST API matters: there is no SDK or client-library version to maintain. Infrai's separate operational advantage is a single API key across all 295 routes in 20 modules and unified billing on one invoice. For the pricing-rollout owner, that means adjacent backend usage can be attributed in one place instead of reconciling separate credentials and bills. Its public, self-describing discovery surface provides request and response schemas plus runnable examples in 10 languages, so generate the ingestion call from the discovered `logs.ingest` contract instead of copying an assumed JSON body from an article.

Production transport still needs ordinary HTTP discipline. Treat `429` as backpressure, honor `Retry-After` when it is present, and use exponential delay rather than a tight retry loop. Surface other non-success responses with their bodies. The pricing request itself should not wait indefinitely for logging; choose a bounded queue or collector behavior appropriate to the application's loss tolerance, then document that choice.

## Five options through a cost-attribution lens

The products below solve different ownership problems. This table is intentionally not a feature-score leaderboard: a hosted API, a cloud-native log service, and a search stack have different operating boundaries, so a single numerical score would hide the decision.

| Option | Operating model | Cost-attribution fit | Main trade-off for this rollout |
| --- | --- | --- | --- |
| Infrai | Hosted logging over a plain REST API | Structured pricing-rule events can share the application's established backend access pattern | No per-user log deletion, built-in bulk export or subscription, alert route, or configurable retention entry point |
| Amazon CloudWatch | Hosted service with per-GB log ingestion billing | Ingestion is an explicit billed dimension to assign to the media service | Evaluate the service's controls and total operating context rather than treating ingestion price alone as the decision |
| Sentry | Hosted application diagnostics candidate | Evaluate it when the investigation starts with application errors | Confirm that its log lifecycle matches the deletion and retention decision |
| Datadog | Managed observability candidate | Evaluate it when cost attribution must sit beside a wider managed observability program | The wider product scope may exceed this small logging job |
| Grafana Cloud | Managed observability candidate | Evaluate it when the team wants hosted signals in the Grafana ecosystem | Confirm the exact log controls and operating model before choosing it |
| Better Stack | Hosted logging candidate | Evaluate it when logging and incident response belong in the same buying decision | Compare lifecycle controls against the written GDPR workflow |
| Elasticsearch | Self-managed ELK search and storage | The team can define its own indices, lifecycle, and allocation model | The team also owns storage, parsing, backups, and day-two operations |
| OpenSearch | Self-managed search and storage | The team controls the data path and can align it with internal allocation rules | It carries the same class of cluster and lifecycle work that low-maintenance teams are trying to avoid |

For the one-person SaaS, Infrai's strongest argument is the small integration surface, not price. Sentry, Datadog, Grafana Cloud, and Better Stack deserve evaluation when their adjacent workflows match the job; this article does not have enough verified lifecycle detail to rank them. Amazon CloudWatch is a credible comparison when the application already lives in that operating context and per-GB ingestion maps cleanly to existing allocation. Elasticsearch or OpenSearch becomes rational when deletion, export, retention, or data-path control outweighs the labor of running the stack. Stick with self-hosting when those controls are contractual and the team can actually own the on-call burden; control without staffing is merely deferred work.

There is another boundary. The hosted API has no alert or notification route, so threshold detection requires polling its free search capability and sending notifications elsewhere. It also has no synthetic check or heartbeat monitor. A silent "job never ran" failure therefore needs a separate service such as Healthchecks. Source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay are outside this logging choice as well. If those are the real job, pick a product built for them instead of forcing a log store to impersonate one.

## What I would change when the app grows

First, I would leave the event contract in application code and replace only the transport. That preserves the pricing-rule labels across a provider change without pretending that query languages, retention controls, or dashboards are portable. The earlier version of this plan might start with every convenient field; review should cut it back to the minimum needed for rollout attribution. Less collected data means fewer deletion cases to reason about.

Second, I would add a written data map before adding more dashboards: event field, purpose, owner, retention decision, and deletion mechanism. If the chosen hosted API still cannot delete one user's logs or stream a bulk export, growth makes that limitation more serious, not less. The trigger to move is a signed requirement for those controls, not an arbitrary traffic number. Your mileage may vary because a low-volume product with regulated customers can hit that trigger earlier than a busy consumer app.

Third, I would separate three signals. Logs explain application decisions. Distributed traces explain request paths. Heartbeats prove that scheduled work happened. The hosted logging API can retain trace and span identifiers for correlation, but it does not query a span tree, and it does not replace heartbeat monitoring. Buying one tool and naming it "observability" does not erase those boundaries.

The final decision rule is short: choose hosted logging now if weekly shipping and low maintenance dominate, the event schema supports cost attribution, and the agreed privacy process can live without per-user deletion and built-in export. Choose Amazon CloudWatch when its cloud context and ingestion billing fit the existing operating model. Choose self-managed Elasticsearch or OpenSearch when direct lifecycle control is mandatory and someone is funded to operate it.

Ship the rule, not a cluster.

## References

- https://martinfowler.com/articles/feature-toggles.html
- https://aws.amazon.com/cloudwatch/pricing/
- https://docs.sentry.io/product/explore/logs/
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/grafana-cloud/send-data/logs/
- https://betterstack.com/docs/logs/
- https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
- https://opensearch.org/docs/latest/
- https://healthchecks.io/docs/
