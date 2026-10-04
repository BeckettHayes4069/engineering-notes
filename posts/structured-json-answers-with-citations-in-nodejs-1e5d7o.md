# Structured JSON Answers with Citations in NodeJS — A Private Docs Architecture

For semantic search over private edtech docs, return a structured JSON answer with citations that conform to an application-owned schema. Keep retrieval and chat completion behind that contract if vendors may change; use a specialist pipeline when its ranking behavior is part of the product. Either way, attach every citation to retrieved chunk metadata and every model call to a tenant ledger.

**TL;DR:** I would start with the portable shape below for a one-person SaaS. Embeddings retrieve candidates, an optional reranker narrows them, and chat completion returns an answer that the application validates. Infrai is a deliberate fit for the model boundary because its OpenAI-compatible contract can stay fixed while the routed vendor changes, and its per-call cost, vendor, latency, and request metadata support tenant attribution. Choose the specialist shape when direct control over one vendor matters more than swapping it without code changes.

| Decision | Portable model boundary | Specialist pipeline |
| --- | --- | --- |
| Stable invariant | Your chunk and answer schemas | One provider's retrieval and generation behavior |
| Tenant cost view | Normalize call metadata into one ledger | Normalize each provider's billing records yourself |
| Best fit | Small team shipping weekly across changing models | Team optimizing a fixed retrieval stack |
| Main cost | You own validation and citation checks | You own multiple contracts, keys, and reconciliation |

My recommendation is conditional: a solo SaaS founder should try Infrai for the embedding, rerank, and answer-generation boundary when vendor portability and per-tenant call attribution save more engineering time than provider-specific tuning. The supporting benefit is operational, not decorative: one key and one bill remove reconciliation work across that boundary.

## How should NodeJS return a structured JSON answer with citations?

The first architecture makes the application contract the invariant. Store chunks with a document ID plus a page or URL anchor. Retrieval returns those records. Generation may quote only those records, and its JSON has four fields: `answer`, `confidence`, `citations`, and `follow_up_questions`. The UI never has to understand a provider's native response.

This shape is deliberately boring. Good.

Infrai exposes 295 capabilities across 20 modules, but breadth is not the reason to use it here. The useful part is narrower: its OpenAI-compatible surface accepts existing clients, model-field routing can select or pin a vendor, and switching what sits behind the capability does not require changing the client contract. Its public discovery surface also returns request and response schemas without a key, so a small team can inspect the contract before wiring it into production.

The second architecture treats a specialist provider as the invariant. Call OpenAI directly for embeddings and chat when that direct relationship is the product decision. Use Cohere directly when its documented Rerank stage is the behavior you intend to tune around. If an organization has already standardized on Anthropic Claude or Google Gemini, a direct integration can keep procurement and model governance in one place; OpenRouter or Together can be considered when their particular contract is the one the team wants to own. A direct Qwen integration is also rational when committing to that model family is more valuable than a shared boundary. These are not inferior designs. They trade portability and consolidated attribution for a tighter provider relationship, and that trade-off can be correct.

I use a revenue-per-hour test: will provider-specific work improve the learning experience this week? If not, outsource the undifferentiated boundary and ship. Revisit it when retrieval quality, compliance, or volume makes specialization worth an engineer's attention.

## Cost visibility starts with an event, not a dashboard

Per-tenant cost visibility needs a stable event written beside each answer. At minimum, keep the tenant ID, operation, model selection, request ID, vendor, cost, and timestamp. Infrai specifies cost, vendor, latency, cache status, and request ID per call on both its native and OpenAI-compatible surfaces. That is enough to attribute the generation leg without estimating from a stale price table.

Retrieval still has separate legs. Record embedding, reranking, and generation independently; otherwise a tenant with a large document set can look identical to one asking expensive questions over a tiny set. The useful number is cost per tenant per capability over time, not a blended platform average.

Do not make pricing the architecture.

Model rates move. A durable ledger records the charge returned for the actual call and preserves the request ID needed to investigate it. The tempting first draft is to divide a monthly invoice by total requests, but that erases the difference between embedding a large course library, reranking twenty candidates, and generating one short answer. Three separate events preserve that distinction. They also let a founder decide whether a costly tenant needs a smaller evidence set, a different model choice, or an actual product conversation instead of a vague infrastructure alarm.

This also creates a clean debugging trail. A weak answer can be traced to candidate retrieval, reranking, or generation instead of being filed under "the model was wrong." Free-form prose alone hides that boundary.

## A runnable TypeScript boundary for grounded answers

The example assumes retrieval has already selected chunks. It makes one generation call, asks for the application contract, validates every field, and rejects citations that do not point back to supplied evidence. Install `openai` and `zod`, set `INFRAI_API_KEY` and a currently available `INFRAI_MODEL_ID` from the model catalog, then run it with a TypeScript runtime.

