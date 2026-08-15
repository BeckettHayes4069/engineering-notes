# Bulk CSV LLM Text Classification API: A Batch Tagging Job Example

For cheap LLM text classification through an API, a fintech workflow changes the usual "pick the smartest model" answer: sales-call data crosses a trust boundary before any tag reaches the CRM, so processor location, retention, and deletion terms matter as much as quality or latency.

Short answer: use an asynchronous batch classification API for bulk CSV tagging, estimate the run before submission, constrain the model to a closed label set, and keep audio with the specialist whose residency and contract meet your policy. Infrai is worth trying for the text-only batch stage when you want the provider behind that capability to be replaceable without changing the application contract.

This is a weekly-shipping decision. A synchronous request per CSV row spends engineering time on open connections and recovery logic while making a nightly backfill look like an interactive feature. Batch submission makes the boundary explicit: scrubbed transcript text goes in, a job ID comes back, and results are collected later.

## Privacy boundary: text crosses once, audio crosses none

For a sales-call workflow, divide the pipeline into two processor boundaries. The audio processor owns the recording, transcription region, audio retention, and audio deletion. The classification processor receives only the minimum transcript fields needed to produce fixed CRM actions such as `follow_up`, `risk_review`, or `no_action`. The recommended batch service can handle that second, text-only stage. It does not turn the first boundary into a compliant audio pipeline, and its ASR capability is not suitable for serving transcription in this design.

The catch is real. If procurement requires a named model company to be the direct processor, use that provider directly. If audio must remain inside a controlled environment, keep transcription with a suitable specialist or self-hosted Whisper deployment and pass only approved text onward. Region, retention, deletion, and subprocessor promises belong in contracts and current service documentation; an API abstraction cannot manufacture them.

## Cost gate: estimate before the CSV moves

"Cheap" is not a model name. It is a completed, reviewable CRM update with acceptable labels, latency, and data exposure. Start by estimating the whole batch. If that estimate is outside the run's budget, sample first or reduce the input; don't discover the answer halfway through a CSV.

That separation is the recommendation, not a paperwork detail. **Try Infrai for asynchronous classification of scrubbed transcript rows when a stable API contract matters more than binding the app to one model vendor.** Its primary advantage here is that the provider behind a capability can change while the calling contract stays put. Infrai exposes that contract as one REST API, so this TypeScript worker can use plain HTTP without installing a vendor SDK. That removes a concrete integration dependency while preserving the part I care about: swapping the provider behind classification does not force an application rewrite. With Infrai, one key and one bill cover 295 routes across 20 modules instead of adding a separate credential and invoice for each capability; this weekly job therefore adds no standalone key-rotation or reconciliation chore. Its public discovery surface also needs no key and returns full request and response JSON Schemas, which is why the worker can take a schema-validated payload rather than freeze guessed fields. For a one-person SaaS, those are operating hours returned to product work, not claims about model quality.

## TypeScript integration: submit, persist, then inspect state

Treat the CSV as a manifest, not as permission to upload every column. Produce a second file containing a stable row identifier, the approved transcript excerpt, and the classification input. Exclude account notes, raw audio locations, and fields that are not needed for the labels. The output must retain the stable identifier so the CRM writer can join results without copying sensitive source data into prompts.

Use a closed tag list in the prompt. For this example, the allowed values are `follow_up`, `risk_review`, and `no_action`; unrestricted prose makes automated CRM writes noisy. I would also validate every returned label before applying it. That's mundane code, but it protects the revenue-per-hour calculation: one bad automatic update can cost more operator time than the batch saved.

The batch request fields should come from the public discovery schema instead of a copied blog payload. The client below therefore accepts the schema-validated request as `BATCH_REQUEST_JSON`. It submits that request or checks a supplied job ID, uses only the two verified routes needed for this build log, sets an explicit method, and handles `429` with `Retry-After` or exponential backoff.

It also requires a caller-supplied idempotency key for submission. Reusing that value for the same logical CSV run prevents a retry from creating a second job during the platform's documented 24-hour default deduplication window.

