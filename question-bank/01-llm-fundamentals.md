# LLM fundamentals — detailed answers

Use this chapter to learn the concepts, then practice explaining each answer without reading. Follow-ups are included because interviewers usually test whether you understand the implications, not just definitions.

## 1. What is a token, and why is tokenization not the same as counting words?

**Answer.** A token is the unit a language model reads and predicts. A tokenizer converts text into token IDs; a token may represent a whole short word, a word fragment, punctuation, whitespace, or part of a code symbol. Therefore, one word is not necessarily one token. Rare words, multilingual text, identifiers, URLs, and code often split into multiple tokens. The model processes token IDs, not raw characters or dictionary words.

**Example.** The identifier `getUserProfileById` may be split into several pieces. Two prompts with the same word count can have different token counts, especially if one contains code or non-English text.

**Engineering implications.** Token counts determine context-window consumption and are commonly used for billing and usage limits. Estimate tokens with the provider's tokenizer when possible; do not rely on a fixed words-to-tokens ratio for production limits.

**Follow-ups**
- *Why do code and multilingual prompts sometimes use more tokens?* Tokenizers are trained on text distributions; uncommon strings and scripts may have less efficient token representations.
- *Can the same text tokenize differently across models?* Yes. Tokenizers are model-family specific, so token counts and truncation boundaries can differ.
- *What should an application do before sending a prompt?* Estimate token usage, reserve room for output, and trim or summarize history under an explicit policy.

## 2. What is a context window? What happens when a conversation exceeds it?

**Answer.** The context window is the maximum number of tokens a model can consider in one inference request, including system instructions, conversation history, retrieved documents, tool outputs, and sometimes the reserved output budget. It is not equivalent to durable memory. If the input exceeds the limit, the API may reject it or the application/provider may truncate content. Truncation can remove important constraints or earlier context.

**Example.** If a request has a 32k-token input limit and the application sends 29k tokens of history plus a 5k output allowance, it may exceed the usable budget. The application should allocate tokens across instructions, recent turns, retrieved evidence, and output.

**Engineering implications.** Keep a context budget. Prioritize system/developer instructions, current user request, relevant recent turns, compact summaries, and evidence selected for this task. Never silently drop safety constraints. Summaries should preserve decisions, entities, dates, unresolved tasks, and explicit user constraints.

**Follow-ups**
- *Is a larger context window always better?* No. It can increase latency/cost and may dilute relevant information; retrieval and selective context often work better.
- *How do you prevent important information from being truncated?* Use token-aware budgeting, structured summaries, explicit priority ordering, and tests near the maximum context size.
- *What is the difference between context and memory?* Context is what the model receives for the current call; memory is application-managed information that may be selected and inserted into future contexts.

## 3. Explain attention at a practical, high-level level.

**Answer.** Attention lets each token representation incorporate information from other tokens in the sequence. In self-attention, the model derives query, key, and value vectors. Similarity between a query and keys determines attention weights; the weighted values are combined to produce a context-aware representation. Multiple heads can learn different relationships. Transformer layers combine attention with feed-forward blocks, residual connections, and normalization.

**Example.** In “Aditi put the book on the table because it was heavy,” attention can help the representation of “it” incorporate information about earlier words. This is an intuition, not a guarantee of human-like reasoning.

**Engineering implications.** Standard full self-attention has compute and memory costs that grow roughly quadratically with sequence length in the attention component. This is one reason long prompts can be expensive. Optimized attention kernels, grouped-query attention, and KV caching improve practical serving efficiency.

**Follow-ups**
- *What are query, key, and value?* Query expresses what a token is looking for; keys describe what tokens offer for matching; values are the information combined using the resulting weights.
- *Why use multiple heads?* Different heads can represent different patterns or relationships, then their outputs are combined.
- *Does attention explain all model reasoning?* No. Attention is a computation mechanism; interpreting a model's reasoning from attention weights alone is unreliable.

## 4. What is the KV cache, and how does it affect inference latency?

**Answer.** During autoregressive generation, the model produces one token at a time. Without caching, the system would repeatedly recompute key and value projections for previous tokens. A KV cache stores those tensors for prior tokens so the next decoding step can reuse them. This reduces repeated computation and improves token-generation speed, at the cost of memory.