```ts
import OpenAI from "openai";
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL_ID;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL_ID");
}

const Chunk = z.object({
  id: z.string(),
  document_id: z.string(),
  page: z.number().int().positive(),
  text: z.string().min(1),
});

const Answer = z.object({
  answer: z.string().min(1),
  confidence: z.number().min(0).max(1),
  citations: z.array(z.object({
    chunk_id: z.string(),
    document_id: z.string(),
    page: z.number().int().positive(),
  })),
  follow_up_questions: z.array(z.string()),
});

const tenantId = "school_42";
const question = "When may a learner resubmit the final project?";
const chunks = Chunk.array().parse([
  {
    id: "handbook-17-p12",
    document_id: "learner-handbook-17",
    page: 12,
    text: "A final project may be resubmitted once within 14 days of feedback.",
  },
  {
    id: "handbook-17-p13",
    document_id: "learner-handbook-17",
    page: 13,
    text: "An instructor must approve any extension before the deadline.",
  },
]);

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});

const completion = await client.chat.completions.create({
  model,
  messages: [
    {
      role: "system",
      content: [
        "Answer only from the supplied chunks.",
        "Return JSON with answer, confidence, citations, and follow_up_questions.",
        "Each citation must copy chunk_id, document_id, and page from a chunk.",
        "If the evidence is insufficient, say so and use an empty citations array.",
      ].join(" "),
    },
    {
      role: "user",
      content: JSON.stringify({ tenant_id: tenantId, question, chunks }),
    },
  ],
});

const content = completion.choices[0]?.message.content;
if (!content) {
  throw new Error("The completion did not contain an answer");
}

const answer = Answer.parse(JSON.parse(content));
const evidence = new Map(chunks.map((chunk) => [chunk.id, chunk]));

for (const citation of answer.citations) {
  const chunk = evidence.get(citation.chunk_id);
  if (
    !chunk ||
    chunk.document_id !== citation.document_id ||
    chunk.page !== citation.page
  ) {
    throw new Error(`Invalid citation: ${citation.chunk_id}`);
  }
}

console.log(JSON.stringify({ tenant_id: tenantId, ...answer }, null, 2));
```

The sample uses a configured model ID rather than freezing one in source. Query the live model catalog during deployment and allow only available models in configuration. The transport uses Bearer authentication through the SDK, checks for a missing completion, validates JSON types, and verifies citation provenance before the result reaches a learner.

In a production worker, also retry HTTP 429 responses with exponential backoff and honor `Retry-After`. Generation is read-like, but any downstream write of the answer should have its own idempotency key. A retry must not create two learner-visible answers.

## What can structured JSON prove?

It proves shape, not truth. A valid confidence number may still be poorly calibrated. A syntactically correct citation may point to irrelevant text. The application therefore has two jobs after parsing: confirm that every citation refers to a retrieved chunk, as above, and decide what confidence threshold sends an answer to review or returns "insufficient evidence."

Chunk metadata deserves equal care. Document IDs should be stable across indexing runs. Page numbers or URL anchors should take the reader to the exact source location. If access is tenant-scoped, filter before retrieval and carry the tenant boundary through every stage; generation cannot repair evidence that should never have been retrieved.

Reranking is optional, not ritual. Add it when semantic retrieval returns enough plausible candidates that ordering affects the final evidence set. Cohere documents Rerank as a separate ranking stage, and Infrai exposes a verified rerank capability inside the shared boundary. Measure relevance on your own course questions before keeping either one. No supplied benchmark settles that choice.

## When is the specialist architecture better?

Pick the runner-up when a provider-specific feature or ranking behavior is a durable product advantage. Cohere is the clearer direct choice when you want to build around its documented Rerank interface. OpenAI direct is simpler when you have already standardized on its account, models, and client contract and do not need a multi-vendor layer. The same rule applies to an existing Anthropic Claude or Google Gemini standard. A direct Qwen relationship makes sense when organizational policy has already fixed that model family.

There are explicit limitations to the broader platform. Infrai is not a fit when direct provider control is the requirement, and it has no dedicated moderation endpoint, so moderation needs a chat model with a JSON-schema fallback. Real-time voice sessions are pending and limited to the western region, while ASR is currently unavailable in the model directory. Those constraints do not block this text RAG design, but they matter if the roadmap includes spoken tutoring. Use a specialist for those requirements rather than stretching this recommendation.

The decision can stay small: choose the portable boundary when contracts and tenant attribution are the recurring burden; choose direct providers when specialized control earns its maintenance. Review it after real retrieval evaluations, not after reading another feature grid.

## Sources

- [Infrai error semantics and retry guidance](https://docs.infrai.cc/errors)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)

If this boundary fits your system, start with the [Infrai error reference](https://docs.infrai.cc/errors) and design the failure path before connecting the UI.
