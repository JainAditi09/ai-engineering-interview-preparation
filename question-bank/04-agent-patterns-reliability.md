# Agent patterns and reliability — detailed answers

## 1. Differentiate a workflow from an agent. When should you avoid an agent?

**Answer.** A workflow is a predefined sequence or graph of steps. An agent uses a model to choose actions dynamically based on the current task and observations. Workflows are deterministic, easier to test, and generally cheaper. Agents are useful when the path cannot be fully specified in advance, such as research across unknown sources. Avoid an agent when a simple API call, search, SQL query, or fixed state machine solves the problem.

**Example.** A refund flow should usually be a workflow: authenticate → fetch order → check policy → request approval if needed → issue refund. An agent may help interpret a natural-language support request, but backend rules should govern the refund.

**Follow-ups**
- *Can a workflow contain an LLM?* Yes. A workflow can use an LLM for bounded tasks while keeping transitions deterministic.
- *How do you justify agent complexity?* Compare task success, recovery, latency, and cost against a simpler baseline.
- *What is a hybrid architecture?* A deterministic workflow with bounded model decisions at selected nodes.

## 2. Explain plan-and-execute and its failure modes.

**Answer.** A planner decomposes a goal into steps; an executor performs them and reports results. The plan can improve visibility and help with long tasks, but it can become stale when observations change. The executor must validate each step, and the system should re-plan when a precondition fails or new information invalidates the plan.

**Follow-ups**
- *Should the whole plan be generated once?* Not necessarily. Replanning at checkpoints handles changing state.
- *How do you prevent unsafe planned actions?* Validate permissions and business rules at execution time, not just during planning.
- *What is the overhead?* Extra model calls, plan storage, and possible re-planning loops.

## 3. What is reflection/self-critique? When can it make results worse?

**Answer.** Reflection asks a model to inspect an earlier answer or action and identify weaknesses before producing a revision. It can catch omissions and improve structure, but it is not independent verification: the same model may repeat the same misconception, introduce new errors, or make an already correct answer worse. Use external checks where possible—tests, schemas, retrieval evidence, or a separate evaluator.

**Follow-ups**
- *When is reflection useful?* Draft review, code critique, checklist completion, and tasks with explicit quality criteria.
- *How do you know it helped?* Compare against a baseline using a fixed eval set and measure net quality, latency, and cost.
- *Can a critic be a different model?* Yes, but model diversity does not eliminate correlated errors.

## 4. When would you use a router or prompt chaining?

**Answer.** A router classifies an incoming task and directs it to a specialized prompt, model, tool, or workflow. Prompt chaining passes the output of one bounded LLM step to another—for example, extract facts, then draft a response. These patterns are useful when tasks have clear subproblems or different quality/cost requirements.

**Follow-ups**
- *How can a router fail?* Misclassification can send a task to an incapable or unsafe path.
- *How do you evaluate routing?* Measure per-class precision/recall, downstream task success, fallback rate, and cost.
- *Why not one giant prompt?* Separate stages can be easier to validate and debug, but they add calls and error propagation.

## 5. What is orchestrator-worker, and what coordination overhead does it introduce?

**Answer.** An orchestrator decomposes a task and delegates sub-tasks to workers, then combines their results. It is useful when work can be parallelized or specialized. Costs include extra model calls, duplicated context, worker disagreement, result aggregation, and more complex tracing. Parallel workers may also hit provider rate limits.

**Follow-ups**
- *How do you prevent duplicated work?* Give workers distinct scopes and structured outputs.
- *How do you handle conflicting results?* Use evidence-based aggregation or a defined tie-breaker; do not blindly majority-vote.
- *When is it not worth it?* Small tasks or tasks with strong dependencies and little parallelism.

## 6. How do you choose between sequential and parallel execution?

**Answer.** Parallelize independent operations when they do not depend on one another and can safely run concurrently. Execute sequentially when a later step needs an earlier result or when ordering protects side effects. Bound concurrency and respect rate limits, deadlines, and cancellation.

**Follow-ups**
- *What if two parallel workers write the same resource?* Use transactional updates, optimistic version checks, locks, or redesign the work as independent writes.
- *Does parallelism always reduce latency?* No; queueing, rate limits, aggregation, and stragglers can dominate.
- *What is a straggler?* A slow worker that delays completion of the overall fan-out task.

## 7. What is context bloat, and how do you control it?

**Answer.** Context bloat is the accumulation of irrelevant, redundant, or stale information in a prompt. It increases token cost and latency and can distract the model from important evidence. Control it with token budgets, selective retrieval, compact summaries, deduplication, and explicit prioritization of current instructions and constraints.

**Follow-ups**
- *What should never be removed casually?* Safety instructions, current task constraints, confirmed decisions, and information needed to execute safely.
- *How do you evaluate summarization?* Test downstream task success and preservation of key facts, not summary fluency alone.
- *Can retrieval replace all conversation history?* No. Some interaction context and unresolved references may need to be retained.

## 8. How can incorrect tool output compound across multiple agent steps?

**Answer.** An agent treats tool results as observations. If a tool returns stale, incomplete, or incorrect data, the model may use it as a premise for subsequent actions. Each step can amplify the original error. Validate tool responses, include freshness/provenance, use authoritative sources for critical decisions, and re-check preconditions before side effects.

**Follow-ups**
- *Should the model decide whether a tool result is trustworthy?* It can flag uncertainty, but deterministic validation and source policy should be enforced by the application.
- *How do you detect stale data?* Include timestamps/version identifiers and enforce freshness limits.
- *What if two sources disagree?* Surface the conflict or use a defined source-of-truth policy.

## 9. How would you build failure-injection tests for an agent?

**Answer.** Wrap tools behind interfaces so tests can simulate timeout, exception, rate limit, malformed output, empty results, stale data, and partial success. Assert that the agent stops or recovers within its budgets, avoids unsafe side effects, records the right terminal state, and gives an honest user-facing result.

**Follow-ups**
- *What is chaos testing?* Deliberately introducing failures to test system resilience under realistic conditions.
- *How do you make tests deterministic?* Use seeded fixtures, fake clocks, mocked provider responses, and controlled tool adapters.
- *What should be measured?* Recovery rate, duplicate side effects, latency, retries, and correctness after recovery.

## 10. What safeguards prevent an agent from retrying forever?

**Answer.** Enforce maximum steps, wall-clock deadline, token/cost budget, per-tool retry limit, backoff with jitter, and repeated-state detection. A retry policy should classify errors: transient failures may be retried; invalid arguments, authorization denials, and deterministic business failures generally should not be. When exhausted, persist partial progress and return a clear incomplete status.

**Follow-ups**
- *Why use jitter?* It spreads retries and reduces synchronized retry bursts.
- *What is a circuit breaker?* A mechanism that temporarily stops calls to a failing dependency and later tests recovery.
- *What should the user see?* The work completed, the point of failure, and whether retry/resume is safe.

## 11. How do you make an agent degrade gracefully when a tool is unavailable?

**Answer.** Detect dependency health and return a typed unavailable result. The agent can use an approved alternative source, answer only the parts supported by available evidence, or ask the user to retry later. It must not pretend the tool succeeded or silently substitute a less-authoritative source for a critical operation.

**Follow-ups**
- *When should a fallback be forbidden?* When it changes security, data authority, or side-effect semantics.
- *How do you prevent cascading failures?* Timeouts, circuit breakers, bounded queues, bulkheads, and fallback budgets.
- *How do you communicate partial completion?* Persist status and clearly distinguish completed, skipped, and failed steps.
