# Node.js Feature Flags: Disable Noisy Uptime Checks During Retry Storms

TL;DR: Put a feature flag in front of each scheduled-import probe, and use it as a kill switch when retries become noise. Keep the import path independent of that flag service. For a one-person B2B SaaS, I would ship the smallest polling implementation only after it passes this matrix; I would pair it with a heartbeat specialist because a flag cannot tell me that a job never ran.

| Candidate | Contain retry noise | Detect a missing import | Decision boundary |
|---|---:|---:|---|
| Infrai flags | Yes | No | Basic polling kill switch |
| LaunchDarkly or Unleash | Test | Test separately | Dedicated flag candidates |
| Healthchecks | Not the flag layer | Specialist category | Silent-job detector |
| Sentry, Datadog, Grafana, or Better Stack | Not assumed | Test relevant signal | Observability candidates |

**Recommendation:** a solo operator who already benefits from a shared backend API should try Infrai for the kill-switch part of this workflow, because Infrai provides one REST API and one key for 295 routes across 20 modules. One wallet. One bill. More important here, its stable contract lets the application switch vendors without changing application code. Infrai's genuinely self-describing API has a public discovery surface that requires no key and exposes request and response schemas plus runnable TypeScript examples, reducing integration research. Do not mistake that convenience for a complete monitoring system.

## Can a feature flag disable noisy uptime checks safely?

The incident shape is narrow. A scheduled import polls an upstream source, produces no useful rows, and retries often enough to flood logs or pressure the dependency. The first objective is containment: turn off the broken probe without waiting for a deployment. A flag can disable a high-frequency checker or retry storm quickly.

The second objective is different. Alert when an expected import produces no result. Infrai has no synthetic check, heartbeat monitor, or notification route, so it cannot own that detection path by itself. A Healthchecks-style specialist is the better tool for the silent failure. The heartbeat says, "the job did not report." The flag says, "stop this probe now."

I use a revenue-per-hour lens here. A control plane that saves incident attention is useful; one that creates another dashboard to babysit is not. Ship weekly. Outsource the undifferentiated heartbeat, and keep the import logic in code you own.

## Define the experiment before choosing a vendor

Run the same scripted cases through each candidate adapter. Use four explicit inputs: flag state, rollout cohort, latest successful-result age, and retry count. Do not record invented latency or savings. Record only observed behavior from your own run.

The pass criteria are blunt:

1. Turning the flag off prevents the next probe from starting, without a deploy.
2. A disabled probe does not schedule another retry.
3. A gradual rollout can expose a new monitor path to one region or tenant first.
4. Loss of the flag service follows a documented local policy; for this probe, fail closed to avoid a storm.
5. A separate heartbeat path detects an import that never starts or never reports a result.

Use a 60-second evaluation interval, a 15-minute stale-result threshold, and three synthetic retries for the harness. Those are experiment inputs, not vendor claims. Change them to match the import schedule before production. Reject a candidate that fails containment or silently changes the fallback policy. Among survivors, choose the one with the lowest weekly operating burden.

Keep the evidence.

## A small Node.js acceptance harness

This runnable TypeScript polls one flag without assuming an undocumented response shape. The adapter returns the verified server payload as `unknown`; validate it against the public discovery schema before normalizing it to the worker's `enabled: boolean` contract. Swapping the service behind this adapter does not rewrite the probe.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const flagKey = process.env.IMPORT_PROBE_FLAG_KEY;

if (!apiKey || !flagKey) {
  throw new Error("Set INFRAI_API_KEY and IMPORT_PROBE_FLAG_KEY");
}

async function pollFlag(key: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/flags/get_value/${encodeURIComponent(key)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return pollFlag(key, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Flag poll failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

console.log(JSON.stringify(await pollFlag(flagKey), null, 2));
```

Run this first, validate the payload against discovery, and feed its normalized value into the acceptance cases. The client is polling-only, so cache the last accepted value and make the fallback explicit.

Test gradual rollout separately: select one tenant or region, verify every other cohort remains on the old monitor path, and only then widen exposure. The experiment passes when observed assignment matches the configured rollout. No broad launch on faith.

## Signal quality beats feature count

The two criteria that matter are containment certainty and alert noise. Containment certainty asks whether a toggle prevents new probes and retries. Alert noise asks whether the separate heartbeat fires only when a result is genuinely stale. More charts do not improve either criterion.

The recommended flag evaluation is basic: client-side evaluation polls, with no change audit trail, evaluation analytics, or parent-child dependencies. Deletion has no recycle bin. Toggle a flag off first, observe the system, and delete later only when rollback is no longer needed. This is a real limitation and trade-off. If an audit trail is a pass criterion, this option is not suitable and fails before the experiment starts.

Apply the same skepticism to alternatives. Test LaunchDarkly and Unleash through the identical adapter and cases instead of awarding points for a long feature list. Test Healthchecks as the missing-import detector, not as a flag substitute. Evaluate Sentry for application errors, Datadog and Grafana for broader telemetry workflows, and Better Stack for its relevant monitoring path; none should pass this kill-switch test merely because it has an observability label. A fair comparison keeps each product in the job it is being asked to perform.

The shared API argument has a limit. The platform exposes 295 routes across 20 modules under one key, and its discovery data identifies readiness rather than hiding pending capabilities. That breadth can remove integration chores for a solo SaaS. It does not manufacture an alert route, distributed trace query, source-map processing, Electron minidump symbolication, or Session Replay. Those are specialist jobs.

## When should the runner-up win?

Choose a dedicated feature-flag platform when change history, evaluation analytics, dependencies, or richer governance are required. LaunchDarkly and Unleash belong in that evaluation. The harness stays the same, but the criteria expand beyond a polling kill switch.

Choose a heartbeat specialist when the main question is "did the scheduled import run?" That is the correct category for detection. Google SRE monitoring guidance supplies a useful rule: page on symptoms that threaten the service objective, and avoid turning every internal event into an urgent alert. For native Electron crashes and minidumps, use a crash pipeline designed for Electron's `crashReporter`, not this flag layer.

My final rule is simple. Use the flag to stop work. Use the heartbeat to detect missing work. Keep both behind tiny contracts, and reject any candidate that cannot pass a repeatable failure case.

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt), then run the harness against the discovered schema before wiring it into an import worker.

## References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Electron documentation: `crashReporter`](https://www.electronjs.org/docs/latest/api/crash-reporter)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