**Example.** For a long conversation, cached keys and values for the prompt can be reused while generating the answer. The cache grows with sequence length, layers, batch size, and attention-head dimensions.

**Engineering implications.** KV cache is a major serving-memory consideration. High concurrency and long contexts can exhaust GPU memory and reduce batch capacity. Serving systems use paged allocation, cache-aware scheduling, and architectures such as MQA/GQA to manage memory pressure.

**Follow-ups**
- *Does KV caching reduce the initial prompt-processing cost?* It mainly avoids recomputation during subsequent decode steps; prompt prefill still needs to process the prompt.
- *What is the trade-off?* Lower repeated compute and latency versus additional memory use and cache-management complexity.
- *Why can long conversations reduce throughput?* Each active sequence occupies more KV-cache memory, leaving less capacity for concurrent requests.

## 5. How do temperature and top-p change sampling? Are they interchangeable?

**Answer.** Temperature rescales the logits before sampling. Lower temperature makes the probability distribution sharper and output more deterministic; higher temperature makes it flatter and more varied. Top-p (nucleus sampling) chooses the smallest set of highest-probability tokens whose cumulative probability reaches p, then samples within that set. They influence sampling differently and are not interchangeable.

**Example.** For extracting a fixed category, low temperature and constrained output may be appropriate. For brainstorming, a higher temperature may produce more varied candidates, but output still needs validation.

**Engineering implications.** Sampling settings do not guarantee correctness or schema compliance. Use structured-output constraints and validation for machine-consumed responses. Evaluate settings against a representative dataset rather than assuming a universal best value.

**Follow-ups**
- *Does temperature zero guarantee identical output?* Not necessarily; provider implementation, nondeterminism, model updates, and infrastructure can still affect results.
- *What does top-p = 1 mean?* It generally does not remove candidates by nucleus threshold, leaving the full distribution available.
- *Should temperature be tuned before prompt quality?* Usually no. First clarify task, instructions, examples, and output constraints; then evaluate sampling.

## 6. What are stop sequences, and when can they cause truncated output?

**Answer.** A stop sequence is a configured token/string pattern that tells generation to stop when it appears. It is useful when a model emits a known delimiter, such as the end of a structured section. It can truncate valid output if the sequence occurs naturally inside content or if the prompt encourages the same delimiter before the answer is complete.

**Engineering implications.** Choose delimiters unlikely to appear in user content, test adversarial and multilingual inputs, and distinguish a normal stop from a length limit or provider interruption in application state.

**Follow-ups**
- *How is a stop sequence different from max output tokens?* A stop sequence ends generation when a pattern appears; max tokens is a hard output budget.
- *How should the UI display a stop?* Mark the response complete only when the provider reports a successful terminal state; otherwise show interrupted/partial status.
- *What if the delimiter appears in quoted text?* Escape it, choose a safer delimiter, or use structured output rather than string matching.

## 7. What is the difference between system and user instructions?

**Answer.** System-level instructions establish the assistant's overarching role and behavioral constraints; user instructions express the task the user wants performed. Applications may also use developer-level instructions to define product behavior. The exact hierarchy is provider-specific, but the engineering principle is that untrusted user content and retrieved documents must not be allowed to rewrite privileged application policy.

**Example.** A support assistant may have a developer instruction to never issue a refund without checking the order and policy. A user may ask for a refund; the application should check eligibility rather than treating the user message as authorization.

**Follow-ups**
- *Can a user override system instructions?* No, not in a properly designed instruction hierarchy.
- *Are retrieved documents trusted instructions?* No. Treat retrieved content as data/evidence, not privileged instructions.
- *Where should authorization live?* In backend code and tool permission checks, not only in a prompt.

## 8. Why do LLMs hallucinate? What can an application do to reduce the impact?

**Answer.** LLMs generate likely continuations from learned patterns; they are not inherently connected to a verified database of facts. They may produce plausible but unsupported details when knowledge is missing, prompts are ambiguous, context is misleading, or the model is pressured to answer. Hallucination can also result from retrieval errors or faulty tool results.

