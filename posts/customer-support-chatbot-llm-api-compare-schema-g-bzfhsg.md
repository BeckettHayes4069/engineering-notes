# Customer Support Chatbot LLM API: Compare Schema-Gated CRM Action Results

Short answer: compare each LLM API for a customer support chatbot by accepted CRM actions from a fixed replay set, then use price only to break a tie between options that clear the same correctness gate.

| First filter | What to record | Decision |
| --- | --- | --- |
| Structured output | Valid shape, allowed values, and evidence | Reject an answer that cannot be written safely |
| Task correctness | Right owner, action, date, and account | Reject a plausible but wrong action |
| Runtime behavior | Complete stream, latency, and retry count | Keep only candidates that meet the product budget |
| Cost | Spend per accepted action | Compare survivors, not raw token rates |

For a one-person SaaS that ships weekly, the recommendation is a replay harness plus a hard schema boundary. Run every candidate against the same support and sales-call transcripts. The cheapest LLM API is the one with the lowest cost per accepted action, not necessarily the smallest price printed beside a token unit.

This changes the usual GPT, Claude, Gemini, and OpenAI-compatible comparison. Brand and protocol still matter, but they come after output correctness. A cheap response that invents a follow-up date or assigns the wrong CRM owner creates manual work, and manual review is the scarce resource in a solo operation.

## How should a customer support chatbot compare LLM API options?

Start with the write path. In this example, an in-app customer support chatbot also summarizes a sales call into CRM actions. The model must return an account identifier, a short summary, and zero or more actions. Each action needs a constrained type, an owner, a due date, and a supporting quote from the transcript.

That last field matters. A structurally valid object can still be wrong. Evidence makes the failure visible before an update reaches the CRM. It also gives a reviewer something faster to inspect than the full call.

Use a compact replay corpus drawn from the traffic the product is expected to handle. Include short calls, long calls, corrections, vague dates, multiple speakers, and calls with no actionable next step. Keep the expected result beside each case, but allow explicit alternatives when the source is genuinely ambiguous. I'm not sure a single gold sentence is useful for every summary; exact matching often measures wording instead of business correctness. The unresolved part should be decided by the product policy, not quietly delegated to a model.

The first score is binary: can the result cross the write boundary? The second is semantic: did it preserve the transcript's facts? Only then record latency and usage. This ordering prevents an attractive per-token figure from hiding a low acceptance rate.

A practical test record can stay small:

```ts
type ExpectedAction = {
  type: "follow_up" | "send_material" | "update_stage";
  owner: string;
  dueDate: string | null;
  evidence: string;
};

type ReplayCase = {
  id: string;
  transcript: string;
  expectedAccountId: string;
  allowedActions: ExpectedAction[][];
};

type TrialResult = {
  caseId: string;
  candidate: string;
  schemaValid: boolean;
  taskAccepted: boolean;
  durationMs: number;
  retryCount: number;
  inputUnits: number;
  outputUnits: number;
  errorCode?: "E_SCHEMA_001" | "E_EVIDENCE_002" | "E_TASK_003";
};
```

Don't collapse these fields into one quality score too early. An average can let a quick, cheap, invalid response cancel out a slower correct one. A gate preserves the difference.

## Structured correctness is a runtime contract

The API boundary should treat model output as untrusted input. Parse it, validate every field, verify that quoted evidence occurs in the source transcript, and apply business rules before writing anything. This is ordinary application engineering — the probabilistic component does not get a private lane around validation.

Consider a call where the buyer says, “Send the security packet next Tuesday, but don't change the opportunity stage yet.” A fluent summary can still cause damage in several ways. It may turn “next Tuesday” into an unsupported date because the call timestamp was omitted. It may emit `update_stage` because stage changes are common in similar calls. It may assign the action to the buyer rather than the account owner. Or it may produce perfect JSON whose evidence quote never appeared in the transcript.

One long example is worth more than ten generic prompts here. Store the call time and timezone with the replay case. Define whether relative dates may be resolved, whether unresolved dates become `null`, and which CRM user IDs are legal for that tenant. Require `actions: []` when no action is supported. Then run the same case across every candidate with the same instructions and the same allowed context. If one candidate needs a materially different prompt, keep that prompt version in the result because prompt maintenance is part of the runtime burden.

The policy should be boring. That's good.

Schema validity alone is insufficient. A validator can prove that `dueDate` is a string; it cannot prove that the date came from the call. Add deterministic checks where the transcript permits them and a review queue where it does not. For high-impact actions, an accepted draft can still require a human click. For low-impact actions, evidence plus tenant rules may be enough to automate the write. The boundary depends on the cost of a wrong action, not on how confident the prose sounds.

Track rejection reasons separately. `E_SCHEMA_001` means the object cannot be parsed or violates the shape. `E_EVIDENCE_002` means an action lacks a supporting source span. `E_TASK_003` means the shape is valid but the expected business action is wrong. These are application-defined codes, not vendor errors. They turn a vague “model quality” discussion into a queue of fixable failure classes.

## Measure accepted work, not advertised units

Raw API price is easy to compare and easy to misuse. Different outputs can consume different amounts of text, require different prompts, or trigger different retry and review paths. For this workload, calculate spend per accepted CRM action over the replay set. An invalid response contributes cost but contributes no accepted action.

No benchmark number belongs in the decision note until it comes from the application's own corpus. Public leaderboards do not contain the tenant vocabulary, transcript noise, CRM rules, or action taxonomy. Your mileage may vary even between two products that both call themselves support chatbots.

Use the revenue-per-hour lens. A candidate that needs frequent prompt surgery, a custom parser, and manual cleanup consumes the same week that could ship a customer-facing feature. Still, don't assign imaginary dollar values to engineering time. Record actual review minutes and implementation work during the trial, then make the trade in plain terms.

