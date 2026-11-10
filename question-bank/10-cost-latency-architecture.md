# Cost, latency, and architecture — detailed answers

## 1. How would you reduce the cost of a 10-step agent?

**Answer.** First trace where cost is spent: model input/output tokens, number of model calls, retrieval, tools, retries, and failed tasks. Then reduce unnecessary steps, trim repeated context, cache stable prefixes/results where valid, route simple steps to smaller models, batch independent work, and improve tool schemas to reduce repair calls. Re-run quality and safety evals after each change.

**Follow-ups**
- *What is the biggest mistake in cost optimization?* Optimizing token price while increasing failures or repeated attempts.
- *What metric should you optimize?* Cost per successful task, with quality and latency constraints.
- *Can fewer steps reduce quality?* Yes; use evals to ensure the agent still gathers sufficient evidence.

## 2. When should a small model handle a step and a stronger model handle another?

**Answer.** Use a smaller model for bounded, low-ambiguity tasks such as simple classification, extraction, or routing when it meets the quality threshold. Use a stronger model for complex reasoning, ambiguous instructions, difficult synthesis, or high-value tasks. Route based on evaluated task characteristics and measured confidence signals, not unsupported self-reported certainty.

**Follow-ups**
- *How do you decide the threshold?* Build a labelled set and compare quality/cost curves across models.
- *What is escalation?* Re-run a difficult case with a stronger model when a validated signal indicates the first attempt may be inadequate.
- *What is the risk?* Incorrect routing can send complex tasks to a weak model or needlessly increase spend.

## 3. What are the risks of model routing?

**Answer.** Different models may have different tool-calling formats, context limits, safety behavior, latency, and output quality. A router may misclassify tasks, oscillate between models, or introduce inconsistent outputs. Use a bounded routing policy, normalize provider responses, validate schemas, record model versions, and test each route.

**Follow-ups**
- *How do you avoid route oscillation?* Set one-way escalation or a maximum number of transitions.
- *What if a model is unavailable?* Use a tested fallback or return a clear degraded state.
- *How do you evaluate routing?* Measure route accuracy, task success, cost, and latency per class.

## 4. How do prompt caching and batching affect cost and latency?

**Answer.** Prompt caching reuses provider-side computation for repeated eligible prompt prefixes, potentially reducing input cost and prefill latency. Batching combines multiple independent requests to improve throughput and utilization, but can increase per-request wait time and complicate latency guarantees. Both depend on provider/model behavior and workload patterns; measure actual cache hit rate and batch wait time.

**Follow-ups**
- *What makes a good cache prefix?* Stable repeated instructions, tool schemas, or reference content that precedes variable request data.
- *When is batching unsuitable?* Interactive requests with strict time-to-first-token requirements or highly variable request sizes.
- *What can invalidate cache benefits?* Changing prefix content/order or provider cache eligibility/TTL rules.

## 5. When is parallel tool execution beneficial, and what can limit it?

**Answer.** Parallel execution reduces wall-clock time when independent calls can run simultaneously. Limits include provider/tool rate limits, downstream capacity, network overhead, straggler calls, data dependencies, and shared-resource contention. Bound concurrency and propagate deadlines/cancellation.

**Follow-ups**
- *How do you handle partial results?* Define whether the task can proceed with available results or must fail as a whole.
- *What is fan-out/fan-in?* Launch multiple independent tasks, then aggregate their results.
- *How do you prevent overload?* Semaphore/concurrency limits, queueing, backpressure, and circuit breakers.

## 6. How does streaming improve perceived latency? What does it not improve?

**Answer.** Streaming lets the client display the first generated tokens before the full response is complete, reducing perceived wait time and enabling progress feedback. It does not necessarily reduce total generation time or provider compute. It also introduces partial-output, reconnect, cancellation, and moderation/validation concerns.

**Follow-ups**
- *What is TTFT?* Time to first token, from request start to first visible generated token.
- *Why should structured JSON be handled carefully?* Partial chunks may not yet form valid JSON; parse only when complete or use structured event framing.
- *What if the stream disconnects?* Persist job/generation state and explicitly report interrupted versus completed.