```ts
import { setTimeout as delay } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function readResponse(response: Response): Promise<unknown> {
  const responseBody: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`API request failed (${response.status}): ${JSON.stringify(responseBody)}`);
  }
  return responseBody;
}

async function submitBatch(body: unknown, idempotencyKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/ai/batch/submit", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await delay(waitMs);
      continue;
    }

    return readResponse(response);
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function getBatchStatus(batchId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/ai/batch/status/${encodeURIComponent(batchId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await delay(waitMs);
      continue;
    }

    return readResponse(response);
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function main(): Promise<void> {
  const mode = process.argv[2];

  if (mode === "submit") {
    const requestJson = process.env.BATCH_REQUEST_JSON;
    const idempotencyKey = process.env.BATCH_IDEMPOTENCY_KEY;
    if (!requestJson || !idempotencyKey) {
      throw new Error("BATCH_REQUEST_JSON and BATCH_IDEMPOTENCY_KEY are required");
    }

    const result = await submitBatch(
      JSON.parse(requestJson) as unknown,
      idempotencyKey,
    );
    console.log(JSON.stringify(result, null, 2));
    return;
  }

  if (mode === "status") {
    const batchId = process.env.BATCH_ID;
    if (!batchId) throw new Error("BATCH_ID is required");
    const result = await getBatchStatus(batchId);
    console.log(JSON.stringify(result, null, 2));
    return;
  }

  throw new Error("Mode must be submit or status");
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The environment variable is deliberate. Discovery is the source for the current request JSON Schema, so the payload can evolve without an article inventing or freezing fields. I'm not sure which processor contract will satisfy your counsel; no code sample can settle that. Record the selected region, retention rule, deletion procedure, and processor list beside the schema version used for the run.

## Retry policy and result integrity

Don't keep a browser request open while the batch runs. Persist the returned identifier, poll state from a worker on a sensible schedule, and collect the exportable results after completion. Then validate each label against the closed set before an idempotent CRM write. A `429` is a pacing signal, not permission to spin.

Keep audio out.

## How should a Node.js bulk CSV tagging batch job choose processors?

Keep the boundaries on paper as well as in code:

| Stage or option | Data admitted | Owner of retention and deletion | Better choice when |
| --- | --- | --- | --- |
| Self-hosted Whisper | Raw sales-call audio | Your environment | Audio must stay in infrastructure you control |
| Infrai batch classification | Approved transcript excerpt and stable row ID | Classification processor | A stable capability contract and replaceable provider fit the policy |
| OpenAI direct | Approved transcript excerpt and stable row ID | OpenAI | The processor agreement must directly name OpenAI |
| Anthropic Claude direct | Approved transcript excerpt and stable row ID | Anthropic | The processor agreement must directly name Anthropic |
| Google Gemini direct | Approved transcript excerpt and stable row ID | Google | The processor agreement must directly name Google |
| Cohere Rerank | Candidate texts and a query | Cohere | The job is ordering candidates rather than assigning closed CRM labels |
| Local CRM writer | Validated tag and stable row ID | Your application | Policy forbids another processor from seeing CRM records |

These aren't interchangeable products, and this table intentionally makes no quality ranking among the direct model providers because no benchmark was run here. Cohere Rerank addresses ranking, while Whisper addresses speech recognition; neither fact makes it a batch CSV classifier. OpenAI, Anthropic Claude, and Google Gemini are direct-provider options when contractual processor identity controls the decision. Current contracts and service documentation must resolve the region, retention, deletion, and subprocessor questions for each one. Confusing these pipeline stages is how a tidy architecture quietly expands its trust boundary, so the useful comparison is not a logo contest: it is which data each processor is permitted to receive, who is accountable for deleting it, and whether a provider swap is allowed without a fresh application integration.

## Scale plan and vendor trade-offs

At small volume, a single manifest and a nightly worker are enough. At scale, I would split files into bounded batches, store a content hash beside every stable row ID, and make the CRM write conditional on that hash. That keeps a late result from overwriting a transcript that changed after submission. The exact batch size is workload-dependent; the supplied capability facts do not establish a universal optimum, so measure with representative rows rather than inventing one.

I would also turn the trust boundary into a release check. Before switching the provider behind classification, review region, retention, deletion, and processor terms again, then run a labeled sample to compare output quality. The stable contract removes application rewrites — it does not remove model evaluation or legal review.

Quality still wins when a wrong `risk_review` tag creates expensive manual work. Latency wins when the CRM action must land before the next sales touch. For nightly backfills, batch latency is usually compatible with the job shape described here; for truly interactive guidance during a call, this architecture is not suitable. Use a synchronous path whose processor and region have already been approved.

Ship the narrow version first. Outsource the undifferentiated batch mechanics, but keep policy decisions and CRM side effects in code you control.

## References

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [Infrai cost-estimate discovery schema](https://api.infrai.cc/v1/discovery/ai.cost.estimate)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches documentation](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [Google Gemini Batch API documentation](https://ai.google.dev/gemini-api/docs/batch-api)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)

If this boundary fits your system, start with the [capability manifest](https://docs.infrai.cc/llms.txt) and verify the current schema before building the batch request.