The operational columns matter too. Capture time to first visible text for the chat experience, time to the validated final object for the CRM path, incomplete streams, and application retries. Server-Sent Events can carry a one-way event stream to the browser, but streamed display text should remain separate from the final validated object. The user may see progress while the application waits to validate the complete action payload. MDN documents the browser event-stream mechanism and its connection behavior; it does not make partial model output safe to write.

Retrieval deserves the same discipline. A Postgres application can use pgvector for vector similarity search, but retrieval should return evidence, not grant authority. Log the document identifiers and passages placed in context. A retrieved policy can support a response to the customer while the CRM action still passes tenant ownership and evidence checks.

Cost comes last for a reason. Once two candidates clear the same acceptance, latency, and maintenance gates, their measured usage and current billing terms can break the tie. Re-run that calculation when prompts, traffic mix, or provider terms change. Price pages move; the harness remains useful.

## A focused TypeScript replay harness

Keep the adapter thin so the test measures the candidate rather than a pile of vendor-specific application code. The endpoint, authorization value, candidate identifier, and response extractor belong in configuration. The application contract stays fixed.

```ts
type CrmAction = {
  type: "follow_up" | "send_material" | "update_stage";
  owner: string;
  dueDate: string | null;
  evidence: string;
};

type CallSummary = {
  accountId: string;
  summary: string;
  actions: CrmAction[];
};

type Candidate = {
  name: string;
  endpoint: string;
  authorization: string;
  buildBody: (transcript: string) => unknown;
  extractText: (response: unknown) => string;
};

function isCallSummary(value: unknown, transcript: string): value is CallSummary {
  if (typeof value !== "object" || value === null) return false;
  const item = value as Record<string, unknown>;
  if (typeof item.accountId !== "string" || typeof item.summary !== "string") {
    return false;
  }
  if (!Array.isArray(item.actions)) return false;

  const allowed = new Set(["follow_up", "send_material", "update_stage"]);
  return item.actions.every((raw) => {
    if (typeof raw !== "object" || raw === null) return false;
    const action = raw as Record<string, unknown>;
    return (
      typeof action.type === "string" &&
      allowed.has(action.type) &&
      typeof action.owner === "string" &&
      (typeof action.dueDate === "string" || action.dueDate === null) &&
      typeof action.evidence === "string" &&
      action.evidence.length > 0 &&
      transcript.includes(action.evidence)
    );
  });
}

async function runCandidate(
  candidate: Candidate,
  replay: ReplayCase,
): Promise<TrialResult> {
  const started = performance.now();
  const response = await fetch(candidate.endpoint, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: candidate.authorization,
    },
    body: JSON.stringify(candidate.buildBody(replay.transcript)),
  });

  const wireValue: unknown = await response.json();
  const text = candidate.extractText(wireValue);
  let parsed: unknown;

  try {
    parsed = JSON.parse(text);
  } catch {
    return {
      caseId: replay.id,
      candidate: candidate.name,
      schemaValid: false,
      taskAccepted: false,
      durationMs: performance.now() - started,
      retryCount: 0,
      inputUnits: 0,
      outputUnits: 0,
      errorCode: "E_SCHEMA_001",
    };
  }

  const schemaValid = isCallSummary(parsed, replay.transcript);
  const summary = schemaValid ? parsed : null;
  const taskAccepted =
    summary !== null &&
    summary.accountId === replay.expectedAccountId &&
    replay.allowedActions.some(
      (allowed) => JSON.stringify(allowed) === JSON.stringify(summary.actions),
    );

  return {
    caseId: replay.id,
    candidate: candidate.name,
    schemaValid,
    taskAccepted,
    durationMs: performance.now() - started,
    retryCount: 0,
    inputUnits: 0,
    outputUnits: 0,
    errorCode: !schemaValid
      ? "E_EVIDENCE_002"
      : taskAccepted
        ? undefined
        : "E_TASK_003",
  };
}
```

The zero usage fields are deliberate inputs for the adapter to populate from its response metadata; don't estimate them from JavaScript string length. The harness can reject malformed HTTP responses before extraction, while the stored trial record should distinguish transport failures from the three application-level rejection codes shown here.

Run replays before a runtime change and on a schedule that matches the product's release rhythm. Pin the candidate configuration and prompt version in each result. A weekly ship cycle does not need a grand evaluation platform. It needs a repeatable command, inspectable fixtures, and a diff that can block deployment.

## When is the compatible runner-up the better choice?

Structured-output correctness wins this decision only when the CRM action is the risky part. Stick with the runner-up when it clears the correctness gate and reduces a larger integration burden: for example, the application already has a stable compatible adapter, the team must switch endpoints without changing its internal interface, or a required deployment environment constrains which service can be reached. Compatibility is valuable when it removes undifferentiated glue work.

The catch is that protocol compatibility does not prove behavioral equivalence. Re-run the corpus after any candidate swap. Request fields may line up while action accuracy, streaming behavior, usage metadata, or output shape changes. The adapter can normalize the wire format; it cannot normalize judgment.

This approach is not suitable when the chatbot only drafts disposable text and never triggers a business action. In that case, a lighter review of relevance, latency, and measured spend may be enough. It is also a poor fit for very low traffic with mandatory human approval on every response, where building a large automated evaluator could cost more time than it returns. Ship the smallest gate that protects the actual workflow.

For the sales-call-to-CRM system, keep the rule firm: correctness gate first, operational fit second, accepted-work cost third. It is a less exciting comparison than a price table. It is also the one that protects the week's shipping time.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