## 7. How would you measure cost per successful task rather than cost per request?

**Answer.** Define success with a product-level criterion, then sum model, tool, retrieval, infrastructure, and retry costs across all attempts for the logical task. Divide total cost by the number of tasks meeting the success definition. Segment by task type, model, tenant, and version. Include abandoned or failed attempts in the numerator so expensive failures are not hidden.

**Follow-ups**
- *Why is cost/request misleading?* A cheap response that fails and triggers three retries can cost more per completed task.
- *How do you attribute shared infrastructure cost?* Use a documented allocation method and report assumptions.
- *What if success is subjective?* Use a rubric and human sampling to validate automated labels.

## 8. How would you reduce cost by 40% while checking that quality did not regress?

**Answer.** Establish a baseline on a fixed evaluation set: quality, task success, latency, and cost. Change one factor at a time—model routing, prompt compression, retrieval, caching, batching, or fewer agent turns. Re-run the same tests and compare category-level results. A 40% reduction is a target, not a claim; report the measured result and uncertainty.

**Follow-ups**
- *Why change one variable at a time?* It makes causal attribution easier.
- *What if cost drops but rare-case quality regresses?* Do not ship until high-risk cases meet their thresholds.
- *What is a Pareto frontier?* The set of solutions where improving one objective (e.g. cost) requires sacrificing another (e.g. quality).

## 9. How would you design an async FastAPI endpoint for a long-running agent?

**Answer.** Authenticate and validate the request, create a durable job record with a stable ID, enqueue the job, and return HTTP 202 with the job ID. A worker claims the job, runs the agent, persists progress/results, and emits status events. Provide GET status/result and SSE/WebSocket progress endpoints. Add tenant scoping, idempotent job creation, cancellation, deadlines, retries, and observability.

**Follow-ups**
- *Why return 202?* The request is accepted for asynchronous processing, not completed synchronously.
- *Where should job state live?* Durable storage, not only worker memory.
- *How do you prevent duplicate jobs?* Use an idempotency key scoped to the authenticated actor and request.

## 10. How should a frontend represent queued, running, interrupted, failed, and completed work?

**Answer.** Model the generation as an explicit state machine. Queued shows accepted/waiting; running streams progress; interrupted preserves partial output and says the result is incomplete; failed shows a safe error and retry option only when retry is safe; completed shows the final persisted result. Include job ID, timestamps, cancellation, reconnect, and accessible status announcements.

**Follow-ups**
- *Should partial output be shown after interruption?* Yes if useful, but clearly label it partial and never style it as a completed answer.
- *Can a retry duplicate side effects?* Yes; use backend idempotency and distinguish retrying a job from starting a new one.
- *How do you handle reconnect?* Fetch persisted status/events since the last event ID rather than relying only on the open stream.

## 11. How do you propagate request IDs and trace context across workers and tools?

**Answer.** Generate or accept a validated correlation ID at the API boundary, create a distributed trace context, and include it in the job record and queue message. Workers and downstream tool adapters extract the context and create child spans. Do not trust arbitrary IDs for authorization; they are for correlation only.

**Follow-ups**
- *What if a queue loses trace headers?* Persist trace context explicitly in message metadata and test propagation.
- *Should request IDs be user-visible?* They can be useful for support but should not expose internal secrets.
- *What is the difference between correlation ID and trace ID?* A correlation ID links business events; a trace ID identifies a distributed trace, though systems may align them.

## 12. How would you design rate limits and tenant-level budgets?

**Answer.** Apply limits at several levels: user, tenant, API key, model/provider, and tool. Track request rate, concurrent jobs, token budgets, and spend. Use token buckets or leaky buckets for rate control, reserve budget before expensive work, and reconcile actual usage afterward. Return clear retry timing or quota status and prevent one tenant from starving others.

**Follow-ups**
- *Why reserve budget first?* It avoids starting work that cannot be paid for under the tenant's remaining allowance.
- *How do you handle provider limits?* Global concurrency control, backoff, and fair scheduling.
- *What is noisy-neighbor isolation?* Preventing one tenant's load from degrading other tenants' service.