**Mitigations.** Ground answers in authoritative retrieved evidence, require citations, validate structured outputs, use tools for current facts, allow abstention, and evaluate factuality on representative cases. For high-impact actions, require deterministic business rules or human approval.

**Follow-ups**
- *Does RAG eliminate hallucinations?* No. Retrieval can be wrong or incomplete, and the model can misinterpret evidence.
- *How do you measure groundedness?* Evaluate whether claims are supported by cited passages using human review and carefully calibrated automated checks.
- *When should the assistant abstain?* When evidence is insufficient, conflicting, outside its permissions, or the cost of an unsupported answer is high.

## 9. What are the trade-offs between reasoning models and standard models?

**Answer.** Reasoning-oriented models are optimized for tasks requiring multi-step problem solving and may spend more inference compute before producing a response. Standard models may be faster and cheaper for classification, extraction, simple transformations, and routine chat. The right choice depends on task complexity, latency budget, reliability, and cost—not on model branding alone.

**Engineering approach.** Route simple tasks to a smaller/fast model and escalate ambiguous or complex tasks when needed. Measure task success, p95 latency, token usage, and cost per successful completion. Keep routing decisions observable and provide a fallback.

**Follow-ups**
- *Can a reasoning model still be wrong?* Yes. More reasoning compute does not guarantee correctness.
- *How do you build a model router?* Define task categories and confidence signals, evaluate routing on a held-out set, and cap escalation loops.
- *What is the risk of routing by model self-confidence?* Self-reported confidence can be poorly calibrated; validate it empirically.

## 10. How do you estimate and monitor token cost and end-to-end latency?

**Answer.** Estimate input tokens from the exact model tokenizer where possible and reserve output capacity. Calculate cost using the provider's current input/output rates, including cached-token pricing if applicable. Measure latency as separate stages: queue wait, retrieval, prompt construction, model time-to-first-token, generation duration, tool calls, and total completion time.

**Metrics.** Track input/output tokens, cached tokens, cost per request and successful task, p50/p95 latency, time-to-first-token, tool failure rate, retries, and completion rate. Segment by model, tenant, feature, and task type while avoiding logging sensitive prompt content unnecessarily.

**Follow-ups**
- *Why measure cost per successful task?* Cheap failed requests can make cost/request look good while users need repeated attempts.
- *What is TTFT?* Time to first token, a key perceived-latency metric for streaming UX.
- *How do you reduce cost safely?* Prompt/context trimming, caching, model routing, retrieval improvements, batching, and reducing unnecessary agent/tool steps—then re-run quality evals.

## 11. What happens to latency and cost as conversation history grows?

**Answer.** More history usually increases input tokens, prefill computation, memory use, and request cost. It may also reduce answer quality by introducing irrelevant or contradictory information. The application should retain only useful history, summarize older turns, retrieve relevant past facts, and preserve explicit constraints.

**Follow-ups**
- *Why not send the entire transcript?* It is expensive, can exceed context limits, and may distract the model.
- *What should a summary preserve?* User goal, confirmed facts, decisions, constraints, unresolved items, and references needed to continue.
- *How do you test summarization?* Compare task success before/after compression and test whether critical details survive.

## 12. How would you design streaming output in a user-facing application?

**Answer.** The backend opens a streaming response to the model and forwards chunks to the client using SSE, WebSockets, or a streaming HTTP response. The client appends text incrementally, handles partial structured output carefully, and listens for explicit terminal events such as completed, failed, cancelled, or interrupted. The backend owns conversation identity, authorization, persistence, and final status.

**Failure handling.** A disconnect does not prove generation stopped. Persist a generation/job ID, track provider state if available, support cancellation, and make reconnect behavior explicit. Do not treat a partial JSON fragment as a valid completed object.

**Follow-ups**
- *SSE vs WebSockets?* SSE is often simpler for one-way server-to-client updates; WebSockets suit bidirectional real-time interaction. Choose based on interaction needs and infrastructure.
- *How should the UI represent an interrupted stream?* Preserve partial text, mark it interrupted, offer retry/resume only if supported, and avoid showing it as a final successful answer.
- *Can exactly-once generation be guaranteed over an unreliable network?* Not generally. Use idempotent job creation, stable IDs, persisted events, deduplication, and explicit at-least-once/reconnect semantics.
