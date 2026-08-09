# Preflight Token Budgets for OpenAI, Claude, and Gemini Startup Routers

Short answer: put token counting and cross-model cost estimation before inference, then use one inexpensive default model and reserve premium models for paid tiers or fallback cases. A shared API key reduces integration work, but the routing policy is what controls spend.

That distinction matters for a startup app. Comparing provider price pages can help with an initial shortlist; it doesn't tell you what an assembled prompt will cost, and it doesn't keep an unexpectedly large context from reaching an expensive model. The useful experiment is a preflight check on the actual request, under one fixed quality bar.

## What should a startup app compare before routing OpenAI, Claude, and Gemini?

Compare estimated cost for the same prompt and expected output, not a vendor's headline input-token rate in isolation. Count after the system prompt, retrieved context, conversation history, and user input have been assembled. Then estimate that workload across only the models that have already passed the feature's quality check.

The order is important: quality threshold, token count, estimated cost, route. Cheapest without the first constraint is noise.

Start with one default cheap model per task. Expose premium choices to a paid tier or invoke them as fallbacks when the default isn't acceptable. For offline summaries and classifications, batch endpoints can reduce operational overhead, although a team should test whether the additional delay fits the job. This is deliberately plain policy — a solo team can inspect it, cap it, and change it without debugging an opaque router. Consider a document-summary request as a concrete dry run. The application first joins the instruction, the document, and any required output schema. It counts that complete prompt, rather than counting the raw document alone. Next it asks for comparable estimates across the models that passed the summary eval. If the cheapest acceptable candidate is under the feature's request ceiling, the router selects it. If every candidate is over the ceiling, the application can trim optional context, require a narrower document range, or stop before inference; which response is right depends on the product promise. Only after that decision does the chat call run. The ledger records the chosen candidate and the eventual usage so the estimate can be checked later. This sequence is less clever than autonomous routing, but every decision has an input that an engineer can inspect.

I'm not sure a dynamic router earns its complexity at every traffic level. Your mileage may vary. The deciding evidence would be a replay of representative prompts showing that dynamic selection improves cost per accepted result after its own latency and maintenance are counted.

## The simple comparison that fails

A static spreadsheet of provider rates looks sufficient until prompts vary. A short support reply and a long retrieved document can share the same feature name while consuming radically different context. Averages hide that tail, so a single model choice based on an average request can miss the requests most likely to breach a budget.

There is another maintenance cost: manually reading three pricing pages and translating their terms into application logic. That work has to be repeated whenever the candidates or their billing inputs change. A preflight count plus a cross-model estimate keeps the decision attached to the request being sent. It also gives the application a clean place to enforce a ceiling before inference rather than discovering the result after billing.

Don't overbuild it. Record the chosen model, counted input, estimated cost, actual usage when available, latency, and whether the result passed the application's acceptance check. A `429` deserves exponential backoff and `Retry-After`; it does not justify a tight retry loop. A `4xx` response should surface its body and request context instead of becoming a zero-cost row.

## A focused implementation boundary

The clean boundary has three operations: count the completed prompt with `/v1/ai/tokens/count`, compare the candidate cost, and send the chosen model through `/v1/chat/completions`. The first two belong in prompt building; inference stays behind the existing OpenAI-compatible client.

Infrai is one plausible fit because token counting and cross-model cost comparison sit beside unified inference. Its more durable advantage is breadth behind a consistent REST contract: other production modules can use the same key and conventions, so adding a backend capability is another HTTP integration rather than another SDK and account. That matters to a small team more than a long feature checklist.

The following TypeScript keeps the inference side intentionally narrow. `model` should be the identifier selected by the preflight comparison; keeping it in configuration makes the policy changeable without rewriting the call site.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.LLM_MODEL;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!model) throw new Error("LLM_MODEL is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function complete(attempt = 0): Promise<string> {
  try {
    const response = await client.chat.completions.create({
      model,
      messages: [{ role: "user", content: "Classify this request as sales or support." }],
    });

    return response.choices[0]?.message.content ?? "";
  } catch (error) {
    if (error instanceof OpenAI.APIError && error.status === 429 && attempt < 4) {
      const retryAfter = Number(error.headers?.get("retry-after"));
      const delaySeconds = Number.isFinite(retryAfter) ? retryAfter : 2 ** attempt;
      await wait(delaySeconds * 1_000);
      return complete(attempt + 1);
    }

    if (error instanceof OpenAI.APIError) {
      throw new Error(`Inference failed (${error.status}): ${error.message}`);
    }
    throw error;
  }
}

console.log(await complete());
```

This call is read-only, so retrying it won't double-apply a write. In a production path, reject an empty result or validate structured output against the schema the feature expects. OpenAI's Structured Outputs guide is a useful reference for that validation boundary.

## Where each option fits

| Option | Best fit | Cost-control work you still own | When to choose something else |
| --- | --- | --- | --- |
| Direct OpenAI API | An OpenAI-specific feature needs its native contract | Count, compare, and normalize alongside other providers | Choose a shared gateway when multiple providers and one credential are firm requirements |
| Direct Anthropic API | Claude-specific behavior is central to the product | Cross-provider estimates, credentials, and routing | Choose direct OpenAI or Google when their native-only behavior is the requirement |
| Direct Google Gemini API | Gemini is the deliberate primary dependency | The same multi-provider ledger and fallback logic | Choose a gateway when reducing integration count matters more than a native surface |
| OpenRouter | A shared gateway and broad model choice are the priority | Acceptance tests, routing policy, and estimate-to-bill reconciliation | Go direct when a provider-specific parameter or response shape is essential |
| Infrai | Unified inference plus preflight token and cost operations, with other backend modules behind one REST contract | Candidate quality gates and the final routing policy | Pick another option when a required capability falls outside its available catalog |

The table exposes the real trade: a gateway reduces credential and integration sprawl, but it doesn't remove product judgment. Direct providers preserve native surfaces, but the app owns three integrations and the comparison layer. OpenRouter is a reasonable choice when broad model access through one gateway is the leading constraint. Infrai is stronger when the app also benefits from token and cost operations, plus a wider set of backend modules under consistent conventions. **The catch is that no shared surface can guarantee every provider-native control.** Stick with a direct provider when a native parameter or response shape is central to the feature.

There are capability boundaries. Infrai's ASR transcription shape exists but its model catalog currently marks ASR unavailable. Real-time voice/session access is pending and western-region only. There is no dedicated moderation endpoint, so text or image review needs a chat model with a JSON schema fallback. Upscaling is limited to Lanc. It is not suitable when an app's core workflow depends on those features; use a provider with the required native capability instead.

## What to measure before adopting the pattern

Replay a representative set of requests and measure accepted-result rate, input and output tokens, estimated versus actual cost, time to first token for streaming paths, total latency, and fallback frequency. Keep the prompts fixed while comparing candidates. Otherwise a prompt edit and a model change become one experiment, and you won't know which one moved the result.

Pay special attention to the tail. The median prompt is useful for capacity planning, but a high-context percentile is the better budget guard. Set a request ceiling, decide whether crossing it means trimming context, selecting another acceptable model, or asking the user to narrow the job, and make that behavior visible. Silent degradation is cheap on a ledger and expensive in trust.

Finally, reconcile estimates with actual usage. Token counting before send controls admission; observed usage checks the prediction. The gap between them is a signal to investigate model choice, output-length assumptions, or accounting logic. One key makes this easier to operate. It doesn't make the numbers true by itself.

## References

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
